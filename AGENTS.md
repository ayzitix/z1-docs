# AGENTS.md

Working notes for anyone — person or agent — who picks up z1. Read this before
touching code, and update it when the state changes.

The goal is that switching between agents costs nothing: the next one reads this
file and knows what is true, what is decided and what is off limits.

---

## What z1 is

A code editor built from the VS Code source (MIT, via VSCodium's build patches)
with an agent core of its own. It is not a wrapper around a binary and not a
plugin: the editor source is in the repository.

Three repositories:

| Repository | Local path | Purpose |
| --- | --- | --- |
| `z1` | `~/Projects/z1-app` | the editor |
| `z1-core` | `~/Projects/z1-core` | the agent core server |
| `z1-theme` | `~/Projects/z1-theme` | the color themes |

The editor is built from `z1-app`. `z1-core` is a separate process. The theme
ships from `z1-theme` and is already vendored into the editor at
`extensions/theme-defaults`.

---

## State

What is done:

- **Editor** builds, runs and is branded. Landing screen, watermark, Carbon
  product icons, square corners, Open VSX gallery, no command center in the
  title bar, Welcome page removed.
- **Core** builds, typechecks across all 14 packages, passes its test suites,
  and serves 138 API paths. The binary compiles for all 12 targets.
- **Theme** is finished: `z1 Ocean` (white), `z1 Default` and `z1 Soft` (dark),
  released as a VSIX and documented with generated palette cards.

What is not done:

- **The core is not connected to the editor.** No panel, no process
  management, no client. This is the next piece of work.
- The core has no LSP client. The config schema exists
  (`packages/schema/src/config/lsp.ts`) but nothing implements it.
- The editor still carries three dark theme variants locally while the theme
  repository ships a different set; they will need to be reconciled when the
  theme is vendored again.

---

## Rules

These are not preferences. They are what the project has decided, mostly after
getting it wrong once.

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
`z1-app`. A theme cannot fix a widget drawn by the editor, and an editor CSS
patch is not a theme change.

### Facts over guesses

Every claim in this repository's documentation cites a file. If something is
unknown, write that it is unknown. Do not fill gaps with plausible details.

---

## Conventions

| Area | Rule |
| --- | --- |
| Documentation | English, in this repository. Code comments in English. |
| Commits | `area: summary` — e.g. `core: fix the pty handoff`. Lowercase after the colon. |
| Versioning | `z1-app` follows the VS Code version. `z1-core` follows its fork base. `z1-theme` is independent. |
| Theme colors | White surfaces, black accents, no hue that is not a state. See `docs/theme.md`. |
| Generated files | `packages/client/src/**/generated/**` is generated. Regenerate it, never edit it. |
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

z1-core/
  packages/core/              agent runtime
  packages/server/            HTTP server
  packages/protocol/          API contract + openapi.json
  packages/client/            generated client
  packages/sdk/               embedded host + public SDK
  packages/cli/               serve, ACP, one-shot runs
  packages/ai/                providers

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
- `z1-core` needs `git` in the working tree to compute its version, and `bun
  test` cannot be run from the repository root.
- A VS Code theme cannot color the watermark, the space behind modern panels, or
  window buttons. Those are editor CSS, not theme keys.
