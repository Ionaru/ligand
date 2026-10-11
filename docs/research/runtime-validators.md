# Runtime validators for port values

Research date: 2026-10-11.

## 1. Question and constraints

Should Ligand validate port values with Ajv, Zod or another validator? Under ADR 0006, types are JSON Schema Draft 2020-12 in shared YAML files with cross-file `$ref`, and every port is validated in production; a failing value parks the run, and an invalid HTTP payload gets a 400 [23]. Constraints:

- full Draft 2020-12, including `unevaluatedProperties` for closed objects;
- extension keywords such as `x-ligand-personal` (personal fields), accepted and readable;
- all errors with JSON Pointer instance paths, for the canvas and the AI fix loop;
- untrusted input (HTTP habitats, scraped pages);
- compile cost repeats per worker thread (ADR 0011);
- a licence compatible with MPL-2.0 runtime libraries (ADR 0007).

## 2. Recommendation

**Keep Ajv (`Ajv2020`, `ajv` >= 8.20.0), with conditions:** load `ajv-formats`, restrict Ligand's schema dialect to what Ajv handles correctly, plug in a linear-time regex engine, emit precompiled modules, and run `@hyperjump/json-schema` as a reference validator in Ligand's tests. Three facts:

1. **Ajv's raw Bowtie score overstates its gaps.** It passes 1047 of 1301 Draft 2020-12 tests, but about 133 of its 219 errored tests are unknown formats (fixed by `ajv-formats`) and about 48 are strict-mode throws. The real gaps are `$anchor` (19 tests), some `$dynamicRef` cases (8 stack overflows plus about 15 wrong answers), JavaScript property names such as `__proto__`, and open `unevaluated*` bugs [1][2][3]. Ligand can forbid the first two in its dialect.
2. **Ajv alone has a pluggable regex engine** (`code.regExp`, e.g. RE2) [4][7]; the others whose source was checked use native `RegExp` with no hook [10][12][14][15][18][19].
3. **Ajv emits real precompiled modules** (standalone CJS or ESM), so each worker imports validators instead of compiling them [6]. Hyperjump serialises an AST [10]; schemasafe is unreleased since 2023 [14].

This confirms ADR 0006's provisional choice, narrowed to a restricted dialect.

## 3. Comparison

| Candidate | Draft 2020-12 (Bowtie) | Cross-file `$ref` | Custom keywords | All errors, instance paths | Precompilation | Regex safety | Maintenance | Licence |
|---|---|---|---|---|---|---|---|---|
| Ajv 8.20.0 | 1047 pass, 27 fail, 219 error, 8 skip; mostly formats and strict mode [1] | `addSchema` / `schemas` by `$id`; no `$anchor` [2][4] | `addKeyword`; standalone allows only `code`/`macro` [5][6] | `allErrors: true`; `instancePath` JSON Pointer [4] | Standalone CJS/ESM [6] | `code.regExp` hook (RE2) [7] | One active maintainer; 19-month gap 2024-07 to 2026-02 [8][9] | MIT |
| @hyperjump/json-schema 1.18.0 | 1297 pass, 4 deliberate `file:` skips (tested 1.17.7) [1] | `registerSchema`; fs/HTTP plugins [10] | `addKeyword` (experimental); unknown keywords become annotations [10] | Spec BASIC/DETAILED, experimental; all-errors unverified [10] | `serialize()` / `restoreValidator` (AST) [10] | Native, no hook [10] | 13 releases in 12 months; one core maintainer [11] | MIT |
| typebox 1.3.36 (`typebox/schema`) | Not in Bowtie; self-reported full [12] | Context map by `$id`, tested locally [12] | No `addKeyword`; unknown keywords pass through [12] | `Errors()` JSON Pointer; `maxErrors` default 8 [12] | `Code()` in `typebox/compile`, being phased out [13] | Native, no hook [12] | 146 releases in 12 months; bus factor 1 [12] | MIT |
| @exodus/schemasafe 1.3.0 | 1259 pass, 1 fail, 37 error, 4 skip [1] | `schemas` option [14] | None; `allowUnusedKeywords` [14] | `includeErrors` + `allErrors`; some checks skipped [14] | `toModule()` [14] | `complexityChecks` lint only [14] | Last release 2023-08-29 [14] | MIT |
| @cfworker/json-schema 4.1.1 | 1249 pass, 42 fail, 4 error, 6 skip [1] | `addSchema` [15] | None documented [15] | `shortCircuit: false`; `instanceLocation` [15] | None [15] | Native, no hook [15] | Last release 2025-01-31 [15] | MIT |
| ata-validator 1.53.0 | 1301/1301 (tested 1.46.0) [1] | Native walker falls back for cross-document refs [16] | `keywords` option; blocks standalone [16] | `instancePath`, ATA codes; all-errors not checked [16] | `bundleStandalone` [16] | Undocumented (unverified) [16] | About 7 months old [16] | MIT |
| json-schema-library 11.6.6 | 1301/1301 (tested 11.6.2) [1] | `addRemoteSchema` [17] | Draft extensions [17] | `data.pointer`; all-errors not checked [17] | Not checked | Not checked | Active [17] | MIT |
| Zod 4.6.5 (`fromJSONSchema`) | Not a validator; experimental subset [18] | Local refs only [18] | Kept as registry metadata [18] | All issues; path arrays [18] | `z.compile` in-process only [18] | Native, no `u` flag [18] | 25 stable releases in 12 months [18] | MIT |
| Effect Schema 4.0.3 | Subset [19] | `#/$defs/<key>` only [19] | Ignored [19] | `errors: 'all'`; path arrays [19] | Not checked | Patterns rejected unless `'apply'` [19] | 4.0.0 stable 2026-10-01 [19] | MIT |

## 4. Candidates

**Ajv.** Gaps: no `$anchor` (#1854, open since 2021) [2]; `$dynamicRef` partly wrong; wrong answers on `__proto__`, `toString` and `constructor` property names [1]; open `unevaluatedProperties` bugs involving `if`/`then` and `oneOf` [3]. Strict mode throws on unknown keywords, so each `x-ligand-*` keyword is registered per instance, as a `code` or `macro` keyword for standalone output [5][6]. With no documented annotation API (unverified), Ligand reads `x-ligand-personal` from the schema itself. Security fixes: CVE-2025-69873 (ReDoS via `$data`, fixed 8.18.0) and a prototype-pollution fix in GitHub release 8.19.0, which is not on npm [8].

**@hyperjump/json-schema.** Highest compliance among established libraries, spec output formats, and unknown keywords readable as annotations [10][11]. Against it: those features are experimental, there is no regex hook, and no throughput figures exist (an interpreter, likely slower; unverified) [10]. Best as Ligand's reference validator and fallback.

**TypeBox.** Validates plain JSON Schema, and resolved a Ligand-style `../../types/listing.yaml` reference locally [12]. Compliance is self-reported, and module emit sits in a module being phased out [13]. It wins if Bowtie confirms full compliance and `typebox/schema` gains stable emit.

**ata-validator, json-schema-library.** Both pass every Bowtie test [1], but ata-validator is about 7 months old [16] and json-schema-library puts flexibility before performance [17]. Re-evaluate in a year.

**schemasafe, cfworker, jsonschema.** Ruled out: no release in 3 years and 20 months respectively [14][15]; jsonschema 1.5.0 stops at draft-07 [22].

**Zod, Effect Schema, ArkType, Valibot.** None validates JSON Schema; each translates a subset. Zod throws on external `$ref`, `not`, `unevaluated*`, `if`/`then`/`else` and `dependent*`, and silently drops some assertions (#6634) [18]. Effect Schema 4.0.3 accepts only `#/$defs/<key>` refs and rejects `$dynamicRef`, `contains`, `dependent*`, conditionals and `unevaluated*` [19]. `@ark/json-schema` 0.0.12 rejects even local `$ref` [20]; Valibot has no importer [21]. For any of them to win, ADR 0006 would have to be reversed to TypeScript-first contracts (rejected because the canvas would execute user code), or Ligand would bundle each contract into one document and ban much of Draft 2020-12. Effect Schema is the one to watch: Ligand already pins Effect (ADR 0010), so a full multi-file import would remove a dependency.

## 5. Risks and containment

- **ReDoS in `pattern`.** Use `code.regExp` with RE2 and lint out lookahead and backreferences, which RE2 lacks; `ajv-formats` regexes may themselves be unsafe [7]. Unverified: RE2 in standalone output, and RE2's licence and binaries. Validation on the main thread escapes ADR 0011's time limit.
- **Correctness gaps.** Lint schemas to forbid `$anchor`, `$dynamicRef`, `$dynamicAnchor` and `$data`; prefer `additionalProperties: false` where nothing is composed. Run every example through Ajv and hyperjump in Ligand's tests and fail on disagreement.
- **`__proto__`-style keys in payloads.** Add fixtures; test whether `ownProperties: true` fixes Ajv's answers (unverified) [4].
- **Worker compile cost.** Generate standalone ESM validators in `ligand dev` and at build [6]. No benchmark of hundreds of linked schemas exists; measure one.
- **Maintenance.** Pin `ajv` exactly (>= 8.20.0) and keep validation behind one seam so hyperjump can replace it [8][9].

## 6. Open questions and unverified claims

- TypeBox's Draft 2020-12 compliance (self-reported) [12].
- Hyperjump's speed, all-errors behaviour, and whether `restoreValidator` needs the schemas registered [10].
- Whether `ajv` 8.20.0 contains the 8.19.0 prototype-pollution fix (plausible, not verified) [8].
- Whether port validation runs on the main thread or in workers (no ADR says).
- All throughput figures found are vendor claims or unreproduced.
- ata-validator's ReDoS behaviour and stability [16].

## References

All sources checked 2026-10-11 unless dated otherwise.

1. Bowtie Draft 2020-12 report, https://bowtie.report/draft2020-12.json (bowtie 2026.7.4, tallied locally).
2. Ajv issue #1854, https://github.com/ajv-validator/ajv/issues/1854 (open, updated 2021-12-19).
3. Ajv issues #2675, #2579, #2559, #2483, https://github.com/ajv-validator/ajv/issues.
4. Ajv API and options, https://ajv.js.org/api.html, https://ajv.js.org/options.html (ajv 8.20.0).
5. Ajv user keywords, https://ajv.js.org/guide/user-keywords.html.
6. Ajv standalone, https://github.com/ajv-validator/ajv/blob/master/docs/standalone.md (8.20.0).
7. Ajv security, https://github.com/ajv-validator/ajv/blob/master/docs/security.md (8.20.0).
8. npm registry `ajv`, `ajv-formats`; https://github.com/ajv-validator/ajv/releases; GHSA-2g4f-4pwh-qvx6, GHSA-v88g-cgmw-v5xw.
9. Ajv commits since 2025-10-11, https://github.com/ajv-validator/ajv/commits.
10. @hyperjump/json-schema README and source, https://github.com/hyperjump-io/json-schema (1.18.0).
11. npm registry and commits, @hyperjump/json-schema.
12. TypeBox schema docs, https://github.com/sinclairzx81/typebox/tree/main/design/website/docs/schema, and a local run of typebox 1.3.36.
13. TypeBox compile docs, https://github.com/sinclairzx81/typebox/tree/main/design/website/docs/compile (1.3.36).
14. schemasafe docs, https://github.com/ExodusMovement/schemasafe/tree/master/doc (1.3.0); npm time.
15. cfworker, https://github.com/cfworker/cfworker/tree/main/packages/json-schema (4.1.1).
16. ata-validator README, https://github.com/ata-core/ata-validator (npm 1.53.0).
17. json-schema-library README, https://github.com/sagold/json-schema-library (npm 11.6.6).
18. Zod, https://zod.dev/json-schema; zod 4.6.5 `src/v4/classic/from-json-schema.ts`; https://github.com/colinhacks/zod/issues/6634; https://github.com/colinhacks/zod/blob/main/packages/docs/content/compile.mdx.
19. effect 4.0.3 `src/SchemaRepresentation.ts`, `src/SchemaAST.ts`, https://www.npmjs.com/package/effect.
20. @ark/json-schema, https://github.com/arktypeio/arktype/tree/main/ark/json-schema (0.0.12, local run 2026-10-11).
21. Valibot packages, https://github.com/open-circle/valibot/tree/main/packages (1.5.0).
22. jsonschema README, https://github.com/tdegrunt/jsonschema (1.5.0).
23. Ligand ADRs 0006, 0007, 0010, 0011, `docs/adr/`.
