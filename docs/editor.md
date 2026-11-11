# Editor

The editor is the VS Code source at **1.143.1** (commit
`e646868227c3e3a9e3f24277c8c7220be42c0be7`, package version `1.143.17502`) with
VSCodium's build patches and z1's branding. It is compiled from `src/`, not a
wrapper around a binary.

Build and run commands are in [commands.md](commands.md).

## Identity

| Surface | File | Value |
| --- | --- | --- |
| Product identity | `product.json` | `nameShort` / `nameLong` / `applicationName` = `z1` |
| Data folder | `product.json` | `.z1` (so configuration lands in `~/.config/z1`) |
| URL protocol | `product.json` | `z1://` |
| Linux icon name | `product.json` | `z1` |
| Gallery | `product.json` | Open VSX |
| Update feed | `product.json` | empty |
| Package | `package.json` | `name: z1`, `version: 1.143.17502` |
| App icon | `resources/linux/code.png`, `code.svg` | z1 mark |
| Title-bar mark | `src/vs/workbench/browser/media/code-icon.svg` | z1 mark |
| Launcher | `resources/linux/bin/code.sh` | templated, becomes `bin/z1` |
| Source assets | `branding/` | SVG, PNG 16–512, ICO, theme originals |

Two icons are easy to confuse: `resources/linux/code.*` is the window and
application icon, `src/vs/workbench/browser/media/code-icon.svg` is the small
mark inside the title bar. Changing one leaves the other.

## Components z1 adds

**Landing screen.** `src/vs/workbench/browser/parts/editor/z1Landing.ts`, styled
by `media/z1Landing.css`. It renders inside an empty editor group rather than as
a tab, and only when no folder or workspace is open. Cards: Open project, Clone
repo, Connect via SSH, About z1.

**Watermark.** `src/vs/workbench/browser/parts/editor/z1Watermark.ts` draws an
SVG into `.letterpress`, referenced from `editorGroupWatermark.ts`. The body
rotates and the eyes follow the cursor. Colour comes from
`--vscode-editor-foreground`, eyes from `--vscode-editor-background`, both in
`media/editorgroupview.css`. **Its opacity (`0.18`) and rotation are hard-coded
in that CSS** — a theme cannot change them.

**Themes.** `extensions/theme-defaults/themes/z1-*.json`, contributed from
`extensions/theme-defaults/package.json`. The default dark theme is a constant:
`COLOR_THEME_DARK = 'z1 Default'` plus `COLOR_THEME_DARK_INITIAL_COLORS` in
`src/vs/workbench/services/themes/common/workbenchThemeService.ts`. The initial
colours matter: without them the window flashes blue while the theme loads.

**Product icons.** `extensions/icons-carbon/` (Carbon Icons), made the default by
`registerDefaultConfigurations` in
`src/vs/workbench/services/themes/common/themeConfiguration.ts`.

## Changes relative to upstream

Grouped by area, from the commit history:

| Area | Files |
| --- | --- |
| Branding | `product.json`, `package.json`, `resources/linux/*`, `branding/*`, `README.md`, `BUILD.md`, `NOTICE.md` |
| Theme | `extensions/theme-defaults/themes/z1-*.json`, `workbenchThemeService.ts`, `themeConfiguration.ts`, `extensions/icons-carbon/*` |
| Landing and watermark | `z1Landing.ts`, `media/z1Landing.css`, `editorGroupView.ts`, `z1Watermark.ts`, `editorGroupWatermark.ts`, `media/editorgroupview.css` |
| Welcome removal | `contrib/welcomeGettingStarted/browser/gettingStarted.ts`, `gettingStarted.contribution.ts`, `media/gettingStarted.css` |
| Title bar | `browser/workbench.contribution.ts` — command center default `true` → `false` |
| Global UI | `browser/media/style.css` — all corner radii `0`, empty-editor shortcut list hidden |
| Build | `z1-build.sh` |

## Repository layout

| Path | Contents |
| --- | --- |
| `src/vs/` | editor, workbench, platform |
| `extensions/` | built-in extensions, including the themes and Carbon icons |
| `build/` | gulp build system |
| `resources/` | icons, Linux desktop and packaging |
| `branding/` | z1 logo, PNGs, ICO, theme originals |
| `out/`, `out-build/` | compiled JavaScript |
| `out-vscode-min/` | staging for the packaged build |
| `.build/` | intermediate extensions and Electron |
| `z1-linux-x64/` | the portable built application |
| `cli/`, `remote/`, `scripts/`, `test/` | CLI, remote server, dev and test scripts |

## Dev loop

For changes to `src/`, `./scripts/code.sh` is enough: it compiles what changed and
launches the development build. Reserve `z1-build.sh` for producing the portable
application.

## Telemetry

No script and no host block: the telemetry level defaults to `none`, the crash
reporter defaults to disabled, and `product.json` configures no endpoint. See
[decisions.md](decisions.md).
