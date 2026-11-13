# Architecture

One editor, one engine, one theme. The engine is a graph runtime; the editor is
its client (planned).

```
┌──────────────────────────────────────────────────────────┐
│ z1-app — the editor                                      │
│   VS Code 1.143.1 source + z1 branding                   │
│   extensions/theme-defaults  ← the theme                 │
│                                                          │
│   ┌────────────────────────────────────────────────────┐ │
│   │ starts the engine (planned: CLI or in-process)     │ │
│   └───────────────────────┬────────────────────────────┘ │
└───────────────────────────┼──────────────────────────────┘
                            │ CLI · JSON-RPC over WebSocket
                            ▼
┌──────────────────────────────────────────────────────────┐
│ z1-sdk — the agent engine                                │
│                                                          │
│   apps/cli        z1 run/tree/fork/module/serve/studio    │
│   apps/canvas     React Flow graph editor                │
│   packages/server JSON-RPC over WebSocket, run control   │
│   packages/protocol method map, events, transport        │
│                          │                               │
│                          ▼                               │
│   ┌────────────────────────────────────────────────────┐ │
│   │ graph engine                                       │ │
│   │   @z1/node      node contract, ports, autonomy      │ │
│   │   @z1/graph     document, validation, digest        │ │
│   │   @z1/runtime   executor: dataflow + flow           │ │
│   │   @z1/nodes     core, flow, llm, tool, prompt       │ │
│   │   @z1/agent     the agent loop as a graph           │ │
│   └───────┬───────────────────┬───────────────┬────────┘ │
│           │                   │               │          │
│           ▼                   ▼               ▼          │
│   @z1/journal          @z1/memory       @z1/module       │
│   SQLite runs,         branch-tree      manifests,       │
│   forks, blobs         context          grants, audit    │
│                                              │           │
│                                              ▼           │
│                                     @z1/node-host        │
│                                     child process,       │
│                                     capability broker    │
└──────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────┐
│ z1-tools — planned, separate repository                  │
│   concrete read/write/shell/git/todo tools as a module   │
└──────────────────────────────────────────────────────────┘
```

## Editor

`z1-app` is the VS Code source tree at 1.143.1 with VSCodium's build patches and
z1's branding. The application is compiled from `src/`, not wrapped.

What z1 adds on top of upstream:

| Addition | Location |
| --- | --- |
| Landing screen in the empty editor group | `src/vs/workbench/browser/parts/editor/z1Landing.ts` |
| Empty-editor watermark with rotating body and cursor-tracking eyes | `src/vs/workbench/browser/parts/editor/z1Watermark.ts` |
| Three built-in themes | `extensions/theme-defaults/themes/z1-*.json` |
| Carbon product icons as the default | `extensions/icons-carbon/`, default set in `themeConfiguration.ts` |
| Square corners everywhere | `src/vs/workbench/browser/media/style.css` |
| No command center in the title bar | `src/vs/workbench/browser/workbench.contribution.ts` |
| No Welcome page | `src/vs/workbench/contrib/welcomeGettingStarted/` |

The default dark theme is a constant, not a setting:
`COLOR_THEME_DARK = 'z1 Default'` in
`src/vs/workbench/services/themes/common/workbenchThemeService.ts`, together
with `COLOR_THEME_DARK_INITIAL_COLORS` that prevents a blue flash while the real
theme loads.

The editor does not start the engine yet: no panel, no process manager, no
client. That is the next piece of work.

## Agent engine (`z1-sdk`)

The engine is a node-graph runtime derived from DeepSeek Harness (MIT). The
`z1/` workspace is new code that composes the upstream engine at the
provider/tool/session boundary; upstream packages are kept unchanged and their
build gates must stay green.

A run is a graph, not a chat turn. Nodes have typed data ports and ordered flow
ports. The executor pulls data from cache and follows flow tokens; loops and
maps are nested graphs, not free cycles. Every execution is a checkpoint in an
append-only SQLite journal, so a run resumes after a crash and a fork is a
pointer rather than a copy.

Data flow, end to end:

```
CLI / canvas / server
  → graph document (or template)
  → @z1/runtime executor
      → node registry (built-in, module, service)
          → llm · tools · fs · artifacts · memory · approval
              → provider  (OpenAI-compatible HTTP)
              → module worker  (child process, capability broker)
  → @z1/journal  (runs, events, blobs, forks)
  → @z1/memory   (ancestors, siblings, recent steps)
```

The server exposes the same runtime over JSON-RPC/WebSocket:
`node.list`, `graph.*`, `run.start/resume/cancel/get/subscribe/fork`,
`approval.respond`, `artifact.get` and `module.*`. `z1 studio` serves the built
canvas from that server.

Details are in [sdk.md](sdk.md).

## Theme

The theme is a separate repository because a colour theme is a distributable
artifact on its own. It is also vendored into the editor at
`extensions/theme-defaults/themes/`, which is what makes it the default.

Its rules are documented in [theme.md](theme.md). The short version: white
surfaces, black accents, no hue that does not carry a state.

## What is deliberately not here

| Absent | Why |
| --- | --- |
| Telemetry endpoints and update feed | removed by configuration |
| Microsoft Marketplace | Open VSX instead |
| Concrete agent tools | a separate repository (`z1-tools`, planned) |
| Editor ↔ engine integration | planned; the canvas is standalone for now |

