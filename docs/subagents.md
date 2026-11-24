# Subagents

A subagent is a child agent the model can hand one self-contained task to. The
child does not see the parent's conversation: it gets a prompt, a filtered copy
of the parent's tools, and nothing else. It reports back text.

That isolation is the point. A child cannot be confused by the parent's history,
cannot spend the parent's context, and cannot reach a tool the parent did not
give it.

Code: `z1-sdk/z1/packages/subagent/` (the contract), `z1-cli/{agents,agents-view,subagent-host}.ts`
(the session and the terminal).

## The contract

```ts
subagent({
  description: string,                    // short label, ≤ 120 chars
  prompt: string,                         // the task, ≤ 64 KiB
  tools: { enable: string[], disable?: string[] },
  wait?: boolean                          // default false: background
})
```

**`tools.enable` is required.** There is no implicit "all tools" default,
because a child that silently inherits everything the parent could do is not a
narrower grant. `enable: []` gives the child no tools; the single entry `["*"]`
gives it every tool the parent has. `disable` subtracts from that set. Deny
always wins, whatever the order. An unknown tool name fails **before** the child
starts, so a typo cannot quietly widen or narrow what the child can do.

Limits, from `z1/packages/subagent/src/types.ts`:

| Constant | Value | Meaning |
| --- | --- | --- |
| `MAX_SUBAGENT_DESCRIPTION_LENGTH` | 120 | label length |
| `MAX_SUBAGENT_PROMPT_BYTES` | 64 KiB | task length |
| `MAX_SUBAGENT_TOOL_ENTRIES` | 64 | entries in `enable`/`disable` |
| `MAX_SUBAGENT_TOOL_NAME_LENGTH` | 128 | one tool name |
| `DEFAULT_SUBAGENT_MAX_CHILDREN` | 4 | concurrent children |
| `DEFAULT_SUBAGENT_MAX_TOTAL` | 8 | children started per session |
| `DEFAULT_SUBAGENT_MAX_STEPS` | 24 | steps one child may take |
| `DEFAULT_SUBAGENT_MAX_DEPTH` | 1 | **no nested subagents** |

**No nesting.** v1 and v2 reject the delegation tool inside a child's grant: a
child cannot start a grandchild. The rejection happens at grant resolution, not
at run time, so it cannot be bypassed by a cleverly worded prompt.

## Background is the default

`wait` decides whether the parent blocks.

- **Omitted or `false` — background.** The call returns
  `{ runId, status: 'running' }` in the same model step. The child keeps working
  after the call returns and after the turn ends.
- **`wait: true` — blocking.** The call returns the child's result text, exactly
  as a subagent call behaved before background existed: same result shape, same
  finish timing, same budget accounting, same cancellation.

The model is told which to use when: `wait` when the answer is needed to
continue, background for independent work.

### What "does not wait" means, precisely

There are four separate not-waits, and it is worth keeping them apart because
only the first is about the tool call:

1. **The call does not wait.** It resolves immediately with a run id.
2. **The turn does not wait.** The parent turn keeps going and closes normally.
   A tool result carrying `status: 'running'` is an ordinary successful tool
   result to the loop; nothing in the turn branches on it.
3. **The result is not a tool result.** The child's answer is not returned to the
   model inside the launching call — it did not exist yet. It is delivered at
   the **start of the parent's next turn**, once, as a marked block.
4. **Nothing blocks silently.** The budget slot is held until the child *ends*,
   not until the call returns; `/exit` has a bounded wait; `--once` has an
   explicit policy instead of a race.

### Lifetime and ownership

A background child owns its **own** `AbortController`. The turn's signal is not
wired to it, so pressing Ctrl-C to stop the turn does **not** stop delegated
work. That is deliberate: the user interrupted the conversation, not the job
they started.

The **session registry** owns every child — not the turn. Consequence, stated
plainly because it is the rule the code follows: `handle.dispose()` at the end
of a turn never touches a child, and `/new` keeps running children (they are
session work), clearing only the finished ones.

A child ends when one of these happens:

| Cause | Recorded as |
| --- | --- |
| The child finishes its own work | `completed` |
| The child fails | `failed` |
| `timeoutMs` expires | `failed`, `errorCode: 'child_timeout'` |
| `maxSteps` is reached | `failed`, result `partial` |
| The child is cancelled | `cancelled` — **not** `failed` |

Cancellation and failure are kept apart on purpose: a child the user stopped did
not go wrong.

### The budget

`maxChildren` counts **running** children; `maxTotal` counts **started** ones.
So a user who ran four children one after another is not blocked from a fifth,
but a session cannot start more than eight in total.

A slot is reserved before the child begins and released when it ends — including
when starting the child throws. If the reserve is refused, no run is created and
no row is written.

## What the user sees

The terminal layer lives in `z1-cli`.

**The launching row** reads `started · /agents` rather than a duration, because
there is no duration yet.

**The footer hint** shows how many children are running whenever there are any:

```
◷ 2 subagents running · /agents
```

It takes the slot the shortcut reminder (`? for shortcuts`) otherwise uses, and
it stays visible between turns — the point is to be readable whenever the user
looks at the prompt. An armed confirmation (`press ctrl+c again to exit`) and an
open picker still win over it, because those describe what the next key does.

**`/agents`** opens an overlay picker, the same way `/tools` does — drawn over
the transcript, owning the keyboard, dismissed with Escape. It opens **while a
turn is running**, which is the reason it is an overlay and not printed output:
a printed list would queue behind the turn.

| Key | Action |
| --- | --- |
| `↑` `↓`, digits | move the selection |
| `Enter` | open that child's transcript and close the picker |
| `x` / `Delete` | stop the selected child and **keep the picker open** |
| `Esc` | close; the draft is left byte-identical |

Stopping a `wait` child is refused, because that child belongs to a blocking
call; the picker says so and names the way to stop it (Ctrl-C in that turn)
rather than silently doing nothing.

**Delivery.** A settled child is drained at the start of the next turn and
injected as one block, marked as subagent output, carrying the run id, the
status and the child's text. It is neutralised with the same formatter as a
foreground result, so a child that prints something resembling instructions
cannot impersonate them. Exactly once: never mid-turn, never twice, never as the
launching call's tool result.

**Non-interactive runs** (`--once`, a pipe, `Z1_TRAY=off`) print the list
instead of opening a picker.

## `--once` and exit

`--once` waits for background children by default, prints each final status on
stderr, and exits. The exit code still reflects the **main** run, not the
children.

`Z1_NO_WAIT=1` detaches instead: the run ids are printed and the process exits
without waiting. Those children keep running until their own `timeoutMs`, and
their rows stay `running` until the next start reaps them.

`/exit` cancels the running children, waits up to five seconds, and leaves. The
wait is bounded and the timer does not hold the process open, so a child that
ignores cancellation cannot wedge the exit.

## How cancelling actually works

This is worth knowing because it was wrong for a long time.

A child's tools run as a child process of the module worker, outside the worker
itself. Cancelling a module execution used to send the cancel frame only: a tool
blocked inside `ctx.process.run` could not acknowledge it, so two seconds later
the host SIGKILLed the **entire module worker** — taking every concurrent
execution with it — and the spawned process survived all of it.

The fix (`z1-sdk c9a8614e`) is host-side and module-agnostic:

1. every `process.run` child is tagged with the execution or service call that
   spawned it;
2. on abort, the host kills **that owner's** process group **before** sending
   `cancel` — SIGTERM, then SIGKILL after the grace;
3. the tool unblocks, unwinds, acknowledges, and the worker kill timer is
   disarmed.

A cancelled call therefore really stops its command, and concurrent executions
are untouched. The scope is the owner, not the worker.

**What remains:** a module that swallows the rejected call and keeps waiting
still hits the worker SIGKILL. That needs `com.z1.tools` to forward `ctx.signal`,
which is a module version bump (`0.1.10`) and not yet done. A child spawned from
a service *factory* lands on a shared owner and has no cancel path; it lives
until its own `timeoutMs`.

## Known limits

- **A launch as the model's last step closes the turn `failed`** at the step
  cap, with no final message. The child still runs and its answer is delivered
  next turn, so no work is lost — but the user sees an error line. This is agent
  loop behaviour, not subagent behaviour.
- **No nesting**, by design, not by omission.
- **The transcript of a `wait` child** is only reachable while its run is in
  memory; `/agents <ref>` cannot open a child from a previous process.
- **Killing the host process** kills its children with it, since they are the
  same process; the shell commands they spawned are reaped by the owner-kill
  above only when cancellation actually runs.

## Tests

| Suite | Location | Covers |
| --- | --- | --- |
| `packages/subagent` (141) | `z1-sdk/z1/packages/subagent/tests/` | grant, budget, background, cancel, result form |
| `subagent.test.mts` | `z1-cli/tests/` | the composed service, `wait` parity with v1 |
| `subagent-background.test.mts` | `z1-cli/tests/` | registry, stop-one-keeps-others, `/new`, `--once` policy |
| `subagent-cancel.test.mts` | `z1-cli/tests/` | a turn abort does not stop a background child |
| `subagent-exit.test.mts` | `z1-cli/tests/` | `/exit` kills a real process; a bystander survives |
| `subagent-turn.test.mts` | `z1-cli/tests/` | a launch keeps its turn; the child is still live after |
| `subagent-ux.test.mts` | `z1-cli/tests/` | footer hint, picker rows and keys, caret cells |

Run one file:

```bash
cd z1-cli
node ../z1-sdk/node_modules/tsx/dist/cli.mjs --tsconfig tsconfig.runtime.json tests/subagent-ux.test.mts
```
