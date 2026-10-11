# The store is versioned key-value storage with visible conflicts

Ligand's store is a pair of I/O atoms, Read and Write, over `.ligand/store.db` (ADR 0012), with keys namespaced per ecosystem and values typed like any port. Read also emits a version, and Write takes an optional expected version and has a `conflict` outcome, so two overlapping runs that update the same key produce a visible case in the graph instead of a silently lost update.

## Considered options

- **Last write wins.** Simpler, but with overlapping HTTP runs (ADR 0014) updates would be lost without a trace.

## Consequences

- Stored values carry their type's hash. A value that no longer fits its type comes out on a `stale` outcome, and `ligand check` warns when a stored type changes incompatibly.
- In `ligand dev`, store atoms run for real against a separate dev store that can be reset: the one exception to write atoms being recorded instead of executed.
- Keys can have a time-to-live. Personal data in the store is the developer's responsibility.
