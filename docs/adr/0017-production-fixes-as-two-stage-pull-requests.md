# Production fixes run in CI as two-stage pull requests

When a run parks in production, the runtime calls a webhook. `ligand init` can scaffold a CI job (for example a GitHub Actions `repository_dispatch` workflow) as its target: the job pulls the parked run through the admin endpoint and opens a pull request that contains only the proposed example. A developer approves the example in the pull request (a review comment or label) or with `ligand review`, and only then does the job push the implementation fix to the same branch. Detection, capture and the proposal happen with nobody present; both approvals stay human (ADR 0001).

## Consequences

- The admin endpoint is reachable from beyond localhost only when configured explicitly.
- The admin endpoint does the scrubbing: it serves only scrubbed values. Originals leave production only when an operator asks for them explicitly from their own machine (`ligand pull --original`), never in CI.
- If a captured value is flagged as unscrubbable, such as a whole HTML page on a personal port, the CI job commits nothing and opens the pull request (or an issue) marked "needs manual scrub", without the value.
- Each parked run has a cap on AI attempts and on model spend.
- A proposal committed to a branch carries a marker (`proposed: true` on the example row or golden file); approving removes it. CI refuses to merge while any marker remains, and `resumeOnFix` counts only unmarked examples. Approving with `ligand review` on the fix branch is a commit that removes the marker, and pushing it triggers the job like the pull-request label does. Locally, pending proposals stay in `.ligand/proposals/` until approved.
- For an I/O atom bug, the parked run carries the recordings of the failed attempt, scrubbed by the admin endpoint, with response bodies treated as unstructured personal values unless their type says otherwise. `ligand pull` replays recordings and never calls out live, so such a fix loop may need a manual scrub.
- `ligand pull` reruns the atom on the scrubbed input and reports whether it still fails the same way. If it doesn't, a developer hand-edits a reproduction, or keeps the original value of that field locally, never committed and noted in the run log. The fix loop starts only once a failing example exists.
- Parked runs are resumed, rerun or discarded with `ligand runs` or through the admin endpoint (HTTP and MCP). Nothing resumes on its own, except with the opt-in `resumeOnFix` per organism: after a deploy, a parked run resumes if the failed atom's implementation changed and an approved example covers its captured input.
- Ligand never commits to the main branch.
