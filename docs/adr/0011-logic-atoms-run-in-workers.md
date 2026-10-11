# Logic atoms run in worker threads with a time limit

JavaScript can't interrupt synchronous code on its own thread, so a logic atom stuck in a runaway computation (for example a regex backtracking on an unexpected page) would freeze every organism in the process, and a timeout around it would never fire. Logic atoms therefore run in a small pool of worker threads with a time limit per call: 1 second by default, adjustable per atom. A call that exceeds it has its worker killed, and its run parks as a logic bug with the input captured for the fix loop.

## Considered options

- **Run logic atoms on the main thread.** No copying of values, but one bad input can stop the whole ecosystem.

## Consequences

- Every value passed to a logic atom is copied into its worker, which is small next to the cost of fetching a page.
- That holds for a schedule that fetches pages, not necessarily for an HTTP organism of many logic atoms under load. Motia was rewritten after the cost it added per step degraded at scale ([research](../research/motia.md)), so the cost each atom adds to a run is measured in the spike from ADR 0012.
- Validating a logic atom's inputs and outputs also happens inside its worker, so it counts toward the time limit (ADR 0013).
- I/O atoms stay on the main thread, with a time limit per attempt (30 seconds by default). A timeout counts as a failed attempt and is retried (ADR 0009).
