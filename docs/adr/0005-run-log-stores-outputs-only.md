# The run log stores outputs only, one line per atom execution

Every atom execution writes one JSON line: the run ID, the path of node names down to the atom, the duration, the attempt number, and the value on each output port that emitted. A run's first line holds the habitat that started it, its start time, the payload the habitat delivered, the resolved values of any environment references in constants (ADR 0003) and the graph version (a hash of the graph files). Inputs are not stored: an atom's input is rebuilt from its upstream outputs through the wiring and mappings of that graph version, which is exact because mappings only select, rename and nest. The run log is Ligand's own record, written by its runtime. It serves the canvas's last-run values, `ligand logs`, the AI's diagnosis, golden runs, turning failures into tests, and rebuilding parked runs when the workflow engine's database is reset (ADR 0012). Executing, retrying and resuming runs is the workflow engine's job.

## Considered options

- **Store inputs and outputs per atom.** Easier to read line by line, but every value is stored at least twice, and a fan-out stores it once per receiving atom.
- **Derive the log from the workflow engine's tables.** One record instead of two, but those tables are Effect's internal, unstable format, so one Effect release could break `ligand logs`, golden runs and the canvas at once.

## Consequences

- Every node in a graph file has a name, unique in that graph and defaulting to its definition's name, so an atom used twice in one organism gets two distinct paths. Paths, workflow activity names and idempotency keys use these names, and `ligand check` rejects duplicates.
- A path starts at the ecosystem, or at the biosphere if `ligand.yaml` names one.
- Lines are keyed by run, path and attempt, so a replay by the workflow engine never writes a duplicate.
- A failed attempt is a line too, so the log shows which runs are parked; the workflow engine holds them.
- Values larger than a few kilobytes are stored once in content-addressed files and referenced by hash, so the log stays greppable.
- A failed I/O attempt also keeps its `ctx.fetch` exchanges as recordings, so an I/O atom bug can become a failing example.
- The run log is kept for 90 days by default (configurable), and parked runs until they are resolved. The index of execution keys (ADR 0012) is kept as long as the log.
- Secret fields are never written. Personal fields are removed after the retention window (30 days by default; parked runs keep theirs until resolved), and replaced with generated values when a captured input becomes a fixture.
- `biosphere > ecosystem > organism > molecule > atom: 12ms, main: 18 listings` is how `ligand logs` and the canvas display a line, not the stored format.
