# Commands

Every command here was read from the repository it belongs to. Times are
estimates from the shape of the task, not measurements.

- [Editor (`z1-app`)](#editor-z1-app)
- [Agent core (`z1-core`)](#agent-core-z1-core)
- [Theme (`z1-theme`)](#theme-z1-theme)

---

## Editor (`z1-app`)

### Requirements

| Item | Value | Where it is enforced |
| --- | --- | --- |
| Node | **24.18.0**, same major as `.nvmrc` | `build/npm/preinstall.ts` |
| npm | below 13 | `build/npm/preinstall.ts` |
| yarn | must not be present | `build/npm/preinstall.ts` |
| Electron headers | 43.7.7, fetched by node-gyp | `.npmrc` |
| Python | 3.11 per `BUILD.md` | — |
| System libraries | `git jq python gcc make pkg-config libx11 libxkbfile libsecret krb5 fakeroot` | `BUILD.md` |

```bash
cat .nvmrc && node -v && npm -v && python3 --version
```

If the Node major does not match, the install prints
`*** Please use Node.js v24.18.0 or newer with the same major version (24)` and
throws before installing anything. `VSCODE_SKIP_NODE_VERSION_CHECK=1` bypasses
the gate, which is not recommended: the native headers are Electron's, not
Node's.

### Commands

| Command | What it does | Time | Writes |
| --- | --- | --- | --- |
| `npm ci` | clean install, runs `preinstall` and `postinstall` | 4–10 min cold | `node_modules/`, `build/npm/gyp/node_modules/`, `~/.cache/node-gyp/43.7.7` |
| `npm i` | same, but exits fast when state is unchanged | seconds | install-state hashes |
| `npm run compile` | TypeScript compile | 2–6 min | `out/` |
| `npm run build-fast` | incremental esbuild build | 1–3 min | `out/`, `.build/` |
| `./scripts/code.sh` | dev launch from source | first run, minutes | `.build/electron/` |
| `npm run electron` | download and unpack Electron | ~1 min | `.build/electron/` |
| `npm run gulp vscode-min-prepack` | compile built-in extensions | 3–8 min | `.build/extensions/` |
| `npm run gulp vscode-linux-x64-min-packing` | bundle and package | 5–15 min | `out-vscode-min/`, `out-build/`, `../VSCode-linux-x64/` |
| `./z1-build.sh` | the whole build | 10–25 min | `z1-linux-x64/` |
| `npm run test-node` | node unit tests | 1–5 min | — |
| `./scripts/test.sh --run <file>` | Electron unit tests | 2–10 min | `.build/crashes/` |
| `npm run test-extension` | integration tests | 5–20 min | temp workspaces |
| `npm run typecheck-client` | `tsc --noEmit` | 3–8 min | `.tsbuildinfo` |
| `npm run hygiene` | lint and hygiene gates | 1–6 min | — |

### `z1-build.sh`

1. `set -euo pipefail`, `cd` to its own directory.
2. `NODE_OPTIONS=--max-old-space-size=8192` — gulp and esbuild run out of memory
   without it.
3. `npm ci`.
4. `npm run gulp vscode-min-prepack` — the packing task alone bundles only a
   handful of extensions, so this step is not optional.
5. `npm run gulp vscode-linux-x64-min-packing`.
6. Find `VSCode-linux-x64` in the repository or next to it, else exit 1.
7. `rm -rf z1-linux-x64 && mv "$SRC" z1-linux-x64`.

### Scripts that do not exist

`verify-clean.sh`, `scrub-built-telemetry.sh`, `fix-checksums.sh` and
`rebrand.sh` are not in the tree. Branding is applied by committing the assets;
checksums are written by gulp; telemetry is removed by configuration, not by a
script.

---

## Agent core (`z1-core`)

### Requirements

| Item | Value | Where it is enforced |
| --- | --- | --- |
| bun | **1.4.4** | `package.json` `packageManager`, `packages/script/src/index.ts` |
| git | required, for the version channel | `packages/script/src/index.ts` |
| `OPENCODE_VERSION` | required only for `latest`-channel builds | same |

```bash
export PATH="$HOME/.bun/bin:$PATH"   # bun is not on the default PATH here
bun -v                               # 1.4.4
```

### Commands

| Command | Working directory | What it does | Time | Writes |
| --- | --- | --- | --- | --- |
| `bun install` | root | install, then `postinstall` fixes the pty helper | 1–4 min | `node_modules/`, `bun.lock` |
| `bun run --cwd packages/cli src/index.ts serve --port 4096` | root | start the HTTP server | instant | config under `$HOME` |
| `bun run typecheck:tsc` | root | typecheck every package | 4–10 min | `.turbo/cache`, `*.tsbuildinfo` |
| `bun run check` | root | lint plus typecheck | 4–8 min | as above |
| `bun run lint` | root | oxlint with local plugins | 1–3 min | — |
| `bun test` | `packages/<pkg>` | that package's suite | seconds–minutes | — |
| `bun run --cwd packages/core build` | root | build core to `dist` | 1–3 min | `packages/core/dist/` |
| `bun run build --single` | `packages/cli` | compile the executable for this platform | 1–3 min | `packages/cli/dist/cli-linux-x64/bin/opencode` |
| `bun run build` | `packages/cli` | all 12 targets | 10–25 min | `packages/cli/dist/cli-*/` |
| `bun run pack` | `packages/cli` | pack `dist` into archives | ~1 min | `dist/npm/`, `dist/archives/` |
| `bun run generate` | `packages/protocol` | regenerate the OpenAPI document | ~30 s | `openapi.json` |
| `bun run generate` | `packages/client` | regenerate the client | ~30 s | `src/*/generated/**` |

There is no interactive command in the CLI: the root command carries no
behaviour.

### Server

```bash
OPENCODE_PASSWORD=secret bun run --cwd packages/cli src/index.ts serve --port 4096
curl -u opencode:secret http://127.0.0.1:4096/api/info
```

The username is fixed to `opencode`. Without `OPENCODE_PASSWORD` the server
generates a password and prints `server password <pw>` in the foreground.
`serve --stdio` prints a single JSON line and exits when stdin closes, which is
the form to use when a parent process wants to own the lifetime.

---

## Theme (`z1-theme`)

### Requirements

| Item | Value |
| --- | --- |
| bun | for `script/check-colors.ts` |
| python3 | for `script/make-images.py` |
| rsvg-convert | required by `make-images.py` |
| node/npm | only for `npx @vscode/vsce` |

There are no dependencies and no lockfile: the theme JSON files are the product.

### Commands

| Command | What it does | Time | Writes |
| --- | --- | --- | --- |
| `bun script/check-colors.ts` | rejects `include` chains, malformed hex, colours that read blue after compositing, and dark surfaces in the light theme | <1 s | nothing |
| `python3 script/make-images.py` | redraws the palette card from the theme JSON | 1–3 s | `.github/compare.svg`, `.github/compare.png` |
| `npm run package` | builds the VSIX | 5–20 s | `z1-theme-<version>.vsix` |
| `code --install-extension z1-theme-<version>.vsix` | installs it | seconds | the editor's extension directory |

Installing does not activate the theme. The user still has to pick it:
`Ctrl+K Ctrl+T`.
