# Future improvements

Ideas deliberately left out of v1, and what would bring each one back.

## Design

- **Broader "Improve" task.** In v1 the AI only proposes examples that catch mutants the tests miss. Later: suggesting missing tests, refactors and better splits of atoms.
- **TypeScript-first contracts.** Contracts written as TypeScript types and read by static analysis, with JSON Schema generated from them (ADR 0006, ADR 0013). Once TypeScript 7.1's programmatic API is available, a spike decides, and ADR 0006 changes only if all four criteria pass:
  1. The canvas reads a contract without executing user code.
  2. The canvas edits a hand-edited contract file and keeps its formatting and comments.
  3. Types that refer to each other across files, and the `x-ligand-*` annotations (as JSDoc tags), produce JSON Schema that Ajv and other languages can use.
  4. Wire type checks still work.
- **Streaming bytes.** Ports carry bytes as base64 strings in v1. A streaming type for large files would avoid holding them in memory.
- **Genomes.** Reusable organism blueprints with open slots, once there are enough organisms to see which shapes repeat.
- **Biosphere runtime.** Several ecosystems working together. v1 has the name only.
- **Atoms written as Effects.** Possible later as an opt-in, but it would make Effect a version range shared with users (ADR 0010).
- **Hidden tests.** Not possible while external agents can read the whole repository (ADR 0001).

## Scope (v1 non-goals)

- UI and frontend applications.
- Languages other than TypeScript.
- Serverless deployment.
- Users who don't write code.
- More than one process per ecosystem (ADR 0012).

## Dependencies to re-evaluate

- **Validators.** ata-validator and json-schema-library both pass every Bowtie test but are young or less mature; look again around October 2027. Adopt Effect Schema if its JSON Schema import handles references across files and all of Ligand's restricted Draft 2020-12 dialect, including `unevaluatedProperties` (ADR 0013).
- **AI providers.** Native Gemini and Bedrock support once Effect ships v4 packages for them (ADR 0004).
