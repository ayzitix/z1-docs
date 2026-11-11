# Decisions

Why z1 is built the way it is. Each entry records what was decided, what it
replaced, and what would make us revisit it.

## The editor is the VS Code source, not a wrapper

**Decided:** compile the editor from source rather than skinning a binary or
writing an extension that repaints one.

**Because:** everything z1 wants to change — the empty-editor screen, the
watermark, the title bar, corner radii, the startup colour — lives in the
application, not in an extension's reach. A wrapper can change neither.

**Cost:** the build takes 10–25 minutes and the repository is large. Accepted.

**Would revisit if:** the changes ever fit into an extension, which they do not
while the landing screen replaces an editor tab.

## VSCodium's patches, not VS Code's build

**Decided:** start from the VS Code source and apply VSCodium's build patches.

**Because:** VSCodium already removes the Microsoft marketplace, the update feed
and the telemetry wiring at build time, which is most of what z1 would otherwise
have to redo.

**Cost:** z1 carries a fork of a fork. Upstream merges are manual work.

## Telemetry is off by configuration

**Decided:** no scrub script and no host block. The telemetry level defaults to
`none`, the crash reporter defaults to disabled, and `product.json` carries no
telemetry or update endpoint.

**Because:** a host block is invisible to a reader and fragile; a default plus an
absent endpoint is auditable in the repository.

**Accepted limitation:** a third-party extension can still send its own
telemetry. Blocking that is not a configuration problem.

## Open VSX instead of the Microsoft marketplace

**Decided:** the gallery is Open VSX and the update URL is empty.

**Because:** the Microsoft marketplace's terms do not allow a non-Microsoft
build to use it.

**Cost:** fewer extensions, and some are older.

## White theme as the flagship

**Decided:** `z1 Ocean` — white surfaces, black accents, no hue — is the theme
the project presents, with two dark themes alongside it.

**Because:** every dark theme eventually looks like every other dark theme; a
strictly monochrome light theme is a position rather than a variation, and it
survives being looked at for eight hours.

**Rules that came out of it:** see [theme.md](theme.md). The important part is
that a theme cannot be judged from its JSON: `vsce` will happily package a theme
that renders wrong, so the project keeps its own colour checker and its own
generated palette images.

## The theme is a separate repository

**Decided:** `z1-theme` is installable on its own, and the editor keeps a copy.

**Because:** a colour theme is useful to people who do not use z1, and a separate
repository is the only way to ship a VSIX.

**Cost:** the copies drift. Vendoring is a deliberate step, not a sync.

## The agent core is a server, not a library

**Decided:** `z1-core` runs as a local HTTP server with an SSE event feed, and
the editor talks to it over a generated client.

**Because:** the upstream project already had this shape — the terminal interface
and the web interface were both clients of the same server. Keeping the shape
means the editor is one more client rather than a rewrite, and the core stays
usable from a terminal, from ACP, or from a script.

**Cost:** process management in the editor: spawning, supervising, restarting,
and holding a password.

**Would revisit if:** in-process embedding through `packages/sdk` proves simpler
in practice. The core supports both.

## What was removed from the core

**Decided:** delete the terminal interface, the web interface, the desktop app,
the interface packages, the simulation harness and the GUI plugin surface.

**Because:** they exist to draw windows in a desktop product. Inside a fork whose
interface is a text editor they are dead weight — and the GUI plugin surface in
particular pulled Electron into a package the server depends on.

**Cost:** the fork is further from upstream than it looks, and rebasing means
resolving deletions again.

## Experiments go on a branch

**Decided:** exploratory work never lands on `main`.

**Because:** two earlier experiments — a webview-based glass panel and a
secondary-sidebar agents panel — were built, judged wrong and removed. Keeping
them on branches would have cost nothing; keeping them on `main` cost a revert of
both.

## Documentation is generated where possible

**Decided:** the palette images in the theme repository are rendered from the
theme JSON, not screenshotted.

**Because:** a screenshot documents a moment; a generated card cannot disagree
with the file. The same principle applies here: this documentation cites files
rather than memory.
