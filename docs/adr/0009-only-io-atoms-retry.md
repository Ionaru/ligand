# Only I/O atoms retry; a throwing logic atom parks its run

Technical failures can only happen where the outside world is touched, so only I/O atoms are retried (3 attempts with backoff by default, set per atom in the organism file). A logic atom is pure, so retrying it would fail the same way: when one throws, the run parks immediately and its input goes to the fix loop. Retryable failures are explicit: failures from `ctx.fetch` itself (network errors, timeouts) and anything an I/O atom throws through `ctx.retryable(reason)`, such as a server error. Any other throw from an I/O atom is a bug: the run parks at once and enters the fix loop, just like a throwing logic atom. Responses that are answers, such as a 404, come out as outcomes declared in the atom's contract.

## Consequences

- Retries are activity retries in `effect/workflow`, and every attempt is a line in the run log.
- Resuming a parked run restarts at the failed atom. Earlier atoms return their memoized results from the workflow engine (ADR 0012), so effects that already happened are not repeated.
- Resuming is allowed only if the graph upstream of the failed atom is unchanged since the run. Otherwise the run is discarded or rerun from the start, by explicit choice.
- A crash between an I/O atom's effect and its log line can repeat the effect. Every I/O atom therefore gets an idempotency key (run ID plus atom path) to pass to services that accept one.
