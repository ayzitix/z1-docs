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

## The agent is a graph, not a chat loop

**Decided:** the top-level primitive is a graph run. A model call, a tool call,
filesystem access, the prompt and the agent loop are nodes with typed data ports
and ordered flow ports. The loop is a built-in graph, not hidden control flow.

**Because:** a chat loop cannot be read, diffed, resumed or forked as a unit. A
graph can. Data ports give caching and validation at the boundary; flow ports
make side-effect order explicit. A loop is then a nested graph with an iteration
label, which is also what makes a fork inside a loop replay only the earlier
iterations.

**Cost:** the graph is more ceremony than a prompt. The built-in templates and
the canvas absorb that, and new workflows are assembled rather than prompted.

**Would revisit if:** the node contract cannot express a workflow without a new
node type every time. That would mean the contract is wrong, not that the graph
is.

## Fork DeepSeek Harness, but keep z1 in its own layer

**Decided:** `z1-sdk` is derived from DeepSeek Harness (MIT), and every z1
addition lives in the `z1/` workspace. Upstream packages are not edited.

**Because:** the upstream engine already provides the provider, tool and session
boundary the graph composes. Keeping z1 in `z1/packages` means an upstream
merge is a merge, not a negotiation, and the upstream 10-gate build stays
meaningful.

**Cost:** two dependency graphs in one repository, and z1 cannot fix an upstream
bug in place without a patch.

## Everything is a module

**Decided:** nodes, services, providers, prompt sections and graph templates are
packages or data with manifests. The executor resolves them; the CLI does not
import a node directly.

**Because:** one extension mechanism means one grant model, one audit log and
one install path. A provider is data in `z1.config.json`; a service is a
manifest entry; a node is an export from the same package.

**Cost:** indirection. A one-line change to a built-in still goes through the
registry.

## The prompt is nodes

**Decided:** prompt sections are nodes, the stack is a node, and the rendered
hash is a node output that the journal records.

**Because:** the prompt behind a run must be reproducible from the journal. If
the prompt were assembled inside the LLM node, the model call would be the only
record of what was sent, and a fork would silently change it.

**Cost:** more nodes in the agent graph and a canvas editor for sections.

## Memory is the run tree

**Decided:** memory is built from the journal's tree — ancestors, siblings and
recent steps — not from a vector index or a rolling summary.

**Because:** a fork is already a first-class operation; the tree is the truth
about what was tried. A sibling records a failed approach, which a summary
would erase. It also needs no extra store.

**Cost:** context quality depends on how well node outputs are named, and the
envelope is capped, so very old context is dropped.

## Worker services, with a host trust exception

**Decided:** a module service runs in the module's child process by default. A
service may run in the host only with `trusted: true`, and the manifest says
`mode: "host"` explicitly.

**Because:** a service is ambient capability, so the safe default has to be the
sandbox. Host mode exists for code the user wrote and is willing to run with
full reach; making it explicit keeps an approved manifest honest.

**Cost:** two service paths to test, and a service-only module still costs a
child process.

## Tools are a separate repository

**Decided:** the SDK owns the tools service contract and the brokered
capabilities; concrete tools (`read`, `write`, `shell`, `git`, `todo`) live in
`z1-tools`.

**Because:** tools change faster than the engine and carry their own security
posture. A separate repository can release on its own cadence and be approved
as a module; the SDK stays a runtime.

**Cost:** the SDK ships a mock tool service, so a bare install cannot edit a
file.

## OpenAI-first documentation

**Decided:** examples and docs use an OpenAI-compatible endpoint as the primary
case. River is documented as one compatible endpoint, and a custom provider is a
config entry.

**Because:** the wire protocol is `openai-completions`; naming one vendor as the
default would imply a gateway or an account that does not exist. A local server
and a hosted one use the same code path.

## One bootstrap, one config

**Decided:** `@z1/bootstrap` builds the runtime (`createRuntime`,
`loadRuntimeConfig`) and both the CLI and the server use it. Precedence is
`overrides > environment > z1.config.json > defaults`.

**Because:** two processes that assemble services differently produce two
different products. One resolver means a flag in the CLI and a config key in the
server cannot drift.

**Cost:** config changes are central and need a test per key.

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
