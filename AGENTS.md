# AGENTS.md

Working notes for anyone — person or agent — who picks up z1. Read this before
touching code, and update it when the state changes.

The goal is that switching between agents costs nothing: the next one reads this
file and knows what is true, what is decided and what is off limits.

---

## What z1 is

A code editor built from the VS Code source (MIT, via VSCodium's build patches)
with an agent engine of its own. It is not a wrapper around a binary and not a
plugin: the editor source is in the repository, and the engine is a graph
runtime, not a chat loop.

Three repositories, one planned:

| Repository | Local path | Purpose |
| --- | --- | --- |
| `z1-app` | `~/Projects/z1-app` | the editor |
| `z1-sdk` | `~/Projects/z1-sdk` | the agent engine: graph runtime, CLI, server, canvas |
| `z1-theme` | `~/Projects/z1-theme` | the colour themes |
| `z1-tools` | — | the concrete tools; **planned, no repository yet** |

The editor is built from `z1-app`. The engine is a separate process or library
from `z1-sdk`. The theme ships from `z1-theme` and is already vendored into the
editor at `extensions/theme-defaults`.

## State

What is done:

- **Editor** builds, runs and is branded. Landing screen, watermark, Carbon
  product icons, square corners, Open VSX gallery, no command center in the
  title bar, Welcome page removed. VS Code 1.143.1.
- **Engine** builds, passes 690 tests in 77 files, and runs the agent loop with
  the mock model or a real OpenAI-compatible provider. It has a node/graph
  contract, executor, SQLite journal with resume and fork, branch memory, prompt
  nodes, sandboxed modules with worker services, a JSON-RPC server, a canvas and
  a CLI. The upstream DeepSeek Harness build keeps its 10 gates.
- **Theme** is finished: `z1 Ocean` (white), `z1 Default` and `z1 Soft` (dark),
  released as a VSIX and documented with generated palette cards.

What is not done:

- **The editor does not start the engine.** No panel, no process management, no
  client. This is the next piece of work, and it should start with the smallest
  slice: start the engine, show its status, send one prompt.
- **`z1-tools` does not exist.** The SDK keeps the tools service contract and
  brokered capabilities (`fs.list`, `fs.stat`, `process.run`, worker adapters);
  the built-in tool service is a mock.
- **No OS sandbox** for untrusted modules. Node's permission model is not a
  network boundary.
- **Approval modes beyond per-node `auto | ask | deny`** are not designed.
  Escalation at runtime is a separate decision and is not implemented.

The editor ships the released themes as-is: `z1 Ocean` (light), `z1 Default`
and `z1 Soft`. `z1 Ocean` is the default light theme and `z1 Default` the
default dark theme. `z1 Dark` was dropped.

---

## Rules

These are not preferences. They are what the project has decided, mostly after
getting it wrong once.

### The engine is z1-sdk

`z1-sdk` is the active agent engine. Do not reintroduce an older runtime or
build a second one beside it. The editor integration targets `z1-sdk` only.

### No concrete tools in the SDK

Tools live in `z1-tools`, a separate repository. The SDK owns the tools service
contract, the module capability broker and the worker adapters — not `read`,
`write`, `shell` or `git` implementations. Adding one to `z1-sdk` is a design
change, not a feature.

### Keep the upstream gates green

`z1-sdk` is a fork. Upstream `pnpm run build` (10 gates) must stay untouched and
pass. New code goes in `z1/` and composes upstream packages at their public
boundary; do not edit upstream packages to make a z1 feature fit.

### Commit frequently

Small commits at working boundaries, not one commit at the end of a session.
`z1-sdk` moves in milestones; each task ends with a review of the diff and the
checks, and the lead commits. Uncommitted work is work the next agent has to
rediscover.

### Do not copy

Clean room. The interface of other editors may be studied and measured, but not
copied: no source, no prompts, no assets, no CSS lifted from Cursor or anything
else proprietary. Reimplement from behaviour.

### Branch per experiment

Exploratory work goes on a branch, never on `main`. If an experiment is
abandoned, delete the branch rather than leaving debris on `main`.

### No half-finished work

One piece at a time, finished and verified, rather than five started. If a
change cannot be verified, say so instead of calling it done.

### Verify, then report

Never report a command as successful from its exit code alone — read its output.
A build script that swallows a failure and exits 0 has already cost this project
an hour. For any UI claim, look at pixels, not at the JSON that should have
produced them.

### Only change what you were asked to change

Editing the theme means editing `z1-theme`. Editing the editor means editing
`z1-app`. Editing the engine means editing `z1-sdk`. A theme cannot fix a widget
drawn by the editor, and an editor CSS patch is not a theme change.

### Facts over guesses

Every claim in this repository's documentation cites a file. If something is
unknown, write that it is unknown. Do not fill gaps with plausible details.

---

## Conventions

| Area | Rule |
| --- | --- |
| Documentation | English, in this repository. Code comments in English. |
| Commits | `area: summary` — e.g. `sdk: fix the fork cursor`. Lowercase after the colon. |
| Areas | `editor`, `sdk`, `theme`, `docs`. |
| Versioning | `z1-app` follows the VS Code version. `z1-sdk` is `0.x` until the editor integration lands. `z1-theme` is independent. |
| Theme colors | White surfaces, black accents, no hue that is not a state. See `docs/theme.md`. |
| Generated files | `z1/packages/*/lib`, `z1/apps/*/dist` and `packages/client/src/**/generated/**` are generated. Regenerate them, never edit them. |
| Images in docs | Generated from data by a script, never screenshots. |

---

## Where things live

```
z1-app/
  src/vs/                     editor, workbench, platform
  src/vs/workbench/browser/parts/editor/z1Landing.ts      landing screen
  src/vs/workbench/browser/parts/editor/z1Watermark.ts    empty-editor mark
  extensions/theme-defaults/themes/z1-*.json              built-in themes
  extensions/icons-carbon/                                default product icons
  branding/                   source assets
  product.json                identity, URLs, gallery
  z1-build.sh                 the whole build

z1-sdk/
  z1/packages/node, graph, runtime    the kernel: contract, document, executor
  z1/packages/nodes                   core, flow, llm, tool, prompt nodes
  z1/packages/prompt                  sections, role presets, stacking, hashing
  z1/packages/journal                 SQLite journal, forks, blobs
  z1/packages/memory, agent           branch context; the agent loop as a graph
  z1/packages/module, node-host       modules, grants, worker sandbox
  z1/packages/provider, template, bootstrap   providers, templates, one runtime
  z1/packages/protocol, server        JSON-RPC over WebSocket; headless server
  z1/apps/cli, canvas                 the `z1` command and the React Flow canvas
  z1/SPEC.md, z1/TASKS.md             contract and milestone board
  packages/*, apps/*                  the upstream engine

z1-theme/
  themes/z1-ocean.json        light: white surfaces, black accents
  themes/z1-default.json      dark
  themes/z1-soft.json         dark, lighter
  script/check-colors.ts      the only gate vsce does not provide
  script/make-images.py       palette cards, generated from the JSON
```

---

## Handing over

When you stop, leave the next reader these three things:

1. **What changed** — the commits, and whether they are pushed.
2. **What is verified** — which commands you actually ran and what they printed.
3. **What is next** — the single next step, not a wish list.

Then update the Status section above so the next agent starts from truth rather
than from the last conversation.

---

## Known traps

Summarised here, detailed in [docs/gotchas.md](docs/gotchas.md):

- `z1-app` needs **Node 24** with the same major as `.nvmrc`; the install aborts
  otherwise. `npm` must be below 13, and yarn must not be present.
- The editor build writes `VSCode-linux-x64` **next to** the repository, not
  inside it. `z1-build.sh` moves it.
- Checksums in `product.json` are unpadded base64.
- `pnpm z1` runs with the workspace root as its cwd, so the CLI reads
  `INIT_CWD`; only `run` has `--cwd`.
- Rebuild `z1/packages/node-host/lib/worker.js` after changing the node host.
- `z1 fork --at` counts a run's **own** sequence numbers; `run.get` replay uses
  merged numbering.
- A module grant, not its manifest, is the authority.
- `--no-modules` disables everything; `--no-service-modules` disables services
  only.
- A theme cannot color the watermark, the space behind modern panels, or window
  buttons. Those are editor CSS, not theme keys.
