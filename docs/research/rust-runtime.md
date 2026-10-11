# Rust as Ligand's runtime

Research date: 2026-10-12.

## 1. Question and constraints

Would Rust be a better (more performant) runtime for Ligand than TypeScript on Effect 4? "Runtime" means `@ligand/runtime`: habitats, the workflow engine, port validation, the logic-atom worker pool, the run log and the store [1]. Constraints that hold whatever the language:

- atoms are TypeScript in v1, written by AI and held to tests that run on Node (Vitest, StrykerJS), so any host still runs them in a JavaScript engine [1];
- every atom execution is a durable activity (ADR 0012), every port value is validated (ADR 0013), and a runaway logic atom is killed at its time limit (ADR 0011);
- the runtime is MPL-2.0 and a production install needs no EUPL code (ADR 0007); one process runs per ecosystem;
- the first app polls a site every 10 minutes (I/O bound); HTTP organisms are capped at 64 concurrent runs per habitat (ADR 0014), and no throughput target exists yet.

Six architectures were compared: **(a)** a Rust host that embeds V8 to run atoms; **(b)** Node as host with Rust native addons on hot paths; **(c)** a Rust engine process that sends each atom to Node workers over IPC (the iii model); **(d)** adopting an existing Rust-cored durable engine; **(e)** the current TypeScript runtime on Deno (Rust around V8) or Bun; **(f)** atoms in Rust or WASM, which is outside v1.

## 2. Recommendation

**Not for v1. Keep TypeScript on Effect 4 and spend the effort on the design levers in section 7, which change per-run cost far more than the language does. Revisit a Rust core only against a stated performance target that the TypeScript runtime still misses after those levers.** Confidence: high for the listing bot, medium for HTTP organisms, because the full benchmark run was cancelled and section 5 rests on smoke runs.

Four facts:

1. **Durability sets the floor, and it costs the same in Rust.** Effect 4.0.3's durable engine takes 3 SQLite write transactions per activity at concurrency 1 (12 statements, counted) and about 2 at concurrency 64, where one claim covers all pending messages [2][3]. `node:sqlite` on Node 24.18.1 defaults to `synchronous=FULL`, and Effect's client never changes it, so every commit waits for an fsync [4][5]. On this machine one FULL commit of a 1 KB row took about 9 ms from Node 24 and from Rust (rusqlite 0.40.2) alike, and 15 µs from both at NORMAL [L1]. Rust can't make an fsync faster.
2. **Rust does shrink host-side CPU, but the largest share is Effect's cluster machinery, and TypeScript can drop that too.** In smoke runs, one durable activity step cost about 1 ms on Effect's cluster engine at NORMAL. A 95-line custom Effect engine that writes one row per activity took 0.14 ms, and plain TypeScript 0.055 ms [L2]. With no durability, a Rust server ran the 10-atom organism on about 0.15 ms of CPU per request, including validating every port. The TypeScript floor needed about 0.55 ms [L3][L4]. So Rust is about 3.5 times leaner on the host side, but a thin TypeScript engine closes most of the gap that matters.
3. **The atoms stay in V8, and every crossing out of V8 costs time.** Node and rusty_v8 run the same V8 JIT [6]. Prisma removed its Rust query engine because serialising across the language boundary cancelled the speed-up [7]. Oxc says the same about JSON transfer between Rust and JS [8]. Here, jsonschema-rs validated 2.5 to 5 times faster than Ajv in process, but only about as fast as Ajv once the value had to cross napi as a JSON string [L5].
4. **Nothing permissive exists to adopt, and a lot would have to be built and shipped.** Restate and Golem are BSL 1.1, iii's engine is ELv2 and Obelisk is AGPL-3.0. duroxide is MIT but still a preview [9][10]. Ligand would replace about 36,000 lines of Effect (workflow, cluster, persistence, rpc) and ship native binaries for every platform, where today it ships no native npm code at all [11].

## 3. Where a run's time goes

| Operation per run | Bound by | What a Rust runtime changes |
|---|---|---|
| Engine commits: 3 per activity (2 under load), every atom an activity | fsync at FULL; WAL write I/O at NORMAL | Nothing: same C SQLite, same kernel and disk [L1] |
| Effect cluster bookkeeping per activity: entity mailbox, sharding, Schema codecs, 12 SQL statements | Main-thread CPU | Removable, but a thin TypeScript engine removes most of it too [L2] |
| One worker hop per logic atom (structured clone both ways) | CPU on both threads | (a) replaces it with an in-process isolate call: about 4 µs against 20 to 25 µs for 1 KB [L6][L7] |
| Ajv validation of every port, inside the worker | CPU in V8 | Stays in V8 unless the value already lives on the Rust side |
| re2js pattern matching | CPU in V8 | The Rust regex crate scans a 200 KB page one to two orders of magnitude faster, but only behind a crossing [L5] |
| HTTP intake and response | Main-thread CPU | axum is faster per request, but not the limit at 64 concurrent runs |
| Run-log line per atom | Page cache (not flushed) | Nothing |

**Listing bot.** About 30 commits per run, at 6 to 9 ms each here, adds 0.2 to 0.3 s to a run that waits on the network anyway, 144 times a day. A language change saves nothing measurable [12].

**HTTP organism, design as written.** With every atom durable and FULL commits, throughput is about 1 / (commits per run × fsync time). The 10-atom organism reached about 3.4 runs/s at 64 connections, with a p50 of about 8 s [L4]. Rust would land in the same place. At NORMAL the limit moves to the CPU on Effect's main thread: about 12 to 14 ms of CPU per run, about 60 runs/s [L4].

## 4. Architectures compared

| Option | What it changes | Gain where Ligand spends time | Cost and risk | Verdict |
|---|---|---|---|---|
| (a) Rust host, embedded V8 (deno_core) | Worker hop becomes an isolate call. A runtime starts from a snapshot in about 1.5 ms and adds about 1.7 MiB, against about 12 MiB for an idle Node worker. The isolate is reusable after termination [L7] | Host CPU and memory. The atoms still run on V8 | No Node built-ins, so I/O atoms that use npm drivers can't run in a bare isolate. Ligand would need its own module loader and type stripping. rusty_v8 releases a new major every 4 weeks [6][13]. Rewrites ADRs 0010 to 0013 and 0015 | No |
| (b) Node host with napi-rs addons | Native validation or regex | Small: Ajv-level speed once the boundary is paid [L5][14] | Per-platform binaries (the napi-rs template targets 14) and no Deno or Bun portability [15] | Only for a hot path that has been measured, such as regex on large strings |
| (c) Rust engine, Node atoms over IPC | Engine CPU moves to Rust | With no durability, a Rust engine with Node atoms over Unix sockets ran about 7,000 to 8,700 requests/s, against about 3,800 to 4,000 for the TypeScript floor [L3][L4] | A second process and an IPC call per atom (iii's own echo costs about 131 µs) [16]. Two runtimes to ship, and an engine to own | Not in v1. This is the variant to prototype if a target is missed |
| (d) Adopt a Rust-cored engine | Engine replaced | None: step cost is still set by commit policy. Restate is about 2 ms per step, Golem about 2 ms per forced commit, and Obelisk 5.7 ms per activity at FULL [9][17] | BSL, ELv2 or AGPL licences, a second process, and memoisation by journal position rather than by path [9] | No |
| (e) Deno or Bun | The host | None for durability, Effect, Ajv or workers. Raw HTTP is faster by vendor and secondary figures [18] | Deno 2.9.6: `worker.terminate()` doesn't stop a runaway worker (verified here), which breaks ADR 0011 [L6]. Its `node:sqlite` also lacks `enableLoadExtension`, which Effect's client calls [L2]. Bun: terminate lags by tens to hundreds of ms, and there's no LTS line [19] | No. Node 26 LTS (2026-10-28) is the free engine upgrade [20] |
| (f) Atoms in Rust or WASM | Removes V8 from the hot path | The only option that removes the atoms' JS cost | A new target language for the AI and for tests. Out of v1 scope | Future work. Keep the atom ABI engine-neutral: JSON in, JSON out, named ports |

The precedents agree. Rust pays off for coarse, CPU-bound batch work with one crossing in and one out: TypeScript 7's Go port is 8 to 12 times faster on full builds, and Rolldown and Turbopack speed up whole bundles [21][22]. It loses on chatty calls that pass structured data back to JS. Ten early-returning JS plugins slowed a Rolldown build 3.7 times. SWC called through N-API was 3 times slower than pure-JS Babel on a large file [22][8]. Ligand's runtime crosses at every atom, which is the chatty side. Motia's 500 ms per request came from spawning a process per step, not from Node [23].

## 5. Measurements on this machine (indicative)

Harnesses for all seven questions were built and smoke-tested, but the full run on a quiet machine was cancelled and the harnesses were then deleted, so a rerun means rebuilding them from the descriptions under L1 to L7. The figures below come from smoke and quick runs during the build, while other harnesses were compiling (load average 3 to 9), with windows of a few seconds. Use them for orders of magnitude and for comparisons measured side by side, not as figures to quote. The machine: Ryzen 9 3950X, Samsung 980 PRO on btrfs (`compress=zstd:1`, 94% full), Node 24.18.1, Node 26.11.1, Deno 2.9.6, Rust 1.99.0 in Docker.

| Question | TypeScript / Node | Rust | Note |
|---|---|---|---|
| One WAL commit, 1 KB row, FULL [L1] | 8.9 ms p50 (Node 24) | 9.2 ms p50 (rusqlite) | Raw 4 KB write plus fsync: 6.1 to 6.3 ms when the machine was quiet |
| Same at NORMAL [L1] | 15 µs | 15 µs | Rows per commit 8: same commit time, 8 times the rows |
| Durable activity step, c=1, 1 KB, NORMAL [L2] | Cluster engine 0.98 ms; custom Effect engine 0.14 ms; plain TS 0.055 ms; in-memory engine 0.035 ms | Not built | Cluster machinery with in-memory storage alone: 0.32 ms |
| HTTP organism, 10 atoms, c=64, requests/s [L3][L4] | Every atom durable, FULL: about 3.4. NORMAL: about 60. Durable per run: about 250. Plus one hop per molecule: about 380. In-memory engine: about 2,200. No engine: about 3,900 | 1 commit per activity, NORMAL: about 400 (btrfs-bound; about 3,400 on tmpfs). With group commit: 740 to 960. In memory: 38,000 to 47,000 | Rust spreads orchestration over 8 cores; Node has one main thread and 4 workers |
| CPU per request, in-memory floor [L3][L4] | About 0.55 ms | About 0.15 ms | Both validate every port |
| Worker hop, 1 KB echo [L6][L7] | 20 to 25 µs (Node 24 and 26), about 45 µs (Deno) | Isolate call: about 4 µs (JSON text), about 18 µs (serde_v8) | 100 KB object through a worker: about 1.3 ms |
| 10-atom chain [L6][L7] | 10 hops: 360 to 390 µs; 1 hop: about 45 µs; direct: about 13 µs | 10 crossings: 200 to 350 µs; 1 crossing: about 23 µs | |
| Validate 20 listings, pre-parsed [L5] | Ajv native regex: about 27 µs; Ajv with re2js: about 58 µs | jsonschema 0.58.6: about 11 µs | Through napi with `JSON.stringify`: about the same as Ajv with re2js |
| Regex scan of a 200 KB page [L5] | V8: 0.4 to 5.7 GB/s; re2js: 10 to 400 MB/s | regex crate: 0.65 to 9.9 GB/s | V8's linear engine rejects 2 of the 5 patterns and runs at 14 to 21 MB/s |
| Stopping runaway sync code [L6][L7] | `terminate()` stops it in about 30 µs. Respawn: about 21 ms (Node 24), about 10 ms (Node 26). TS atom import: 26 to 30 ms cold, 0.4 ms with the compile cache | `terminate_execution`: tens of µs, isolate reusable | Deno 2.9.6: the thread never stops |
| Memory per atom sandbox [L6][L7] | Idle worker: about 12 MiB (21 to 25 loaded) | deno_core runtime from a snapshot: about 1.7 MiB each | |

How to read this: the two largest factors for HTTP are TypeScript design choices. Making every atom durable costs about 4 times. Effect's SQLite-backed cluster engine costs about 6 times against its in-memory engine. Rust's lead only shows once durability is out of the way: about 3.5 times less CPU per request, and about 10 times the in-memory throughput, because tokio uses every core. Whether that lead matters depends on a target Ligand hasn't set.

## 6. What a Rust runtime would cost to build and ship

- **Engine.** No permissive Rust equivalent of effect/workflow exists, so Ligand would write one. That replaces Effect's workflow (4.1k lines), cluster (18.3k), persistence (6.4k) and rpc (7.1k) modules. iii's engine, at 112k lines of Rust, is an upper reference for the class [11][24].
- **Crates.** Every other piece has a maintained, permissively licensed crate: tokio 1.53.2, axum 0.8.9, reqwest 0.13.5, rusqlite 0.40.2, jsonschema 0.58.6, croner 4.0.1 [25]. The churn moves rather than disappears. rmcp went through three majors in five months. OpenTelemetry Rust traces are still Beta. serde_yaml is unmaintained, and jsonschema-rs published 67 releases in 11 months [25][26].
- **Distribution.** One npm package per platform, selected through `optionalDependencies`: esbuild has 26, oxc-parser 19, the napi-rs template 14 and Biome 8. Add a CI matrix, a glibc floor and documented failure modes (`--omit=optional`, `node_modules` copied between platforms) [15][27].
- **Licences.** Permissive crates are fine inside an MPL-2.0 Larger Work (section 3.3) [28]. Real lockfiles also pull in MPL-2.0, CDLA-Permissive-2.0, ISC and OpenSSL-style code, so Ligand would need a cargo-deny licence gate of its own [24][29].
- **ADRs.** Options (a), (c) and (d) rewrite 0010 to 0013, and (a) also 0015. ADR 0007 gains an audit duty. Option (b) touches the fewest. Dry runs and recordings would have to live in the Rust runtime or be duplicated in the dev server, which puts golden runs at risk (ADR 0016) [1].

## 7. Levers that matter more than the language

1. **Set `synchronous=NORMAL` on `engine.db`.** In WAL mode NORMAL can't corrupt the database, and it loses only recent commits on power loss [5]. The engine DB is disposable (ADR 0012). ADR 0005 must then state when run-log lines are flushed, so rebuilding from the log still holds. This lever alone took about 3.4 to about 60 requests/s here [L4].
2. **Make durability coarser.** Use one activity per pure molecule or per I/O atom: the run log already stores every output (ADR 0005). That gave about 4 times the throughput [L4].
3. **Replace the cluster engine with a thin engine.** Effect's `WorkflowEngine` is an open service with about 10 methods. A one-row-per-activity version took 95 lines in the harness and cost about 7 times less per activity step [2][L2]. The price is owning code against an unstable interface (ADR 0010).
4. **Move SQLite off the event loop.** A writer thread with group commit reached about 8,000 writes/s against about 155, and kept the main thread's p99 loop delay at about 1.8 ms. Per-write FULL commits on the main thread pushed it to about 805 ms [L1].
5. **Use one worker hop per pure molecule.** That's about 45 µs against 360 to 390 µs for 10 hops [L6].
6. **Enable the compile cache inside workers.** It cuts the TS atom import after a kill from 26 to 30 ms to about 0.4 ms [L6].
7. **Move to Node 26 LTS on 2026-10-28.** It brings V8 14.6 and halves worker spawn time here [20][L6].

## 8. Corrections to the ADRs found along the way

- **ADR 0011** says JavaScript can't interrupt synchronous code on its own thread. Node's `vm` `timeout` option does exactly that, through a watchdog thread that calls V8's TerminateExecution [30]. Checked live on 24.18.1: a `for(;;)` stopped at 101 ms on a 100 ms limit, and the context stayed usable. Workers are still the better choice, for other reasons: heap limits, state isolation, multiple cores, and no stalled main loop. `vm` stalls the loop for the whole limit and adds 40 to 75 µs per call [L6].
- **ADR 0012** says no supported purge API exists. `MessageStorage.clearAddress` is public in effect 4.0.3. It deletes a completed execution's rows in one transaction, after which only Ligand's own key index prevents a rerun [3][L2].
- **ADR 0012 spike items.** The single-process stack (SingleRunner, SQL message storage, `node:sqlite`) resumes after `kill -9`. No completed activity re-executed; only the one in flight ran twice. Polling isn't on the happy path, and a 5 s reply poll and a 60 s message poll left latency unchanged [L2].
- **ADR 0015 and option (e).** Deno 2.9.6 can't host ADR 0011 as written: `worker.terminate()` resolves, but the thread keeps running. The harness reproduced this with both `node:worker_threads` and Web Workers [L6]. In a direct check with `node:worker_threads`, the worker's counter moved 48 million times in the 500 ms after `terminate()`. Node 24 stops it.

## 9. What would change the answer

- **A missed target.** If HTTP organisms get a stated target (p99 and requests/s within the 64-run cap) that the TypeScript runtime misses after section 7, prototype (c): the engine and its writes in Rust, atoms in Node workers, with coarse crossings.
- **More atom languages.** The YAML and JSON Schema files are already language-neutral. A shared core then becomes a reuse argument, as it was for Temporal's sdk-core and, originally, for Prisma [7][31].
- **Atoms in Rust or WASM (option f).** It's the only option that removes V8 from the hot path.
- **Memory-bound deployments.** On small VPS or serverless hosts, memory per sandbox can dominate: about 12 MiB per Node worker against about 1.7 MiB per snapshot isolate [L6][L7].

## 10. Open questions and unverified claims

- Every timing in section 5 comes from short windows on a busy machine. Still unmeasured: p99 and GC pauses under load, RSS over a soak, the full Deno end-to-end run, and the per-layer CPU profile of Effect's engine.
- btrfs on a consumer NVMe is a slow-fsync case. ext4 or XFS on a server NVMe with power-loss protection shrinks FULL's penalty, but doesn't change the ratio between languages.
- Effect publishes no workflow or cluster throughput. Its 6.4 times speed-up of v4 over v3 is a vendor claim with no hardware or workload stated [32].
- TechEmpower was sunset after Round 23, and no current independent Node against Rust HTTP figures were retrieved [33].
- Only Node is integration-tested for Effect's SQL durable path. Effect's CI runs on Deno and Bun, but not this stack [34].
- Deno's worker code calls `terminate_execution`, yet the thread kept running. The cause wasn't investigated [35].

## References

All sources checked 2026-10-11 or 2026-10-12.

1. Ligand `docs/design.md` and ADRs 0005, 0007, 0010 to 0016, `docs/adr/`.
2. effect 4.0.3, https://registry.npmjs.org/effect; source https://github.com/Effect-TS/effect at 22e7129, `packages/effect/src/workflow/WorkflowEngine.ts`, `cluster/ClusterWorkflowEngine.ts`, `cluster/SingleRunner.ts`, `cluster/SqlMessageStorage.ts`.
3. effect 4.0.3 `dist/cluster/MessageStorage.d.ts` (`clearAddress`, line 135); `cluster/SqlMessageStorage.ts`.
4. @effect/sql-sqlite-node 4.0.3 `src/SqliteClient.ts` (one serialised connection, WAL, `busy_timeout`, no `synchronous`).
5. Node v24.18.1 `deps/sqlite/sqlite.gyp` and `sqlite3.c`, https://github.com/nodejs/node/tree/v24.18.1/deps/sqlite; SQLite, https://sqlite.org/pragma.html#pragma_synchronous and https://sqlite.org/wal.html.
6. rusty_v8 README at 2f58804, https://github.com/denoland/rusty_v8; deno_core 0.412.0, https://crates.io/crates/deno_core.
7. Prisma, https://www.prisma.io/blog/from-rust-to-typescript-a-new-chapter-for-prisma-orm (2025-01-30), https://www.prisma.io/blog/rust-to-typescript-update-boosting-prisma-orm-performance (2025-03-03), https://www.prisma.io/blog/prisma-orm-without-rust-latest-performance-benchmarks (2025-07-24).
8. Oxc, https://oxc.rs/blog/2025-10-09-oxlint-js-plugins.html; https://github.com/oxc-project/bench-javascript-parser at 1aeeb3c.
9. Restate LICENSE at 752283e and v1.7.13, https://github.com/restatedev/restate; https://restate.dev/blog/building-a-modern-durable-execution-engine-from-first-principles; iii engine/LICENSE at 5c000f8, https://github.com/iii-hq/iii; Obelisk at 24d7595, https://github.com/obeli-sk/obelisk; Golem LICENSE at 2a9fc58, https://github.com/golemcloud/golem.
10. duroxide 0.1.30, https://crates.io/crates/duroxide; Temporal v1.32.1, https://github.com/temporalio/temporal.
11. `npm pack effect@4.0.3`, physical line counts per module.
12. Derived from [1], [2] and [L1]; not measured end to end.
13. deno_core `libs/core` README at a18ce33, https://github.com/denoland/deno; Deno Node API support, https://docs.deno.com/runtime/reference/node_apis/.
14. jsonschema-rs issue #239 (conversion up to 85% of validation time), https://github.com/Stranger6667/jsonschema/issues/239; napi-rs `crates/napi/src/js_values/de.rs` at 806d9e8, https://github.com/napi-rs/napi-rs.
15. napi-rs package template at 76f4c58, https://github.com/napi-rs/package-template.
16. iii bench data, https://iii-hq.github.io/iii/dev/bench/data.js (`ws_roundtrip/invoke_echo`, commit 4d115a81).
17. Obelisk, https://obeli.sk/blog/challenging-fastest-workflow-engine/; Golem benchmark results, https://golemcloud.github.io/benchmark-results/.
18. Deno 2.9, https://deno.com/blog/v2.9; Bun 1.4, https://bun.com/blog/bun-v1.4 (vendor figures).
19. Bun issues #44216 and #44886, https://github.com/oven-sh/bun/issues.
20. Node release schedule, https://raw.githubusercontent.com/nodejs/Release/main/schedule.json; https://nodejs.org/dist/index.json.
21. TypeScript 7.0, https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/.
22. Rolldown, https://rolldown.rs/in-depth/why-plugin-hook-filter; Vite 8, https://vite.dev/blog/announcing-vite8.
23. `docs/research/motia.md`; https://github.com/iii-hq/iii/issues/1132.
24. iii at 5c000f8 (`engine/src` line count, `Cargo.lock`), https://github.com/iii-hq/iii.
25. crates.io API, https://crates.io/api/v1/crates/<name>, for each crate named.
26. rmcp, https://github.com/modelcontextprotocol/rust-sdk at 808124b; opentelemetry-rust README at 8d81bb7; https://github.com/dtolnay/serde-yaml.
27. npm registry for esbuild 0.28.2, oxc-parser 0.153.0, @biomejs/biome 2.5.15; esbuild `lib/npm/node-platform.ts` at f6058f8.
28. MPL-2.0, https://www.mozilla.org/en-US/MPL/2.0/.
29. cargo-deny 0.20.2, https://github.com/EmbarkStudios/cargo-deny.
30. Node `src/node_watchdog.cc` and `doc/api/vm.md`, https://github.com/nodejs/node/tree/v24.x.
31. Temporal, https://temporal.io/blog/why-rust-powers-core-sdk.
32. Effect 4.0 release, https://effect.website/blog/releases/effect/40.
33. TechEmpower sunset notice, https://github.com/TechEmpower/FrameworkBenchmarks/issues/10932.
34. Effect CI, `.github/workflows/check.yml` at 22e7129, https://github.com/Effect-TS/effect.
35. Deno `runtime/web_worker.rs` at a18ce33, https://github.com/denoland/deno.

Local measurements (from smoke and quick runs on 2026-10-11 and 2026-10-12, harnesses since deleted; see section 5):

- L1. `h1-sqlite-commit`: commit latency for Node 24, Node 26 and rusqlite, FULL, NORMAL and OFF; writer-thread variant.
- L2. `h2-effect-engine`: Effect 4.0.3 engine matrix, transaction counts, crash and resume, `clearAddress`, Deno check.
- L3. `h6-rust-lower-bound`: axum, rusqlite and jsonschema server for the same organism; variant (c) with Node atoms over Unix sockets.
- L4. `h5-e2e-http`: the organism on Effect HttpServer, durable engine, worker pool and Ajv with re2js, with seven configurations.
- L5. `h4-validation`: Ajv with and without re2js, jsonschema and boon, regex engines, napi-rs crossing.
- L6. `h3-worker-hop`: worker hop, `vm` timeout, kill and recovery, and memory, on Node 24, Node 26, Deno and Bun.
- L7. `h8-embedded-v8`: deno_core call cost, snapshots, `terminate_execution`, memory; rquickjs comparison.
