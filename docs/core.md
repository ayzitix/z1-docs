# Core

`z1-core` is the agent runtime behind z1. It runs as a local HTTP server and has
no interface of its own: the editor, a terminal or an ACP client is the UI.

A fork of [`anomalyco/opencode`](https://github.com/anomalyco/opencode) (MIT),
stripped to the runtime. What was removed and why is in `NOTICE.md` in that
repository and summarised in [decisions.md](decisions.md).

## Packages

| Package | Role |
| --- | --- |
| `packages/core` | the runtime: sessions, tools, providers, permissions, MCP, worktrees, plugins, snapshots |
| `packages/server` | the HTTP server that exposes the runtime |
| `packages/protocol` | the API contract and the generated OpenAPI document |
| `packages/client` | the generated client, `@opencode/client` |
| `packages/sdk` | `@opencode/sdk` — composes client, core and server, and can run the host in-process |
| `packages/schema` | shared data schemas |
| `packages/ai` | model providers |
| `packages/plugin` | the plugin API |
| `packages/cli` | the executable: `serve`, ACP, one-shot runs |
| `packages/codemode` | confined code execution over tools |
| `packages/util`, `packages/script`, `packages/httpapi-codegen`, `packages/http-recorder`, `packages/plugin-browser` | support |

## Request flow

```
client → HTTP /api  +  SSE event feed
       → packages/server/src/process.ts     ServerProcess.start
       → packages/server/src/routes.ts      createRoutes
       → service graph                      LayerNode
       → packages/core                      session → tool → provider
       → packages/ai                        provider call
```

The server answers `/openapi.json` at runtime, generated from
`packages/protocol`.

## API

| | |
| --- | --- |
| Document | `packages/protocol/openapi.json`, OpenAPI 3.1.0 |
| Size | 138 paths, 169 operations, 32 tags |
| Auth | HTTP basic, username fixed to `opencode` |
| Password | `OPENCODE_PASSWORD`, else generated and printed once |
| Also accepted | `?auth_token=<base64 user:pass>`, or a signed session cookie |

```bash
OPENCODE_PASSWORD=secret bun run --cwd packages/cli src/index.ts serve --port 4096
curl -u opencode:secret http://127.0.0.1:4096/api/info
```

`/api/info` returns the version, pid, bound URLs and temp path. It answers before
the service graph finishes booting, so a client can poll it instead of sleeping.

Do not hand-edit the client: regenerate it.

```bash
cd packages/protocol && bun run generate
cd ../client     && bun run generate
```

## Embedding in the editor

The core is built to be owned by a parent process:

- **`serve --stdio`** prints a single JSON line with the URL and exits when stdin
  closes. The parent owns the lifetime.
- **`OPENCODE_PASSWORD`** passes the credential in, so the parent never parses
  output. In `--stdio` mode the password is removed from the environment the tools
  inherit.
- **`@opencode/sdk`** can run the same host in-process, without a child, if that
  turns out to be simpler.

What the editor will need: start the process, hold the password, poll `/api/info`
until ready, then attach a client. None of this is written yet — see the state
section of [../AGENTS.md](../AGENTS.md).

## ACP

`packages/cli/src/acp/` speaks the Agent Client Protocol over stdio, so an editor
that supports ACP can attach any agent, including this one.

```bash
bun run --cwd packages/cli src/index.ts acp          # ACP server
bun run --cwd packages/cli src/index.ts acp --login  # authenticate first
```

Supported: new, load, list, delete, resume, close and fork a session; set mode
and config option; prompt; cancel. MCP transport is stdio and HTTP; SSE is not
advertised.

## Sessions and tools

A session is durable: it survives restarts, and its history is replayed to the
model through a projection rather than a raw log. Tools run under a permission
service, so an editor can approve, deny or auto-accept per tool.

The pieces worth knowing before changing behaviour:

| Concern | File |
| --- | --- |
| Session lifecycle | `packages/core/src/session/` |
| Tool registry and execution | `packages/core/src/tool.ts`, `tool/` |
| Permissions | `packages/core/src/permission.ts`, `permission/` |
| Providers | `packages/core/src/provider.ts`, `packages/ai/` |
| MCP servers | `packages/core/src/mcp/` |
| Plugins | `packages/core/src/plugin.ts`, `plugin/` |
| Worktrees | `packages/core/src/worktree.ts` |
| Snapshots and revert | `packages/core/src/snapshot.ts` |

## Not implemented

**LSP.** The configuration schema is at `packages/schema/src/config/lsp.ts`, and
the normaliser reads it, but there is no LSP client in `packages/core`. Anything
that claims LSP support is wrong.
