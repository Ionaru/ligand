# Effect is Ligand's runtime, and atoms never see it

Ligand's runtime and tools are built on Effect 4, including modules that Effect still marks unstable (workflow, ai, http, cli, sql), because Ligand intends to grow together with them. Atom implementations never see Effect: a logic atom is a plain synchronous function, and an I/O atom is a plain async function that receives a context (an abort signal, secrets, an idempotency key, a way to park the run, and a `fetch` whose requests and responses are recorded; `ligand check` flags the global `fetch` in an I/O atom). The runtime wraps each atom in Effect for timeouts, retries and durability.

## Considered options

- **Allow atoms written as Effects.** Typed errors and Layers suit I/O, but Effect would become a version range that Ligand shares with every user, which limits Ligand to Effect's stable modules and adds a learning curve for atom authors and for AI.
- **Stable modules only, with Ligand's own durability, plain `node:http` and a separate AI SDK.** Safer against breaking changes, but Ligand would build and maintain what Effect's unstable modules already provide.

## Consequences

- Because Effect never appears in user code or in Ligand's public types, Ligand can use unstable modules without passing that instability on to its users.
- Breaking changes in Effect's unstable modules are Ligand's to absorb, in Ligand's own releases.
- Every exchange through `ctx.fetch` can be saved as a recording (`atoms/<name>/recordings/`). An I/O atom's example is an input, the recordings it plays back and the exact output expected.
- In `ligand dev`, a write atom's `ctx.fetch` sends nothing: it replays the newest matching recording or, if there is none, answers a synthetic empty `200`, and the canvas marks the run as dry. The atom's own code still runs, so its output is real. A write atom that doesn't go through `ctx.fetch`, such as one using a database driver, has to be switched live in dev.
- Every Effect package is pinned to one exact version, as a regular dependency of the runtime and the CLI (not a peer dependency), and upgraded only in a Ligand release whose tests pass, including the listing bot's golden runs. A user's own `effect` version, if any, doesn't have to match.
