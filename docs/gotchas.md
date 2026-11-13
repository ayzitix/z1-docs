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

## Agent engine (z1-sdk)

**`pnpm z1` does not run where you think it does.**
`pnpm` runs a workspace script with the package root as `process.cwd()`, so a
relative `--db`, `z1.config.json` or graph path resolves against the repository,
not the directory the caller typed. The CLI reads `INIT_CWD` to recover the
caller's directory (`z1/apps/cli/src/bin.ts`). Only `run` also accepts
`--cwd <path>`; the other commands use `INIT_CWD` alone. A script that changes
directory without preserving `INIT_CWD` will quietly write `.z1/` in the wrong
place.

**Rebuild `lib/worker.js` after changing the node host.**
The child process runs the built `z1/packages/node-host/lib/worker.js`, not the
`src/` file. `resolveWorkerBootstrap` prefers `./worker.js` next to `host.js`,
then `../lib/worker.js` when running from `src`, and only then `worker.ts`. A
stale build means tests exercise yesterday's broker and a fix appears not to
work. `pnpm -C z1 build` is the fix.

**A fork's sequence numbers are not the ones `run.get` reports.**
`journal.fork` takes `--at <seq>` in the run's **own** numbering
(`journal.ownedEvents`), because a fork only copies its parent's prefix. The
server's `run.get` and `run.subscribe` replay **merged** numbering: the parent
prefix plus this run's events (`TeeJournal`). Picking a seq from a `run.get`
replay and passing it to `z1 fork --at` selects the wrong event, and can be
past the end. `--at <nodePath>` avoids the mismatch entirely.

**The grant is the authority, not the manifest.**
`z1 module propose` prints what the manifest asks for; `z1 module approve` may
grant less, and `z1 module list` prints the grant. A service or capability the
grant omits is never started or brokered, even though the code and manifest are
present. A pre-services grant (`services: []`) approves no services. Read the
grant before debugging why a service is missing.

**`--no-modules` and `--no-service-modules` are not the same switch.**
`--no-modules` (env `Z1_USE_MODULES`, file key `useModules`) disables node and
service modules, host and worker alike. `--no-service-modules` (env
`Z1_SERVICE_MODULES=0`, key `useServiceModules`) disables only services, and is
per run — a later run uses the service again. A service-only child may still
start if the runtime was constructed with services enabled and a later run
disables them.

**Worker and host services fail differently.**
`mode` defaults to `worker`; `mode: "host"` requires `trusted: true` and an
approval, and runs in the host process with full reach. Worker services go
through the broker: `fs`/`net` calls are grant-checked and an approval request
is refused unless it carries a known run id. A legacy manifest with
`trusted: true` and no `mode` is a host service. Do not test one mode and claim
the other.

**Module-node approvals are ambiguous under concurrent runs.**
An approval request from module node code resolves to the latest live run. With
two runs in flight, a decision can go to the wrong one. Approve module nodes
with `deny` or `auto`, or serialise the runs.

**Tools v1 has no approvals.**
The planned tool set (`z1-tools`, a separate repository) runs `write`, `edit`,
`apply_patch`, `move`, `mkdir` and `shell` as soon as the model asks; no tool
calls the approval service. The capability grant is the only boundary, so point
it at a workspace you are willing to let a model rewrite. Do not assume
`--approve ask` covers a tool call: it covers nodes, not tool internals.

**`stat` and `move` are the awkward tools.**
The worker runs under Node's permission flags, which deny `realpath`, `rename`
and `readFile` even on write-granted paths. `stat` therefore uses the brokered
`ctx.fs.stat` (`lstat`, so a symlink reports `symlink`), while containment
(`realpath`), `move` and the atomic-patch rename step go through minimal `node
-e` helpers spawned with `ctx.process.run`. Consequence: `move` is atomic within
one filesystem, but an `EXDEV` move copies and deletes, so an interruption can
leave both copies. `stat` does not follow symlinks; do not build a
workspace-containment check on it alone.

**Rate limits are retried, but only with `--live`.**
Without `--live` the run uses the mock and never contacts a provider; `--model`
alone warns. On a real endpoint, 429/5xx are retried with backoff and
`retry-after` (`z1/packages/provider-openai/src/retry.ts`), tuned by
`Z1_LLM_RETRY_ATTEMPTS`, `Z1_LLM_RETRY_BASE_MS`, `Z1_LLM_RETRY_MAX_MS`. A 4xx
other than 429 is not retried.

**Node's permission model is not a network boundary.**
The worker's `--permission` flags gate the filesystem and child processes, but a
module can still open a socket or call `fetch` directly. Only `ctx.net.request`
enforces the manifest's host allowlist and only brokered calls are audited.
Untrusted modules need an OS sandbox that this repository does not provide
(`SECURITY.md`, `z1/packages/node-host/README.md`).

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
