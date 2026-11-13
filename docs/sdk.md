# z1-sdk

The agent engine behind z1. It is a journaled **node-graph runtime**, not a chat
loop: a model call, a tool call, filesystem access, the prompt and the agent
loop itself are nodes with declared inputs and outputs. A workflow is
assembled, not prompted.

The repository is a fork of [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
(MIT, commit `e8f7b93a34`, `0.2.2-rc.2`). Upstream packages are kept as they are;
the `z1/` workspace is new code that composes them. Provenance and removals are
in `NOTICE.md` in the repository.

## Layout

| Path | Package | Contents |
| --- | --- | --- |
| `z1/packages/node` | `@z1/node` | node contract, ports, autonomy, capability types, registry, schemas |
| `z1/packages/graph` | `@z1/graph` | graph document, parsing, validation, digest |
| `z1/packages/runtime` | `@z1/runtime` | executor, cache, resume, events, mock services |
| `z1/packages/nodes` | `@z1/nodes` | built-in core, flow, llm, tool, prompt and memory nodes |
| `z1/packages/journal` | `@z1/journal` | SQLite journal, forks, blobs |
| `z1/packages/memory` | `@z1/memory` | branch-tree context envelope |
| `z1/packages/prompt` | `@z1/prompt` | sections, slots, budgets, hashing, role presets |
| `z1/packages/agent` | `@z1/agent` | the agent loop as a graph, `agent.loop` template |
| `z1/packages/module` | `@z1/module` | manifests, capabilities, store, grants, rollback, audit |
| `z1/packages/node-host` | `@z1/node-host` | child-process worker, capability broker, service adapters |
| `z1/packages/provider`, `provider-openai` | `@z1/provider*` | provider registry and the OpenAI-compatible client |
| `z1/packages/template` | `@z1/template` | template registry and static template files |
| `z1/packages/bootstrap` | `@z1/bootstrap` | config, runtime construction, shared paths |
| `z1/packages/protocol`, `server` | `@z1/protocol`, `@z1/server` | JSON-RPC method map and the WebSocket server |
| `z1/apps/cli` | `@z1/cli` | the `z1` command |
| `z1/apps/canvas` | `@z1/canvas` | React Flow graph editor and run inspector |

Upstream `packages/*` and `apps/*` remain, including the React UI packages the
canvas builds on. The upstream 10-gate build must stay green.

## Node contract

A node is a versioned, categorised definition with a zod config schema and two
kinds of ports:

- **data** — a typed value, pulled from cache. A data port has a schema, is
  `required` by default, and may be a stream (`mode: 'stream'`). Connected
  inputs form a readiness group: `waitFor: 'all'` (default) or `'any'`.
- **flow** — control, ordered, not cached. Flow edges sequence side effects.

```ts
export const chat = defineNode({
  id: 'z1.llm.chat',
  version: '1.0.0',
  category: 'llm',
  config: z.object({ system: z.string().optional() }),
  inputs: [
    { kind: 'data', id: 'messages', schema: z.array(messageSchema) },
    { kind: 'flow', id: 'in' },
  ],
  outputs: [
    { kind: 'data', id: 'text', schema: z.string() },
    { kind: 'flow', id: 'out' },
  ],
  async execute(ctx) {
    const result = await ctx.llm.chat(
      { model: ctx.config.model, messages: ctx.inputs.messages },
      (delta) => ctx.emit('text', delta.text),
    )
    return { text: result.message.content, out: true }
  },
})
```

`ctx` carries `runId`, `nodeId`, the hierarchical `path` (for example
`loop1/0/chat`), `attempt`, the run seed as `input`, the parsed `config`, an
`AbortSignal`, `log`, `emit`, and the ambient services `llm`, `tools`, `fs`,
`artifacts`, `approval` and `runSubgraph`. Module nodes get the same services
brokered across the process boundary except `runSubgraph`, which is not
available to worker code (`z1/packages/node-host/README.md`).

Autonomy is per node instance: `auto`, `ask` or `deny`. The effective policy is
the graph node's `autonomy`, then the definition's, then `auto`. `ask` calls the
approval service before `execute`; `deny` fails without running, and a cached
output from another instance cannot satisfy it. Timeouts default to 120 s;
`retry` is linear backoff on thrown errors.

## Graph document

```ts
interface GraphDocument {
  schemaVersion: 1
  id: string
  title?: string
  nodes: GraphNode[]   // id, type, version, config, autonomy?, position, label?
  edges: GraphEdge[]   // id, kind: 'data' | 'flow', from, to
  metadata?: Record<string, Json>
}
```

`validateGraph` rejects duplicate ids, unknown types or versions, unknown ports,
port-kind mismatches, unconnected required inputs, dangling edges, self-edges,
free flow cycles, duplicate data edges and invalid config. Node positions and
labels are display-only: `graphDigest` hashes id, type, version, config,
autonomy and edges, so moving a node does not change the run's identity.

Cycles are legal only inside `z1.flow.loop`, `z1.flow.map` and
`z1.graph.subgraph`. Those carry a nested `GraphDocument` in config.

## Executor

`runGraph` is the only scheduler. Its rules, from `z1/SPEC.md` §6:

1. Root nodes have no incoming data or flow edges. `z1.core.input` is a root and
   receives the run seed.
2. A node runs when every connected data input has a value (or its `waitFor:
   any` group has one), and it holds at least one flow token if it has incoming
   flow edges.
3. On completion, data outputs propagate to connected inputs. If a definition
   declares flow outputs, exactly one must be `true` and only that edge fires;
   otherwise every outgoing flow edge fires.
4. If nothing is runnable: complete when a `z1.core.output` ran, otherwise fail
   with a deadlock error naming the blocked nodes.
5. Every config and output value is parsed with its zod schema. A parse failure
   fails the node, not the run silently.
6. The per-run cache key is a sha256 of canonical `{type, version, config,
   inputs, input}`. A hit emits `node.cached` and reuses outputs; `cacheable:
   false` disables it.
7. Timeout, retry, approval and cancellation behave as described above.
8. Nested runs share services, registry and journal, and prefix the execution
   path with the iteration label. A fork inside a loop therefore replays only
   the earlier iterations.

Events: `run.started` (input + digest), `node.started`, `node.stream` (not
journaled), `node.cached`, `node.completed`, `node.failed`, `run.completed`,
`run.failed`, `run.cancelled`. Nested runs emit node events only.

Built-in node ids are namespaced `z1.`: `core.input/output/const/template/get/
concat/message/last/merge/value/fail`, `flow.if/loop/map`, `graph.subgraph`,
`llm.model/chat`, `tool.call`, `memory.context`, `prompt.section/stack/text/
context`. The full port table is `z1/SPEC.md` §7.

## Journal, resume and fork

`@z1/journal` uses `node:sqlite` (`DatabaseSync`). Tables: `runs`, `events`,
`blobs`. Default path is `.z1/runs.db` under the project directory. Appends are
synchronous.

- `run.started` records the input and its digest. Resuming with a different
  input fails with `RESUME_MISMATCH` instead of replaying stale output.
- Resume replays `node.completed` outputs and approval decisions, then continues
  at the first unfinished path.
- A suspended run (`approval.requested` with no decision) is persisted, not a
  parked process. `z1 run --resume <runId>` continues it.
- A fork records `parent_run_id` and `fork_seq`, and reads its parent's prefix
  through the chain (depth 64) rather than copying events. Fork points use the
  run's **own** sequence numbers; `run.get` and `run.subscribe` replay **merged**
  numbering (parent prefix plus own), which is the trap in
  [gotchas.md](gotchas.md).
- Blobs are content-addressed and used for artifacts.

## Branch memory

`@z1/memory` turns the run tree into context: the ancestor spine (root first),
siblings forked from the same parent, this run's recent node outputs, and
in-memory summaries. The envelope is rendered under an explicit untrusted-history
heading and capped (4000 characters, 8 recent nodes, 3 ancestor nodes by
default). A sibling records what was tried, not what is true.

`z1.memory.context` is a root node that produces `text` and `messages`;
`z1 memory context <runId>` prints the same envelope for inspection. Summaries
are not persisted: the journal remains the single source of truth.

## Prompt as nodes

The prompt is a graph, not a string built by the runtime.

- `z1.prompt.section` renders a body with `{{path}}` variables from a vars object,
  under a slot (`core`, `role`, `task`, `context`, `contract`, `custom`), a
  priority and a `required` flag.
- `z1.prompt.stack` orders by slot, priority and id, deduplicates, drops optional
  sections over the token budget, and outputs `text`, `sections`, `tokens` and a
  content `hash`.
- `z1.prompt.text` is a literal replacement for a stack.
- `z1.prompt.context` normalises memory, tools, autonomy and graph into vars.

The hash is a node output, so it is journaled with `node.completed`. The prompt
behind a branch is reproducible from the journal. `@z1/prompt` also holds the
default sections and role presets (`DEFAULT_SECTIONS`, `ROLE_SECTIONS`); the
agent graph passes them through `z1.prompt.section`.

## Modules

A module is a package with a manifest, installed with an explicit grant.

```json
{
  "schemaVersion": 1,
  "id": "com.example.echo-llm",
  "version": "1.0.0",
  "title": "Echo LLM service",
  "license": "MIT",
  "engines": { "z1": ">=0.1.0" },
  "entry": "index.js",
  "nodes": [],
  "services": [{ "id": "llm", "kind": "llm", "mode": "worker" }],
  "capabilities": { "fs": { "read": ["${moduleDir}/**"] } }
}
```

Lifecycle: `z1 module propose <dir>` copies and hashes the directory and shows
what it asks for; `approve <id>[@version]` records a grant; `list` shows the
grant, not the manifest; `rollback` returns to the previous version; `audit`
prints the hash-chained log. **The grant is the authority**: it may approve
fewer services or capabilities than the manifest requests, and anything the
grant omits is never started. Approving a module is never implied by
`--approve auto`; module approvals default to `ask`.

Node code runs in a child process under Node's permission model: read access to
the module directory, write access only to declared `fs.write` patterns, no
child process or worker permission. Every privileged call (`fs`, `net`,
`process`, `llm`, `tools`, approval) goes through the host broker, is checked
against the grant, and is audited. Capabilities are declared as `fs.read`,
`fs.write`, `net.hosts`, `process.spawn`, `llm.providers` and `tools.names`.

**Services.** A module may also provide an ambient service of kind `llm`,
`tools`, `fs`, `artifacts` or `memory`.

| Mode | Runs in | Boundary |
| --- | --- | --- |
| `worker` (default) | the module's child, next to its nodes | the same broker; `fs`/`net` calls are grant-checked; approval requests reach only the run that called it |
| `host` | the host process | requires `trusted: true`; the grant is the only boundary |

One child per module hosts its nodes and granted worker services; a crash is
respawned lazily. Service precedence for a run is `overrides > approved service
module > built-in default`; the highest module version wins, ties keep the lower
module id. `--no-modules` disables node and service modules; `--no-service-modules`
disables only services and is not cached.

**Network is not a syscall boundary.** Node's permission model does not block a
direct socket or `fetch` inside a worker. Only `ctx.net` enforces the manifest's
host allowlist. Untrusted modules need an OS-level sandbox that this repository
does not provide (`SECURITY.md`).

## Providers

A provider is configuration, not code. The only wire protocol is
`openai-completions`; any compatible `/chat/completions` endpoint works.

```json
{
  "providers": [
    { "id": "openai", "envKey": "OPENAI_API_KEY", "baseURL": "https://api.openai.com/v1" },
    { "id": "local", "envKey": "LOCAL_API_KEY", "baseURL": "http://127.0.0.1:11434/v1" }
  ]
}
```

A run selects `--model <provider>/<model>`. Without `--live` the mock model is
used; `--live` routes to the configured provider. `Z1_<ID>_BASE_URL` overrides
a base URL (`Z1_OPENAI_BASE_URL` for `openai`). Rate limits and 5xx responses are
retried with backoff and `retry-after`; tune with `Z1_LLM_RETRY_ATTEMPTS`,
`Z1_LLM_RETRY_BASE_MS`, `Z1_LLM_RETRY_MAX_MS`. Secrets are resolved by the host
and never passed into a worker.

## Templates

A template builds a graph document from parameters. `agent.loop` is built in;
static templates are JSON files under `templatesPath` (`.z1/templates`), loaded
lazily so a malformed file only affects template commands.

```bash
z1 template list
z1 run --template agent.loop --message 'hello' --model openai/gpt-4.1-mini --live --json
z1 run --template agent.loop@1.0.0 --message 'hello'
```

`agent.loop` wires prompt sections → `llm.chat` → tool calls → `flow.loop` →
memory, with `--role`, `--system`, `--max-steps` and `--budget-tokens` as
parameters. `--template` flags are rejected without `--template`.

## Bootstrap and config

`@z1/bootstrap` is the one place a process is assembled:

- `loadRuntimeConfig()` resolves `overrides > environment > z1.config.json >
  defaults` for `dbPath`, `graphsPath`, `modulesPath`, `templatesPath`, `live`,
  `useModules`, `useServiceModules`, `useMemory`, `fsMode` and `providers`.
- `createRuntime(config)` builds the journal, provider registry, template
  registry, built-in services and the module runtime; CLI and server share it.
- `builtinRegistry()` registers the built-in nodes.

Environment keys include `Z1_DB`, `Z1_GRAPHS_DIR`, `Z1_MODULES_DIR`,
`Z1_TEMPLATES_DIR`, `Z1_LIVE`, `Z1_USE_MODULES`, `Z1_SERVICE_MODULES`,
`Z1_USE_MEMORY`, `Z1_FS`.

## Protocol and server

`@z1/protocol` is the method map and event types. `@z1/server` binds a WebSocket
endpoint and implements:

```
node.list
graph.validate / list / get / save / delete
run.start / resume / cancel / list / get / subscribe / fork
approval.respond
artifact.get
module.list / propose / approve / rollback / audit
```

`run.event` carries `{ runId, seq, event }`; `run.status` carries status
changes. Authentication is a bearer token in the query string or header; the
server generates one when `--token` is absent. `z1 serve --json` prints
`{ url, token, port }` and nothing else. `z1 studio` additionally serves the
built canvas from the same process and opens a browser unless `--no-open`.

## Canvas

`z1/apps/canvas` is a React + React Flow editor: node palette from
`node.list`, inspector driven by each node's config JSON Schema, autosave with
conflict handling, run overlay, per-node run inspector (outputs, errors,
duration, prompt hash), approval queue and branch tree with fork-from-path. It
is served by `z1 studio` or run with `pnpm z1:canvas`.

## CLI

```bash
pnpm -C z1 build
pnpm -C z1 test
pnpm z1 run --template agent.loop --message '[[tool:echo]] hi'
pnpm z1 run graph.json --input '{"prompt":"hi"}' --json --trace
pnpm z1 tree
pnpm z1 fork <runId> --at <path|seq>
pnpm z1 memory context <runId>
pnpm z1 module propose|approve|list|rollback|audit
pnpm z1 serve --port 4096
pnpm z1 studio
```

Exit codes: 0 completed, 1 failed, 2 invalid graph or usage, 3 cancelled,
4 suspended. `--json` prints one JSON object and nothing else; `--trace` sends
events to stderr. Commands are listed with requirements and expected output in
[commands.md](commands.md).

## Tests

`pnpm -C z1 test` runs 690 tests in 77 files with vitest: graph validation,
executor semantics, cache, resume, fork, memory, prompt rendering, module store
and grants, real worker children (including crash and timeout), provider retry
against a local SSE server, protocol and server end to end, and the CLI. The
canvas has a browser end-to-end test. Upstream `pnpm run build` keeps its 10
gates.

## Limitations

- **No concrete tools in this repository.** The SDK keeps the tools service
  contract and brokered capabilities; `z1-tools` is planned as a separate
  repository. The built-in tool service is a mock.
- **No editor integration.** The canvas and server are standalone.
- **No OS sandbox** for untrusted modules (see above).
- **Canvas is local**: one server token, no multi-user model.
- **Worker services** cannot call `runSubgraph`; `dispose()` is invoked over
  RPC when the runtime closes, then the process is killed.
- **Module-node approvals are ambiguous for concurrent runs**: a request
  resolves to the latest live run.
- **Static templates** are one JSON file per template; there is no template
  package format yet.
