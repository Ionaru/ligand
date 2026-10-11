# Ajv validates ports, within a restricted dialect

Values are validated at every port by Ajv (`Ajv2020` with `ajv-formats`, pinned to an exact version of 8.20.0 or later), behind an interface Ligand owns. Ajv is the only maintained candidate that both accepts a regex engine without catastrophic backtracking and emits precompiled validator modules. Its weak raw score in Bowtie, the JSON Schema compliance suite, comes mostly from formats it doesn't load by default and from strict-mode refusals (see `docs/research/runtime-validators.md`).

## Considered options

- **@hyperjump/json-schema.** The best compliance among established libraries (1297 of 1301 Bowtie tests, no failures), but no way to plug in a safe regex engine and no precompiled modules. It checks Ajv instead.
- **Zod, Effect Schema, ArkType or Valibot.** Each translates only part of JSON Schema and follows references only within one file, so they would fit only if their importers covered Ligand's dialect across files, or if ADR 0006 were reversed.

## Consequences

- `ligand check` rejects `$anchor`, `$dynamicRef`, `$dynamicAnchor` and `$data` in type files, which is where Ajv is known to be wrong. Ligand's own `x-ligand-*` keywords are registered with Ajv so its strict mode accepts them.
- Patterns run on `re2js` through Ajv's `code.regExp` option, so matching time is linear in the input. `ligand check` rejects lookaheads and backreferences, which RE2 can't run. Whether precompiled validators can reference `re2js` is checked in the spike from ADR 0012.
- Validators are precompiled by `ligand dev` and `ligand build` (ADR 0015), and are not committed. A logic atom's inputs and outputs are validated inside its worker, within its time limit (ADR 0011). I/O atom outputs and HTTP payloads are validated on the main thread.
- `ligand check` validates every example and fixture value with both Ajv and `@hyperjump/json-schema`, and fails when they disagree.
- This holds for now: TypeScript-first contracts are to be re-evaluated once TypeScript 7.1's programmatic API is available (see `docs/future-improvements.md`).
