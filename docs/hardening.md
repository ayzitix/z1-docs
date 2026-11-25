# Hardening and failure contracts

What several review passes changed, per surface: the prior symptom, the contract the code now
enforces, the constants and strings a user can see, the test that pins it, and what remains.
Every contract below was re-read against the source; where a review lead and the code disagreed,
the code wins and the difference is noted inline. The prior symptoms are as the reviews report
them — the code no longer contains them — while the contract and the test are what this file
verifies. Mechanisms are in [engineering.md](engineering.md).

## Approval handshake over the server

**Was.** An unusable `approval/request` reply — `null`, a bare string, an unknown decision —
threw inside the peer, rejected the run's promise and could take the server down as an unhandled
rejection. A late `approval.respond` returned `{}` for an approval nobody was waiting for.

**Now.** `approvalReplyDecision` (`z1/packages/server/src/server.ts`) requires an object with
`decision` in `['allowed-once', 'rejected']`. An unusable reply is answered with a fresh
`approval/request` frame for the same `approvalId`, `nodeId` and `path` under a new JSON-RPC id
— the repeat is the refusal, because a result has no reply channel. After
`APPROVAL_REPLY_ATTEMPTS = 3` the server stops asking, sends `approval.abandoned` with
`reason: 'replies-exhausted'`, and leaves the run to its deadline. `approval.respond` is bound
to the connection that was asked.

- `approvalTimeoutMs`, default `DEFAULT_APPROVAL_TIMEOUT_MS = 300_000`. On expiry
  the executor reports `NODE_FAILED: approval request for node "<id>" failed:`
  plus `Error: approval <id> for node "<id>" timed out after 300000ms`.
- Unknown, foreign or already-decided id: `1003`, `unknown approval: <id>`.
- Owner, after `run.cancel` or a settle with no decision: `1004`,
  `run <id> is not waiting for approval`.

The brief said "after settle 1003": true only when the settle came from a decision. A run that
settles with an approval still pending is abandoned with `reason: 'run-settled'`
(`abandonRunApprovals` in the `runAndSettle` finally block, `server.ts:533`) and its owner gets
`1004`. That path is reachable but **unpinned**: it needs the run to end while an entry it created
is still pending — a node that requested an approval without awaiting it, or an executor-side wait
(the node's `timeoutMs` through `executeAttempt`, or `AutonomyPolicy.timeoutMs` inside
`requestApproval`) that stopped waiting before the server's deadline, leaving the server's promise
unsettled. No test asserts `run-settled`; the reasons the specs do pin are `replies-exhausted` and
`run-cancelled`. No error frame is ever sent for a bad reply, because the two directions have
separate id namespaces.

**Pinned by** `server.spec.ts`: "an unusable approval reply is asked again and the client can
answer", "three unusable approval replies hand the run to the deadline", "an unusable
approval/request reply is answered by a repeat, not by an error frame", "approval.respond is
refused once the run is cancelled", "a decided approval cannot be answered twice".

**Residual.** The canvas queues one dialog per frame (`z1/apps/canvas/src/store.ts` appends to
`approvals`), so a repeat shows the dialog again for the same `approvalId`; a client that only
calls `approval.respond` never sees the refusal.

## `run.cancel` across the prune

**Was.** Cancel succeeded while the settled run had state and answered "unknown run" once it was
pruned: one request, two answers.

**Now.** With no `runs` entry, `run.cancel` reads the journal: unknown is `1001`; settled is
`1004` with `run <id> is already <status>`. While live, cancel aborts the controller, abandons
the run's approvals (`run-cancelled`) and answers `{}`.

**Pinned by** `server.spec.ts`: "a settled run keeps one run.cancel answer across the prune" (20
attempts across the boundary, identical refusal), "run.cancel reports an unknown run as not
found".

**Residual.** Not idempotent by design: a second cancel of a cancelled run is a `1004`, not a
silent success.

## JSON-RPC frame validation

**Was.** A frame that was not an object, or whose `method` was not a string, was routed as a
response and answered with silence.

**Now.** `JsonRpcPeer.#onText` (`z1/packages/protocol/src/transport.ts`) answers every frame it
can classify; the reply id is `null` whenever the incoming id cannot be echoed.

| Input | Reply |
| --- | --- |
| unparsable JSON | `-32700`, `parse error: invalid JSON` |
| not an object, or an array | `-32600`, `invalid request: not an object` |
| `jsonrpc` not `"2.0"` | `-32600`, `invalid request: jsonrpc must be "2.0"` |
| `method` present, not a string | `-32600`, `invalid request: method must be a string` |
| `id` neither string nor number | `-32600`, `invalid request: id must be a string or a number` |
| unknown method | `-32601`, `method not found: <name>` |
| handler throws | its code if a `JsonRpcFault`, else `-32603` |
| notification | no reply; an unhandled name is ignored |

A `null` id is treated as a notification. A response is matched by id; an unknown id is dropped;
a response without `result` rejects with `-32603`, `response has no result`; `result: undefined`
is sent as `null`. No batch support: an array is `-32600`, not a batch response.

**Pinned by** `transport.spec.ts`: "replies -32700 to invalid JSON", "answers every invalid
request with -32600 instead of silence", "answers a frame whose method is not a string instead
of treating it as a response", "echoes a usable id when only the version is wrong".

## Provider stream (SSE)

**Was.** The reader split on `\n` only, dropped an event cut at EOF, skipped a `data:` line that
was not JSON, and returned a partial answer when the stream stopped early.

**Now** (`z1/packages/provider-openai/src/sse.ts`, `openai.ts`):

- A line ends on CRLF, LF or a bare CR; a trailing CR is held back as a possible half of a CRLF
  pair.
- A tail without a terminator is accepted only when it is a whole frame, else
  `openai-compatible stream ended mid-event`.
- `[DONE]` is compared after `trimEnd()`, so `data: [DONE] \t` terminates.
- No `[DONE]` means complete only if a `finish_reason` arrived; neither is
  `openai-compatible stream ended before completion`.
- A payload that is not JSON is `openai-compatible stream sent an event that is not JSON`; an
  upstream error object becomes `openai-compatible stream error: <redacted detail>`.
- `tool_calls` merge by `index`; a missing id gets `call_<seq>_<index>`. `usage: null` is "not
  yet", which `stream_options.include_usage` sends on every chunk but the last.

**Pinned by** `provider.spec.ts`: "accepts a stream framed with bare CR line endings", "accepts
a stream that ends without [DONE] once finish_reason arrived", "rejects a stream that ends
before completion", "accepts a [DONE] terminator padded with trailing whitespace", "rejects an
event whose JSON is truncated instead of dropping it", "rejects a stream that stops in the
middle of an event".

**Residual.** `event:`, `id:` and comment lines are ignored, and each `data:` line must be a
whole JSON object: a payload split across two `data:` lines (legal SSE, joined with `\n`) is not
supported.

## Module capabilities

**Was.** `**` and `${moduleDir}/**` denied every read, because a pattern with no literal
directory produced no anchor; one non-matching pattern could cancel the patterns after it;
`127.1`, `2130706433`, `0x7f.0.0.1` and `0177.0.0.1` classified as public while `fetch` reached
loopback; on Windows a grant spelled `Sub/**` denied the directory `sub`.

**Now** (`z1/packages/module/src/capability.ts`):

- `anchor()` maps a pattern with no literal directory onto the real module root, so `**` and
  `${moduleDir}/**` grant the same tree and stop at its edge.
- A matching pattern that cannot be anchored records
  `capability pattern has no base directory: <pattern>` and continues; denial happens only after
  the loop, with `fs.<dir> path is outside the granted patterns: <path>` or
  `fs.<dir> path resolves through a symlink outside the granted pattern <pattern>: <real>`.
  Fail-closed, without one pattern vetoing the rest.
- `parseIPv4Legacy` accepts one to four parts, each decimal, octal (`0…`) or hex (`0x…`), the last
  filling the remaining bytes; the four spellings above are loopback, so a wildcard cannot reach
  them.
- `foldedPath` lower-cases on `win32` for comparison only: in `checkFs`, in `checkFsReal` (the
  lexical repeat and the base `startsWith` containment test), and nowhere a path is reported.
- `..` in a path or a pattern is refused (`path contains a parent directory segment`), never
  normalised.
- A `net.hosts` entry may carry a port; a port-scoped grant matches only that port, a portless one
  any port.

**Pinned by** `capability.spec.ts`: "`**` grants the module tree and stops at its edge", "`**`
and `${moduleDir}/**` mean the same tree in either order", "pattern order decides which pattern
is tried first, not the verdict", "a grant reaches a directory spelled with another case on
Windows only", "the legacy inet_aton spellings of loopback are private".

**Residual.** Containment is a realpath of the longest existing ancestor plus a string boundary;
the check and the syscall are separate, so a symlink swapped in between is not prevented.

## Module audit chain

**Was.** The lifecycle log was JSONL with no integrity check, so an edited, reordered or spliced
entry read back as fact.

**Now** (`z1/packages/module/src/audit.ts`): each entry carries `seq`, `prev` and
`hash = sha256(canonicalJson(entry without hash))`, from `GENESIS_HASH = '0'.repeat(64)`.
`read()` verifies shape, `seq` continuity, the `prev` link and each hash, and throws
`AuditChainError`: `audit chain broken at seq <n>: <reason>` — `empty line`,
`line is not valid JSON`, `entry shape is invalid`, `expected seq <n>, found <m>`,
`prev hash does not match the previous entry`, `hash does not match the entry content`. `append`
rewrites through a temporary file, so a crash leaves the old log or the new one.

**Pinned by** `audit.spec.ts`: "starts at the genesis hash and links each entry", "writes
atomically and leaves no temporary files", "throws on the first tampered line", "throws on
malformed JSON and a broken prev link".

**Residual — a stated limitation.** Tail truncation is **not** detectable: dropping the last
entries leaves a shorter chain that verifies from genesis, because the file carries no anchor
outside itself. A separate head file has the same problem. The file's length is part of the
trust boundary; "does not detect tail truncation without an external anchor" keeps that
deliberate.

## Prompt compaction

**Was.** Checkpoint detection matched a prefix, so any message quoting the preamble — including
a tool result spilling this package's own source — counted as a checkpoint and filled
`maxCompactions`. A model that reworded the mandated heading defeated the node the same way.

**Now** (`z1/packages/nodes/src/context.ts`): a checkpoint is a `user` message whose content
starts with the preamble plus `<compacted-summary>`, ends with `</compacted-summary>`, and
contains `<!-- z1 checkpoint -->` or the legacy heading `## Primary Request and Intent`. A
marker inside a tool result is not a checkpoint; a wrapper without a closing tag is not one.
Real checkpoints count toward `maxCompactions`, so a stuck transcript stops paying; a checkpoint
written before the tag existed still counts, which keeps an old journal's cap after an upgrade.
Thresholds: `keepRecentTurns = 6`, `maxSummaryTokens = 2000`, `minimumReduction = 0.1`,
`maxCompactions = 3`, `minDroppedTokens = 256`.

**Pinned by** `context.spec.ts`: "the checkpoint count is capped, so a stuck transcript stops
paying", "a real checkpoint counts toward maxCompactions", "a user turn that merely repeats the
preamble is not a checkpoint", "recognises its own checkpoint even when the model renames the
heading", "a checkpoint written before the tag still counts", "marker text quoted in a tool
result is not a checkpoint".

**Residual.** A fully forged wrapper that reproduces the preamble, the tags and the writer tag
passes: the tag is not a signature. "a whole forged wrapper counts: the tag is not a signature"
pins it; the cost is one summary call, not a lost transcript.

## `z1.llm.chat` message validation

**Was.** A non-string `content` was silently dropped from the request, orphaning the tool call
it belonged to, and the run completed on a transcript the provider never saw.

**Now.** `z1/packages/nodes/src/llm.ts` validates each message first and throws
`z1.llm.chat: message content must be a string (role '<role>', index <n>)`; the node fails
instead of completing.

**Pinned by** `llm.spec.ts`: "fails clearly when message content is not a string", "rejects
multimodal content parts, which the messages schema does not carry".

**Residual.** Only `string` content is supported; an image or text-part array is refused, not
converted.

## `ctx.clip` and the context nodes

**Was.** Clipping could return more than `maxChars`, because the marker length depends on the
numbers it prints; with a small cap nothing but the marker fitted.

**Now** (`z1.ctx.clip` in `z1/packages/nodes/src/context.ts`): the reserve is
`min(140, max(16, floor(maxChars / 3)))`, the kept text is trimmed twice and cut on code-point
boundaries, and a cap too small for the numbered notice falls back to `… [z1: clipped] …` and
then to its prefix. The value is at most `maxChars` at every cap. An object result is serialized
once; a spill path is workspace-relative and rejects an absolute path or `..`.

**Pinned by** `context.spec.ts`: "the value never exceeds maxChars, however small the cap", "a
tiny cap is honoured down to a single character", "a surrogate pair is never split by the cut",
"a barely-too-long value is not inflated by the marker", "a spill directory cannot leave the
workspace".

**Residual.** The marker is model-visible prose; a later re-encoding of whitespace can change
the measured length after the node returns.

## Agent graph: roles and exports

**Was.** An unknown `--role` failed inside a section lookup, naming neither the role nor the
valid set; test helpers reached into module internals for `z1.tool.background` and the last-text
node.

**Now.** `templateRole` (`z1/packages/agent/src/index.ts`) rejects a name before the graph is
built: `unknown role "<name>"; expected planner, coder, reviewer, summarizer, judge, router`.
`@z1/nodes` exports `toolBackground` (`z1/packages/nodes/src/tools.ts`) and `coreLastText`
(`core.ts`), and `z1/SPEC.md` §7 documents the `literal` flag, `maxCompactions` and
`minDroppedTokens`.

**Pinned by** `agent.spec.ts`: "rejects an unknown role by name instead of failing on a section
lookup".

**Residual.** `RoleId` is a closed union: a new preset is a code change in
`z1/packages/prompt/src/defaults.ts` plus the builder.

## Runtime approval timeout

**Was.** `onTimeout: 'approve'` let the node run but journaled no decision, so a replay read the
approval as still pending.

**Now.** `runNode` (`z1/packages/runtime/src/executor.ts`) emits `approval.decided` with
`allowed-once` **before** the node runs and completes.

**Pinned by** `executor.spec.ts`: "approval timeout can be configured to approve" asserts one
request, one decision, and that the decision is ordered between the request and
`node.completed`.

**Residual.** A timeout that denies journals no decision, so a resumed run asks again: the node
is neither silently allowed nor silently blocked.

## Journal open failure

**Was.** A path that opened but was not a database failed on the first statement, with a bare
SQLite message and the handle still held; the message named no path.

**Now** (`z1/packages/journal/src/sqlite.ts`): both the open and the pragma/schema step are
wrapped. The error is `journal: cannot open database at <path>: <cause>`, and the handle is
closed before it is thrown, so a rejected file holds no lock. Later use reports
`journal: database at <path> is closed`. `createRuntime` lets the error out; the CLI's top-level
catch prints it and exits 1.

**Pinned by** `journal.spec.ts`: "unusable database paths fail with a clear message", "a file
that is not a database is refused with its path", "close is idempotent and further use is
reported".

**Residual.** Corruption found after a successful open still surfaces as the raw SQLite error,
without the wrapper.

## Session store: atomic writes and the trash

**Was.** On Windows a rename failed with `EPERM` / `EBUSY` while another process held the
destination; `remove()` could fail while another process listed sessions; the trash grew without
bound; a moved entry could be pruned immediately after a clock jump.

**Now** (`z1/packages/session/src/fsutil.ts`, `store.ts`, `paths.ts`):

- `renameWithRetry`: `WRITE_ATTEMPTS = 3`, `WRITE_RETRY_MS = 20`, retrying only `EPERM`, `EBUSY`,
  `ENOTEMPTY`. Failures name the file and the cause:
  `session: could not acquire <file> for an atomic write after <n> attempt(s): <cause>`,
  `session: could not move <id> to the trash after <n> attempt(s): <cause>`. Nothing has moved
  when the second fires, so the session stays complete under `sessions/`.
- Atomic write: temp file in the same directory, `fsync`, rename, directory `fsync`; the temp file
  is unlinked on failure.
- `remove()`: trash move, stale `.lock` removal, latest pointer, `utimesSync(target, now, now)`,
  then `pruneTrash(root, { protect: [target] })`.
- Retention: `TRASH_KEEP = 50` entries and `TRASH_KEEP_MS = 30 * 24 * 60 * 60 * 1000`, whichever
  first. A protected entry keeps its slot and displaces the oldest unprotected one; a negative age
  is fresh, never old.

**Pinned by** `atomic-write.spec.ts`: "a transient EPERM is retried until the replace succeeds",
"a replace that keeps failing is reported as could-not-acquire", "a transient EPERM on the trash
move is retried", "a trash move that keeps failing leaves the session in sessions/".

**Residual.** Success under natural Windows contention is proven by injection, not by an
observed race; three attempts 20 ms apart is shorter than a slow antivirus scan.

## z1-tools: glob, grep and `.gitignore`

**Was.** A pattern of openers was quadratic to price and compile; a 30 KB `.gitignore` line cost
seconds per walk; an escaped `]` was read as a closing bracket, so the tail behind it was never
priced and a 190-character name backtracked for 27.8 s.

**Now** (`z1-tools/lib/search.js`):

- `MAX_GLOB_PATTERN_LENGTH = 1024`, checked before pricing or compiling:
  `glob.pattern is too long (N chars, limit 1024)`, `grep.glob is too long (N chars, limit 1024)`.
- `MAX_GLOB_COST = 8_000_000`; price `alternatives × C(GLOB_PRICE_TEXT + stars, stars)` with
  `GLOB_PRICE_TEXT = 200`, computed in one linear pass that stops at the cap.
- `skipClass` returns `-1` for an unterminated class **and** for one whose only `]` is escaped; an
  unterminated `[` is a literal, so the scan continues after it instead of hiding the tail.
- An over-limit or over-cost `.gitignore` line is skipped and counted:
  `… (ignored 1 unsafe .gitignore rule)` / `… (ignored 2 unsafe .gitignore rules)`.
  The rest of the file still applies.
- Refusals: `glob.pattern is too expensive; simplify the alternations or wildcards`,
  `grep.glob is too expensive; simplify the alternations or wildcards`,
  `glob.pattern is not a valid glob: unterminated character class at character N`,
  `glob.pattern is not a valid glob: "/" is not a file pattern`.

**Pinned by** `glob-cost.test.js`: "the length gate bounds the price scan and the translation",
"the gate refuses the shapes that hang a backtracking engine", "the gate allows the patterns a
source search writes", "an unterminated class does not hide the braces behind it", "glob refuses
a long pattern by length, before it prices or walks", "an expensive .gitignore line is skipped,
counted, and the rest still applies", "an escaped `]` never closes a class, so the tail behind
it is priced".

**Residual.** The price is calibrated for a ~200-character name, and the test measures only that
shape. `*a*a*b` prices at 1 373 701 ways and is permitted; on a 1000-character name it took 313
ms here (measured, not asserted) against 12 ms on a 200-character one. A permitted shape on a
long path is slow, not hung.

## z1-tools: symlinks, counters, tree, move

**Was.** A symlink to a directory was not counted as a directory for `.gitignore`; glob and grep
reported different walk counters; a `tree` link pointing outside the workspace looked like one
inside; `move` into a non-empty directory merged or failed with a bare `EPERM`; on Windows
`move("d", "D/x")` reached the helper instead of the guard.

**Now.**

- `standsForDirectory` counts a link to a directory as a directory for ignore rules; the walk
  still never descends through a link.
- One `walkCounts` shape produces `N files walked[; M ignored paths skipped]` for both searches,
  so identical trees give identical numbers.
- `treeLabel` appends ` (outside workspace)` when the target resolves outside; an unknown or empty
  target is not labelled, because the label would be a guess.
- A non-empty destination refuses:
  `<label>: destination directory is not empty; refusing to merge into it`.
- The own-subtree guard folds case on `win32`:
  `move: cannot move "<from>" into its own subtree ("<to>")`.

**Pinned by** `reporting.test.js`: "glob and grep report the same walked and ignored counters",
"a link to a directory is counted as a directory, and its outside target is named", "tree reads
a junction target with its kernel prefix, inside or outside"; `move.test.js`: "move into a
non-empty directory refuses to merge and keeps both trees", "move refuses its own subtree
whatever the case (win32)".

**Residual.** `pointsOutside` strips a `\\?\`/`\??\` kernel prefix and resolves lexically; an
unreadable target is not labelled, and a junction with an empty target prints as a bare symlink.

## Subagent hand-off and time budget

**Was.** `promptTokensOf` looked for `promptTokens`/`totalTokens`, while the shipped `ChatUsage`
is `{ inputTokens, outputTokens }`, so the 75% hand-off never fired in production and the fixed
24-step cap was the real limit. When it did fire, `controller.abort()` made the graph resolve
`cancelled`, and `settleChild` returned in the cancelled branch before the hand-off branch — no
report. `resultText` also relabelled any `partial` with steps as a step cap. The report transcript
scraped every completed node, so it contained both `loop/N/chat` and `loop/N/assistantMsg` (one
duplicate per turn) and put the system message after the user message.

**Now.** `promptTokensOf` accepts `inputTokens`/`input_tokens` first and the loop budget node's
`tokens` estimate is a floor: `max(usage, estimate)`. The working limits are
`SUBAGENT_HANDOFF_RATIO = 0.75` and `DEFAULT_SUBAGENT_TIME_BUDGET_MS = 2_400_000` (40 minutes);
either stops the child, makes one tool-free `llm.chat` capped at
`SUBAGENT_REPORT_TIMEOUT_MS = 60_000`, and returns `partial` with `context_limit` or `time_limit`
and a report asking **Done / Found / Not done / Continue / Problems**. A hard `timeoutMs` defaults
to the budget plus `DEFAULT_SUBAGENT_REPORT_GRACE_MS = 120_000`; an explicit shorter value wins
and is `child_timeout` with no report. `DEFAULT_SUBAGENT_MAX_STEPS = 10_000` is a runaway guard;
`MAX_SUBAGENT_MAX_STEPS` rejects a higher explicit cap as `invalid_input`. `timeMinutes` (1–120,
model-facing) can only lower the host budget. The report transcript is the loop's
`messagesMerge` value, so it is `[system, user, …]` with one assistant per turn and resolvable
`toolCallId`s. A host cancel during the report wins and returns `cancelled`; a graph that
completed keeps its real answer even if a timer fired during the `run.completed` append, and the
row and last event stay `completed`.

**Pinned by** `handoff.spec.ts` ("inputTokens crossing 7 500 makes one report: partial/context_limit",
"a host abort during the report turn is a cancellation, not a report", "the report transcript is a
conversation a provider would accept"), `time-budget.spec.ts` ("ends the run with one report:
partial/time_limit", "a timer firing during the run.completed append keeps the real answer",
"the first budget to fire owns the report, in either order") and `background.spec.ts`
("a cancel() during the hand-off report wins over the report").

**Residual.** The budget estimate cannot see tool schemas, so a tool-heavy child with no usage
can cross 75% later than the true prompt size. An explicit `timeoutMs` shorter than the budget
silently disables the report (documented and tested). The report cap abandons rather than
cancels the provider call, because `LlmService.chat` takes no signal. The report transcript ends
one turn behind the stop: the aborted turn is not synthesised, because its `toolCalls` have no
tool results; its text reaches the parent as the kept `partial` text.

## Residual limits, in one place

| Limit | Why it stays | Pinned by |
| --- | --- | --- |
| audit misses tail truncation | no anchor outside the store | `audit.spec.ts` |
| a forged checkpoint wrapper passes | tag is not a signature | `context.spec.ts` |
| permitted globs slow on long names | price calibrated at ~200 chars | `glob-cost.test.js` |
| Windows rename retry on real contention | proven by injection | `atomic-write.spec.ts` |
| POSIX session/capability behaviour | code review only; no POSIX machine here | not covered |
| module-node approvals, concurrent runs | resolves to the latest live run | [sdk.md](sdk.md) |
| network is not a syscall boundary | only brokered `ctx.net` checks hosts | `SECURITY.md` |

## User-visible changes

- A repeated `approval/request` shows the dialog again in the canvas, for the same `approvalId`;
  the first usable answer wins.
- A glob pattern longer than 1024 characters is rejected with the length in the message instead of
  being walked.
- An `.gitignore` line too expensive to compile is skipped and counted, so an answer can end with
  `… (ignored 1 unsafe .gitignore rule)`.
- Non-string message content now fails the node by role and index instead of producing a silently
  different request.
