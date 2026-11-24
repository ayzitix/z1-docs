<div align="center">

# z1 docs

**How z1 is built, how to run it, and how to work on it without breaking it.**

</div>

z1 is a code editor built from the VS Code source, with its own agent engine and
its own theme. The engine is a journaled node-graph runtime.

| Repository | What it is | Stack |
| --- | --- | --- |
| [z1](https://github.com/ayzitix/z1) | the editor — VS Code 1.143.1 source with z1 branding | TypeScript, Node 24, gulp |
| [z1-sdk](https://github.com/ayzitix/z1-sdk) | the agent engine — a node-graph runtime, CLI, server and canvas | TypeScript, Node 24.9+, pnpm 11 |
| [z1-cli](https://github.com/ayzitix/z1-cli) | the interactive terminal agent: REPL, slash commands, subagents | TypeScript, Node 24 |
| [z1-tools](https://github.com/ayzitix/z1-tools) | the concrete tools as the `com.z1.tools` module: fs, search, shell, git | TypeScript |
| [z1-theme](https://github.com/ayzitix/z1-theme) | the colour themes, shipped alone so they can be installed anywhere | JSON only |

The editor integration of `z1-sdk` is planned. The canvas is currently a
standalone app.

This repository holds the documentation for all of them. It is written for
someone who has just cloned a repository and needs to know what to type, and
for an agent that needs to continue the work without rediscovering it.

## Start here

- **[docs/commands.md](docs/commands.md)** — every command, with requirements
  and expected output. Start here if you want to build or run.
- **[AGENTS.md](AGENTS.md)** — the ground rules: what is done, what is not, what
  not to touch, and how to hand work to the next person or agent.
- **[docs/architecture.md](docs/architecture.md)** — how the editor, the engine
  and the theme fit together.
- **[docs/sdk.md](docs/sdk.md)** — the engine in depth: nodes, graphs, journal,
  memory, modules, providers, protocol and canvas.
- **[docs/engineering.md](docs/engineering.md)** — the engineering handbook: who
  owns what, where state lives, and the contracts the executor, journal, prompt,
  modules and server keep.
- **[docs/hardening.md](docs/hardening.md)** — failure contracts from the
  hardening pass: constants, error strings, the test that pins each, and the
  limits that remain.
- **[docs/subagents.md](docs/subagents.md)** — child agents: the grant, the
  background default, and what a user sees in the terminal.
- **[docs/gotchas.md](docs/gotchas.md)** — the traps that cost hours.

## The short version

```bash
# editor
cd z1
nvm use 24.18.0 && npm ci && ./z1-build.sh
./z1-linux-x64/bin/z1

# agent engine
cd z1-sdk
pnpm install
pnpm -C z1 build && pnpm -C z1 test
pnpm z1 run --template agent.loop --message '[[tool:echo]] hi'

# terminal agent
cd z1-cli
z1                       # or: node --experimental-strip-types chat.ts
z1 --once "summarise this repo"

# theme
cd z1-theme
bun script/check-colors.ts && npm run package
```

## Status

The editor builds and runs, and the themes are shipped in it. `z1-sdk` builds,
runs the agent loop with a mock model or a real OpenAI-compatible provider, and
passes 1068 tests (21 fail on a Windows machine without symlink privilege and
with a hanging `serve` suite; see [commands.md](docs/commands.md)). The terminal
agent works against the real tools module and runs subagents in the background
by default. The editor does not yet start the engine. Details and the next steps
are in [AGENTS.md](AGENTS.md#state).

## License

MIT. z1 is derived from VS Code and VSCodium, and the engine from DeepSeek
Harness; the original notices are kept in each repository.
