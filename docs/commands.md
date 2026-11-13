# Commands

Every command here was read from the repository it belongs to. Times are
estimates from the shape of the task, not measurements.

- [Agent engine (`z1-sdk`)](#agent-engine-z1-sdk)
- [Editor (`z1-app`)](#editor-z1-app)
- [Theme (`z1-theme`)](#theme-z1-theme)

---

## Agent engine (`z1-sdk`)

### Requirements

| Item | Value | Where it is enforced |
| --- | --- | --- |
| Node | **24.9.0** or newer | `engines.node` in `package.json` |
| pnpm | **11** (`11.28.5` pinned) | `packageManager` in `package.json` |
| git | not needed to build; used by the repository | — |
| canvas only | a Chromium-based browser for the e2e test | Playwright |

```bash
node -v        # v24.9+ ; v26 is fine
pnpm -v        # 11.x
```

Node 24.9 is the first release with the stable `node:sqlite` API the journal
uses. A missing or older Node fails at import time, not at install time.

### Install and build

```bash
pnpm install
pnpm -C z1 build                 # tsc -b tsconfig.json; no output on success
pnpm -C z1 test                  # vitest; Test Files 77 passed, Tests 690 passed
pnpm z1:canvas:build             # production canvas bundle (for studio)
```

`pnpm -C z1 build` is a solution build: it typechecks every `@z1/*` package and
writes `lib/` next to each `src/`. Run it after changing `@z1/node-host`; the
worker bootstrap `z1/packages/node-host/lib/worker.js` is what child processes
execute.

`pnpm -C z1 clean` removes the build outputs.

### Commands

All commands are run from the `z1-sdk` repository root as `pnpm z1 <command>`.
The wrapper is `tsx z1/apps/cli/src/bin.ts`.

| Command | What it does | Output |
| --- | --- | --- |
| `pnpm z1 list` | list registered built-in nodes | `id  version  category  title`, one per line |
| `pnpm z1 validate <graph.json>` | parse and validate a graph | `ok: <n> nodes, <m> edges` or `code: message` lines, exit 2 |
| `pnpm z1 run <graph.json>` | run a graph document | `status: completed` then `port = value`; `--json` prints one object |
| `pnpm z1 run --template agent.loop --message <text>` | build and run the agent loop | `status: completed` then `output = "..."` |
| `pnpm z1 template list` | list built-in and static templates | `agent.loop@1.0.0  Agent loop` |
| `pnpm z1 tree` | list runs and their lineage | `runId  status  graphId  digest  ISO time  <- parent@seq` |
| `pnpm z1 fork <runId> --at <path\|seq>` | fork a run at a node path or own sequence | new run id, or `{ runId, parentRunId, forkSeq }` with `--json` |
| `pnpm z1 memory context <runId>` | print the branch-tree envelope | markdown sections, or JSON with `--json` |
| `pnpm z1 module list` | installed modules and their grants | id, version, services, capabilities |
| `pnpm z1 module propose <dir>` | hash and stage a module | `proposed <id>@<version>`, capabilities, integrity, next step |
| `pnpm z1 module approve <id>[@version]` | record a grant and install | `approved <id>@<version>`, grant, services |
| `pnpm z1 module rollback <id>` | return to the previous version | `rolled back <id> to <version>` |
| `pnpm z1 module audit` | print the hash-chained audit log | one JSON entry per line |
| `pnpm z1 serve` | headless JSON-RPC/WebSocket server | `z1 listening on ws://127.0.0.1:7331 token=<hex>` |
| `pnpm z1 studio` | server plus the built canvas | `z1 studio on http://127.0.0.1:7331/?token=...` |

Useful flags:

| Flag | Commands | Meaning |
| --- | --- | --- |
| `--json` | all of the above | one JSON value on stdout, nothing else |
| `--trace` | `run` | every run event as one JSON line on stderr |
| `--db <path>` | `run`, `tree`, `fork`, `memory context`, `serve`, `studio` | journal path (default `.z1/runs.db`) |
| `--message <text>` / `--input <json>` / `--input-file <path>` | `run` | run seed; `--message` is plain text, `--input` is JSON |
| `--resume <runId>` / `--suspend` | `run` | continue or pause at an approval |
| `--approve auto\|ask\|deny` | `run` | node autonomy; `ask` prompts on stdin |
| `--module-approve ask\|auto\|deny` | `run` | module node approvals, default `ask` |
| `--model <provider>/<model>` / `--live` | `run --template`, `serve`, `studio` | real provider instead of the mock |
| `--no-modules` / `--no-service-modules` | `run` | disable node modules / service modules |
| `--cwd <path>` | `run` | project directory for config and relative paths |
| `--host` / `--port` / `--token` | `serve`, `studio` | bind address, default `127.0.0.1:7331` |

`--cwd` exists only on `run`. Under `pnpm`, the CLI otherwise uses `INIT_CWD`,
because pnpm runs the script with the package root as the process cwd.

Exit codes: `0` completed, `1` failed, `2` invalid graph or usage, `3`
cancelled, `4` suspended.

### Running the agent loop

```bash
# deterministic mock model; [[tool:echo]] makes the mock emit a tool call
pnpm z1 run --template agent.loop --message '[[tool:echo]] hi'
# status: completed
# output = "mock: done"

# a real OpenAI-compatible endpoint
export OPENAI_API_KEY=sk-...
pnpm z1 run --template agent.loop --live \
  --model openai/gpt-4.1-mini --message 'hello'

# a local server
export OPENAI_API_KEY=local
export Z1_OPENAI_BASE_URL=http://127.0.0.1:11434/v1
pnpm z1 run --template agent.loop --live --model openai/qwen3 --message 'hello'
```

`--model` without `--live` warns and keeps the mock. OpenAI is the documented
example; River and any compatible endpoint are configured the same way. Add a
provider in `z1.config.json` (`id`, `envKey`, `baseURL`) and select it with
`--model <id>/<model>`. `Z1_<ID>_BASE_URL` overrides a base URL. Rate limits and
5xx retries are tuned with `Z1_LLM_RETRY_ATTEMPTS`, `Z1_LLM_RETRY_BASE_MS`,
`Z1_LLM_RETRY_MAX_MS`.

### Server and canvas

```bash
pnpm z1:canvas:build
pnpm z1 studio
# z1 studio on http://127.0.0.1:7331/?token=...
```

`z1 studio` refuses to start with `canvas build not found; run
pnpm z1:canvas:build` when `z1/apps/canvas/dist/index.html` is absent. `z1 serve
--json` prints `{ "url": "ws://…", "token": "…", "port": … }`. The server
authenticates WebSocket requests with `Authorization: Bearer <token>` or
`?token=`.

### Diagnostics

```bash
pnpm -C z1 test --reporter=verbose       # per-test names
pnpm -C z1 test z1/packages/runtime      # one package (path filter)
pnpm -C z1 exec tsc -b tsconfig.json     # same build, without the script
```

The journal is a SQLite file; `sqlite3 .z1/runs.db '.tables'` shows `runs`,
`events` and `blobs`. Inspect it read-only while a command is not running.

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
