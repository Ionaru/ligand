# Production runs a built `dist/` and type-stripped atoms

`ligand build`, run in CI, writes a `dist/` of generated files (precompiled validators, the graph and its hash) under 0BSD. `ligand-runtime start` runs that `dist/` together with the atom sources and refuses to start if the hash doesn't match the graph files, so a production install needs no EUPL code (ADR 0007) and nothing is generated at boot. Atom implementations run directly through Node's type stripping, which is stable since Node 24.12 and needs no compile step.

## Considered options

- **Generate at boot in the runtime.** No build step, but every start would compile validators, and the runtime would have to carry the generator.
- **Compile atoms to JavaScript.** Allows all TypeScript syntax, but adds a build artifact per atom and a gap between the file the AI wrote and the code that runs.

## Consequences

- Ligand's minimum Node version is 24.12.
- `ligand check` rejects syntax that type stripping can't run: enums, namespaces with runtime code, parameter properties, import aliases and decorators.
- Node refuses to strip types inside `node_modules`, so atom packages ship compiled JavaScript. A package lists its atoms in `package.json` (`ligand.atoms`), each with its contract and types, and graphs refer to them by package path, such as `@ligand/atoms-http/request`.
- Whether type stripping works inside worker threads (ADR 0011) is added to the spike from ADR 0012.
