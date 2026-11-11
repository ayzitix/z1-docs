# Architecture

Three pieces, three processes, one product.

```
┌─────────────────────────────────────────────┐
│ z1-app — the editor                         │
│   VS Code 1.143.1 source + z1 branding      │
│   extensions/theme-defaults  ← the theme    │
│                                             │
│   ┌──────────────────────────────────────┐  │
│   │ spawns as a child process (planned)  │  │
│   └──────────────────┬───────────────────┘  │
└──────────────────────┼──────────────────────┘
                       │ HTTP + SSE
                       │ basic auth: opencode:<password>
                       ▼
┌─────────────────────────────────────────────┐
│ z1-core — the agent server                  │
│   packages/cli      serve, ACP, one-shot    │
│   packages/server   138 API paths           │
│   packages/protocol contract + OpenAPI      │
│   packages/core     sessions, tools,        │
│                     providers, permissions  │
│   packages/ai       model providers         │
└─────────────────────────────────────────────┘
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

## Agent core

`z1-core` is a fork of `anomalyco/opencode` stripped to the core: the terminal
interface, the web interface, the desktop app, the interface packages, the
simulation harness and the hosted services are gone. What remains is the runtime
and the server that exposes it.

Request flow:

```
client → HTTP /api (Effect HttpApi)  +  SSE event feed
       → packages/server/src/process.ts   ServerProcess.start
       → packages/server/src/routes.ts    createRoutes
       → service graph (LayerNode)
       → packages/core                     session, tool, provider, permission, mcp
       → packages/ai                       provider call
```

Key entry points:

| What | File |
| --- | --- |
| Server start | `packages/server/src/process.ts` |
| Route construction | `packages/server/src/routes.ts` |
| CLI commands | `packages/cli/src/commands/commands.ts` |
| Child-process lifecycle | `packages/cli/src/server-process.ts` |
| ACP server | `packages/cli/src/acp/` |

Authentication is HTTP basic with the fixed username `opencode` and a password
from `OPENCODE_PASSWORD`. A signed cookie is also accepted, which is what makes
the browser-based pairing flow work.

## How the two are meant to meet

The editor starts the core as a child process and talks to it over HTTP. The
core already supports being owned that way:

- `serve --stdio` prints one JSON line with the URL and exits when stdin closes,
  so the parent controls the lifetime.
- `packages/sdk` can also run the host in-process, which is the alternative if a
  child process turns out to be awkward.

Nothing in the editor does this yet. There is no panel, no process manager and no
client wiring. That is the next piece of work, and it should start with the
smallest possible slice: start the process, show its status, send one prompt.

## Theme

The theme is a separate repository because a colour theme is a distributable
artifact on its own. It is also vendored into the editor at
`extensions/theme-defaults/themes/`, which is what makes it the default.

Its rules are documented in [theme.md](theme.md). The short version: white
surfaces, black accents, no hue that does not carry a state.

## What is deliberately not here

| Removed | Why |
| --- | --- |
| Terminal interface, web UI, desktop app | the editor is the UI |
| GUI plugin surface (`plugin.gui`, `./gui-experimental`) | it exists to draw windows inside a desktop app |
| LSP client | not implemented in this fork; only the config schema remains |
| Telemetry endpoints and update feed | removed by configuration |
| Microsoft Marketplace | Open VSX instead |
