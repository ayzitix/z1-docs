<div align="center">

# z1 docs

**How z1 is built, how to run it, and how to work on it without breaking it.**

</div>

z1 is a code editor built from the VS Code source, with its own agent core and
its own theme. Three repositories, one product:

| Repository | What it is | Stack |
| --- | --- | --- |
| [z1](https://github.com/ayzitix/z1) | the editor — VS Code source with z1 branding | TypeScript, Node 24, gulp |
| [z1-core](https://github.com/ayzitix/z1-core) | the agent core — a local HTTP server the editor talks to | TypeScript, Bun, Effect |
| [z1-theme](https://github.com/ayzitix/z1-theme) | the color themes, shipped alone so they can be installed anywhere | JSON only |

This repository holds the documentation for all three. It is written for
someone who has just cloned a repository and needs to know what to type, and
for an agent that needs to continue the work without rediscovering it.

## Start here

- **[docs/commands.md](docs/commands.md)** — every command, with requirements,
  expected time and the files it writes. Start here if you want to build or run.
- **[AGENTS.md](AGENTS.md)** — the ground rules: what is done, what is not, what
  not to touch, and how to hand work to the next person or agent.
- **[docs/architecture.md](docs/architecture.md)** — how the editor, the core and
  the theme fit together.
- **[docs/gotchas.md](docs/gotchas.md)** — the traps that cost hours.

## The short version

```bash
# editor
cd z1-app
nvm use 24.18.0 && npm ci && ./z1-build.sh
./z1-linux-x64/bin/z1

# agent core
cd z1-core
bun install
bun run --cwd packages/cli src/index.ts serve --port 4096

# theme
cd z1-theme
bun script/check-colors.ts && npm run package
```

## Status

The editor builds and runs. The agent core builds, typechecks and serves its
API, and is not yet wired into the editor. The theme is finished and released.
Details and the next steps are in [AGENTS.md](AGENTS.md#state).

## License

MIT. z1 is derived from VS Code, VSCodium and opencode; the original notices
are kept in each repository.
