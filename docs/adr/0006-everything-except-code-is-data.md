# Everything except code is data

Contracts, molecule and organism graphs, habitats and example tables are YAML files, and types are JSON Schema kept in shared files. TypeScript holds only implementations and coded tests, and `ligand dev` generates TypeScript types from the schemas. The canvas reads and writes these files without running any user code, and supporting another language only needs a type generator for it.

## Considered options

- **TypeScript-first contracts with a schema library such as Zod.** Familiar to TypeScript developers, but the canvas would have to execute user code to read a contract, and couldn't safely rewrite a TypeScript file someone has edited by hand.
- **Plain TypeScript types read by static analysis.** Nothing extra to write, but TypeScript 7.0 has no stable programmatic API (one is planned for 7.1). This option stays open and is to be re-evaluated once 7.1's API is available.

## Consequences

- Node positions live in a separate layout file per graph, so moving a node never shows up in the wiring diff.
- Files are written with keys in a fixed order, and existing comments are preserved.
- Every file carries a format version (`ligand: 1`).
- Every value on a port is JSON. Bytes travel as base64 strings typed with JSON Schema's own keywords (`contentEncoding: base64`, `contentMediaType`), so validation, the run log and workers need nothing special.
- Values are validated against these schemas at every port, in production too. The validator is Ajv, within a restricted dialect (ADR 0013). A value that fails parks the run at the atom that produced it, and an invalid HTTP payload gets a 400 without starting a run. Effect Schema's JSON Schema import is not used: it supports only a subset of Draft 2020-12 and only references inside one file.
- Wires are type-checked by generating a small TypeScript file per graph that passes each output's type through its mapping into the input it's wired to, and letting the TypeScript compiler check it. This avoids writing a JSON Schema compatibility checker; another language would bring its own check.
