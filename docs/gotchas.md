# Gotchas

The things that cost hours. Each one is checked against a file.

## Editor

**Node must be 24, same major as `.nvmrc`.**
`build/npm/preinstall.ts` reads `.nvmrc` and throws if the running major differs
or the minor is below 18.0. The error is loud but the cause is easy to miss when
a version manager silently keeps an older shell on `PATH`. It also refuses npm 13
and above, and refuses to run when yarn is present.

**The build output lands outside the repository.**
The gulp task writes `VSCode-linux-x64` to `path.dirname(root)` — that is,
`~/Projects/VSCode-linux-x64`, not `z1-app/VSCode-linux-x64`. Because of that it
is not covered by the repository's `.gitignore`. `z1-build.sh` moves it into
`z1-linux-x64/`.

**Only `vscode-min-prepack` builds the built-in extensions.**
Running the packing task alone produces an application with three extensions
instead of the full set. Both gulp tasks are needed.

**Checksums are unpadded base64.**
`build/gulpfile.vscode.ts` computes `sha256 → digest('base64') → replace(/=+$/,
'')`. Copying a padded value from another tool makes the editor report a corrupt
installation.

**The extension cache survives rebuilds.**
`cacheHome` equals `userDataPath`, and the profile cache lives in
`~/.config/z1/CachedProfilesData`. A stale cache makes a freshly built theme or
icon set not appear. Deleting it is safe; it is regenerated.

**Two different places hold the app icon.**
`resources/linux/code.{png,svg}` is the window and application icon;
`src/vs/workbench/browser/media/code-icon.svg` is the mark in the title bar.
Changing one does not change the other.

**Telemetry is removed by configuration, not by a script.**
There is no scrub script and no host block in the tree. What exists: the
telemetry level defaults to `none`, the crash reporter defaults to disabled, and
`product.json` carries no telemetry or update endpoint. A third-party extension
can still send its own telemetry, and that cannot be fixed by configuration.

## Agent core

**`git` is required to build.**
`packages/script/src/index.ts` derives the version channel from
`git branch --show-current`. Without a git work tree the script fails; on a
detached HEAD the channel becomes `detached`.

**Tests cannot be run from the repository root.**
Both the root `test` script and `bunfig.toml` (`root =
"./do-not-run-tests-from-root"`) block it. Always `cd packages/<pkg>` first.

**`bun install` will fail after a dependency bump if a patch no longer applies.**
Fourteen dependencies are patched in `patches/`. Never delete that directory;
regenerate a patch rather than dropping it.

**A package published minutes ago looks missing.**
`bunfig.toml` sets `minimumReleaseAge = 259200` (three days). The allowlist is
`minimumReleaseAgeExcludes`.

**The core has no LSP client.**
The configuration schema exists in `packages/schema/src/config/lsp.ts`, the
editor-side integration does not. Documentation that claims LSP support is
wrong.

## Theme

**`vsce` does not validate a theme.**
It checks the manifest and packages whatever JSON is there: blue colours,
malformed hex and `include` chains that point at files outside the package all
pass. `script/check-colors.ts` is the only gate.

**A theme cannot colour everything.**
These are drawn by the editor, not by a theme key:

| Element | Why |
| --- | --- |
| Empty-editor watermark | an SVG with `opacity: 0.18` in `editorgroupview.css` |
| The space behind modern-UI panels | a hard-coded `WORKBENCH_BACKGROUND` per theme type |
| Window buttons | `titlebarpart.css` |
| Extension icons in the list | `<img>` from the marketplace |
| Walkthrough screenshots | files chosen by theme type, not recoloured |
| Shadows and elevation | static `rgba()` in `style.css` |

**A missing key is not a neutral key.**
Anything the theme does not set falls back to the registry default for that
theme type. The light defaults include a blue status bar (`#007ACC`), a dark
activity bar (`#2C2C2C`) and a blue selection (`#0060C0`) with white text. This
is why a light theme must cover the inheritance chains rather than the visible
surfaces.

**`menu.selectionBackground` is not what colours a menu.**
Menus take `list.hover*` through `platform/theme/browser/defaultStyles.ts`.
Editing the menu key changes nothing visible. The same trap applies to shortcut
chips, which use `keybindingLabel.*` and not `textCodeBlock.*`.

**Documentation images are generated.**
`.github/compare.png` is rendered from `themes/*.json` by
`script/make-images.py`. Changing a colour without rerunning it leaves a stale
image next to a wrong claim.

## Documentation habits that prevented mistakes

- Measure pixels before concluding a colour is wrong. A screenshot once proved
  that 881 of 1.86 million pixels were warm, all of them in one extension icon.
- Read a command's output, not its exit code. A build script that reported
  success while `npm ci` had failed cost an hour.
- Check whether a claimed script exists before documenting it. Four scripts in
  the editor repository were described in earlier notes and never existed.
