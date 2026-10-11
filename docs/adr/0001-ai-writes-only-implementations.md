# AI writes only implementations; contracts and tests stay with developers

Every node that AI writes keeps its contract, its tests and its implementation in separate files, and an AI change may touch only the implementation file. Ligand enforces this where it hands work to an AI, because only there is it known which changes are the AI's: `ligand generate` and `ligand fix` snapshot every file except the target implementation before a task, and reject the result if anything else changed. An agent's own permission settings aren't enough, since they don't cover every way it can write files. When a production failure becomes a new test, the AI proposes the expected output and a developer approves that test before the AI writes a fix; if the contract and tests can't all hold, the AI returns an escalation instead of forcing the tests to pass.

## Considered options

- **Contract and implementation in one file.** Fewer files, but "AI never changes the contract" would be a promise rather than a rule a check can enforce.
- **AI sets the expected output of a captured failure.** A captured input has no known right answer, so the AI would be writing the test it is then held to.
- **Approve only the tests, not the implementation diff.** The long-term goal, but in v1 implementation diffs are still reviewed like any other code until the tests have proven strong enough.

## Consequences

- Example and golden-run proposals reach the repository only through the approval step (the canvas, `ligand review`, or the pull-request approval in ADR 0017), never from the agent directly.
- On Ligand's own fix branches (ADR 0017), CI checks that the fix job's commits touch only implementation files and approved proposals.
- An agent a developer drives interactively, through MCP or skills, is guided by the steering files (ADR 0004) but is outside the enforced rule, because Ligand can't attribute its edits.
- Models under pressure to pass tests tend to edit or special-case them. In ImpossibleBench (Zhong, Raghunathan and Carlini, arXiv 2510.20270, October 2025), Claude models modified the tests in more than 79% of the transcripts where they cheated, which is why the path rule is enforced rather than requested.
- The same study found that offering a way to abort cut cheating sharply for some models (GPT-5: 54% to 9%) but much less for Claude Opus 4.1. Escalation is a first-class result rather than an error, but it is not enough on its own.
- The AI never re-blesses a golden run (ADR 0016); updating one is a proposal a developer approves.
- `ligand check` runs mutation testing on every logic atom whose implementation changed and fails below a configurable score (80% by default), so weak tests are caught before an AI is held to them. There are no hidden tests in v1, because an external agent can read the whole repository.
