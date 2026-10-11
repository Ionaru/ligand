# effect/workflow runs organisms; its database is disposable

Each run is one `effect/workflow` execution of a single generic workflow that walks the organism's graph. The execution key always includes the organism and the name of the habitat that started the run, plus the run ID for HTTP runs, the scheduled time for schedules, or the emitting run and port for events (ADR 0008), and two habitats of one organism never collide. Each run's first line in the run log holds its execution key, and the runtime checks an index of those keys, kept next to the run log, before starting any run, so neither a restart, a replay nor a rebuild on a fresh workflow engine can start the same run twice. Every atom execution, logic atoms included, is an activity named by its path, so completed atoms are memoized and a resumed run never recomputes them with newer code. A parked run is a suspended execution. The engine's database (`.ligand/engine.db`) is disposable: anything that must survive lives in the run log (ADR 0005) or the store (`.ligand/store.db`), and parked runs can be rebuilt on a fresh engine by having completed atoms return their logged outputs, the same mechanism golden runs use.

## Considered options

- **Ligand's own durability on the run log.** Less reliance on unstable modules, but Ligand would build and maintain what `effect/workflow` already provides.
- **Refuse upgrades while runs are parked.** Simpler, but parked runs wait for fixes, so draining them before an upgrade may be impossible.

## Consequences

- Before the runtime is built, a spike proves the risky parts: whether resuming a suspended execution runs the failed atom again as a new attempt (`SuspendOnFailure` with `resume`, or Ligand's own `DurableDeferred` as the fallback); how a workflow start issued from inside another workflow behaves; type stripping in worker threads (ADR 0015); `re2js` in precompiled validators (ADR 0013); `effect/ai`'s MCP server with real agents (ADR 0004); and the cost each atom adds to a run, measured under load (ADR 0011).
- A parked run is not active: it doesn't block a `skip` or `queue` overlap policy, and it doesn't count toward an HTTP habitat's limit of concurrent runs. An HTTP run still going after its 504 does count. Later runs may overtake a parked one; stale store reads in a resumed run show up as a `conflict` (ADR 0018).
- Secret fields are encrypted before they reach the engine's storage, with a key from an environment secret. A run's engine rows are deleted when it completes; parked runs keep theirs until resolved. No supported purge API exists, so this touches Effect's internal tables, which is acceptable because Effect is pinned (ADR 0010).
- A parked run whose upstream outputs include secret fields can't be rebuilt from the run log. It is discarded or rerun, by explicit choice.
- Ligand's CI parks runs on the previous release and resumes them on the new one.
- One process per ecosystem: the runtime (`ligand-runtime start`, which `ligand start` wraps) takes an exclusive lock on `.ligand/`, because Effect's single-process runner doesn't coordinate with other processes.
- Overlap policies apply per habitat: a schedule habitat skips only while its own previous run is still active, whatever runs other habitats of the same organism have going.
- Runs interrupted by a restart resume at their last completed atom, under the same rule as parked runs (ADR 0009): if a deploy changed the graph upstream of where a run stopped, it is parked with the reason "graph changed" instead of resuming.
