# Motia and what happened to it

Research date: 2026-10-11.

## 1. Question and short answer

How close was Motia (motia.dev) to Ligand, and why did it wind down: a generic pivot to AI, or a product that did not work?

**It is a false dichotomy, and Motia did not shut down.** The same founder (Mike Piccolo), the same company (Motia LLC) and the same GitHub repository replaced the Motia framework with iii, a lower-level runtime of Functions, Triggers and Workers on a Rust engine [1][3][14]. The only stated reason is architectural: the founder wrote that "a framework can't solve this", meaning the integration sprawl Motia set out to remove [1]. The evidence best supports a third reading, which is my inference: **Motia's Node runtime, which spawned a process per step, hit a performance ceiling the team had acknowledged since mid-2025, and the rewrite became a repositioning one layer down**, with agents as the headline use case of the successor [9][10][14].

- "Generic AI pivot": weak. Motia was marketed with AI from early 2025, and the plan for a Rust core (July 2025) mentions no AI [9][17]. iii's marketing is agent-heavy, so AI shaped the successor's pitch, not the decision as stated.
- "Product did not work": partly. The runtime did not scale, and no release past the 1.0 candidates was ever current [7][10]. Attention grew steadily to about 15k stars, but usage stayed modest and fell after December 2025 [7][8].
- Business reasons: unknown. No funding, revenue or customer data exists in any source found, and the hosted product was never priced [18][21].

Confidence: high that this was a rename and re-platform, not a shutdown; medium that the runtime ceiling drove it; low on any commercial motive.

## 2. Timeline

- **2024-12-15**: first commit, by Piccolo. GitHub org MotiaDev created 2024-12-28; repo (id 911303109) 2025-01-02; npm `motia` 2025-01-22 [3][7].
- **2025-02 to 2025-06**: site titles move from a framework for intelligent workflows (2025-02-03) to "AI Agent Framework for Software Engineering Teams" (2025-04-02) to a unified backend framework for APIs, events and AI agents (2025-06-15), with a Motia Cloud beta sign-up [17].
- **2025-07-23**: issue 482, by a core contributor, proposes rewriting the core in Go or Rust; 14 reactions [9].
- **2025-11-27**: licence changes from MIT to Elastic License 2.0 (ELv2) [6].
- **2025-12**: npm downloads of `motia` peak at 24,007 a month, in a month with 122 published versions. The hosting page lists Motia Cloud's price as "Coming Soon" (2025-12-05) [7][18].
- **2026-01-12**: a core contributor concedes on issue 1132 that process management is slow at scale and says a Rust binary will replace it [10].
- **2026-01-30 to 2026-02-25**: iii-hq org created; `iii-sdk` on npm (02-10); `motia` 1.0.0-rc builds depend on `iii-sdk`, and the README calls 1.0-RC "iii-powered" (02-24) [4][7]. A stable 1.0.0 (02-11) is deprecated as "Published by mistake" [7].
- **2026-02-19**: licence changes to Apache-2.0 [6].
- **2026-03-03 to 03-09**: the iii engine merges into the Motia repo (#1220, 03-03). The repo is renamed to iii-hq/iii between 03-03 and 03-07 (Wayback shows 200, then 301). First iii release v0.8.0 (03-06). Dual licence (03-09): engine ELv2, SDKs, CLI, console, frameworks and docs Apache-2.0 [3][4][6].
- **2026-03-12**: Heavybit calls Piccolo the founder of iii, "formerly known as Motia" [19].
- **2026-03-15**: last `motia` publish, 1.0.4-rc.1, still the `latest` tag, with no npm deprecation notice [7].
- **2026-04-16**: a blog post signed by Piccolo as founder and CEO of Motia and iii deprecates Motia in favour of iii [1]. A site banner appears between 04-14 and 04-22; deprecation docs land 04-15 and 04-17 [2][4].
- **2026-04-20 and 04-22**: 68 issues closed with a "Motia becomes iii now" note; PR 1512 deletes the framework (575 files, about 48K lines) and its CI and release pipelines [5][12].
- **2026-05-11 to 06-07**: the full site with banner gives way to one page: "Motia is winding down. iii.dev is what's next." It is unchanged today [2].
- **2026-10-11**: iii is at v0.24.6-rc.3 (2026-10-09), still pre-1.0, with 18,835 stars and daily pushes [3].

## 3. What Motia was, compared with Ligand

Motia's one primitive was the **Step**: a file exporting a config (name, type `api`, `event` or `cron`, the topics it emits and subscribes to, an optional Zod or JSON Schema for the body or input) and a handler. Steps in TypeScript, JavaScript and Python (Ruby in beta) shared key-value state and streams, and were connected by string topic names. The **Workbench** was a local UI that drew the graph from the code, with an endpoint tester, logs, traces and a state editor [15].

| Concept | Motia (0.17, January 2026) | Ligand | Same or different in kind |
|---|---|---|---|
| Unit of work | Step: one handler mixing trigger, effect and logic [15] | Atom: logic or I/O, contract in YAML, only atoms hold code | Different |
| Connection | `emits`/`subscribes` string topics; edges derived by matching names [15] | Typed wires between ports, compiler-checked | Different |
| Grouping | `flows: [...]` labels for display and log context, no execution semantics [15] | Molecules and organisms are executable graph files | Different |
| Entry points | `api`, `event`, `cron` step types [15] | HTTP, schedule and event habitats | Same |
| Types | Zod or JSON Schema on HTTP body and event input [15] | JSON Schema 2020-12 on every port, validated in production | Same family, narrower in Motia |
| Visual tool | Workbench views the derived graph; `noop` steps with virtual edges draw what code does not declare [15] | Canvas reads and writes graph files; test status, last-run values, AI tasks, proposals | Different |
| State | Key-value by group and key [15] | Store: versioned, with `conflict` and `stale` outcomes | Same idea, stricter in Ligand |
| Failures | Queue retries per step; Cloud retries event steps 3 times [15] | Only I/O atoms retry, for explicit reasons; runs park | Different |
| Tests | Optional integration tester that posts and watches topics [15] | Examples, fixtures, properties, golden runs, mutation gate | Different |
| AI | Instruction files (`AGENTS.md`, Cursor rules); agents as app content [15] | Five AI tasks held to tests, proposals, escalation, `ligand mcp` | Different |
| Languages | TS, JS, Python; a Node core spawning processes [9][15] | TypeScript in v1; language-neutral files | Different |
| Deployment | Self-host, or Motia Cloud beta on AWS [15][18] | Self-hosted, one process per ecosystem, built `dist/` | Different |

Motia was the closest match in shape: small units, event-driven, the same three entry kinds, a visual local UI, JSON Schema typing, an AI pitch, TypeScript first. It lacked all three things in Ligand's prior-art test: the graph as source, AI held to tests, typed wiring. Ligand's glossary already avoids Motia's words for its own concepts (step, flow, Workbench) [24]. iii is further away still: it describes itself as "the engine underneath" agent frameworks and workflow engines, with no graph editor or test gate [14].

## 4. Why it wound down

**Stated reasons.** The post gives one: Motia tackled integration sprawl at the framework level, but a framework is one more tool to integrate, so the answer had to be primitives [1]. It says Motia compiles down to iii, that iii's Rust engine goes far beyond the original in performance, and lists agents as first-class among iii's benefits. It says nothing about funding, adoption, revenue, team or failure, and gives no end-of-support date [1]. iii's migration guide says Motia "was a higher-level framework built on top of iii-sdk" [13].

**Traction.**

| Measure | Value | Source |
|---|---|---|
| GitHub stars | 3 (2025-02-28), 7,716 (2025-09-07), 15,147 (2026-03-03); about 4 to 8 a day March to May 2026; 18,835 (2026-10-11) | [3][8] |
| `motia` npm downloads a month | 983 (2025-04), 8,607 (2025-10), 24,007 (2025-12), 8,388 (2026-01), 4,478 (2026-09) | [7] |
| Last 30 days to 2026-10-10 | `motia` 5,344; `iii-sdk` 150,967 (CI and transitive installs not separable) | [7] |
| Hacker News | Three submissions, 3, 2 and 1 points, almost no comments | [20] |
| Outside contributions | About 188 of 1,907 PRs from non-staff (staff classified by commit volume) | [3] |
| Named production users | None found | [11] |

The December peak is probably inflated by 122 prereleases and automated installs (inference). Stars far outran usage.

**Competing explanations.**

1. *Runtime ceiling.* For: issue 482 calls Motia "pretty much an orchestrator of processes" [9]; issue 1132 measured about 500 ms per request and drew the concession above [10]; users reported slowdown over time, slow startup, an out-of-memory dev server and idle CPU in production, and asked for structured logs and OTLP export before using it in production [11]; 1.0 RC already ran on the Rust engine [4][7]. Against: none found. It explains a rewrite, but not by itself the rename; the post's answer to that is the layer argument.
2. *Generic AI pivot.* For: about 7 of iii's 10 blog posts (2026-04-28 to 08-05) are about agent harnesses and loops, and the homepage demo composes LLM router and provider workers [14]. Against: AI was in every Motia site title checked from 2025-02-03 [17]; the Rust plan predates the agent pitch by nine months and mentions no AI [9]; iii says it is not an agent framework [14]; the post lists agents as one benefit among several [1].
3. *Product did not work (adoption).* For: downloads fell about 65% in January 2026 and never recovered; the 1.0 line never left release candidates; no named production user; thin discussion [7][11][20]. Against: stars, downloads and outside issues rose through December 2025 [3][7][8]; the January drop came as a 1.0 rewrite was announced, which may have paused users (inference, unsupported).
4. *Business or funding.* For: Motia Cloud stayed unpriced through December 2025 [18]; three licence changes in 3.5 months, ending with a source-available engine, look like a search for a model that protects a hosted runtime (inference) [6]; a private `iii cloud` CLI exists [22]. Against: commits rose (128 in March, 147 in April 2026), the LLC continues, and no layoff, funding or shutdown report exists [3][14][21].

**Verdict (inference).** Explanations 1 and 4 are compatible: a runtime that did not scale, plus a company still looking for a hosted product, chose to rebuild at the engine layer and sell that, with agents as the market of the moment. "Did not work" holds for the runtime, not clearly for demand.

## 5. What this means for Ligand

1. **Measure the per-atom overhead in the spike.** Motia's cost was a process hop per step [9][10]. Ligand pays a worker-thread hop, value copying and Ajv validation per logic atom, plus a run-log line and a workflow activity per atom (ADR 0005, 0011, 0012). The listing bot runs every 10 minutes and will never expose this. Add a sixth spike item: an HTTP organism of ten logic atoms under load, measuring added latency per atom and memory over a long run.
2. **Keep the canvas dependent on files only.** Motia's Workbench drew a graph inferred from strings, needed fake `noop` nodes for edges the code did not declare, and was sunset when the runtime changed [15][16]. ADR 0006 already avoids both problems; keep any canvas feature from depending on runtime internals.
3. **Do not promise other languages before the runtime can host them cheaply.** Motia's Python support was a cause of the process-per-step design (Python projects still needed `npx`) [9]. ADR 0006 keeps files language-neutral; a second language also needs an execution model with a cost budget, not just a type generator.
4. **Answer the "one more tool" critique in positioning.** Motia's founder concluded a framework is one more thing to integrate [1]. Ligand is a framework too. Its claim should stay what design.md says, verifiable AI-written logic on a typed graph, with I/O atoms that talk to existing services, not a promise to replace queues, cron and state systems, as Motia's December 2025 claim that one Step could replace more than ten frameworks did [17]. Motia's site title changed three times between February and December 2025 [17].
5. **Count builders, not stars.** About 15k stars sat on top of a few thousand real downloads a month and no named production user [7][8][11]. Done-criterion 3 (two outside developers build something unaided) is the right signal; also track a second real application beyond the listing bot.
6. **Settle licensing and any hosted plan before users arrive.** Motia relicensed three times in 3.5 months, while its site still said MIT [6][18]. ADR 0007 is fixed early, but leaves EUPL/MPL protection against hosted use to legal review; resolve that before release.
7. **Keep file formats stable and migrations tooled.** Motia users migrated 0.17 to 1.0 RC (`emit` became `enqueue`) and then to `iii-sdk` by hand [13][16]. Ligand's files carry `ligand: 1`; a format change should come with an automatic migration.

**Design.md, line 45.** Literally accurate, but it reads as a dead project. Suggested wording: "Motia, once the closest match, was deprecated on 2026-04-16 in favour of iii, a lower-level runtime (functions, triggers and workers on a Rust engine) from the same team; its repository was renamed to iii-hq/iii and the framework removed. iii sits below the layer Ligand occupies." Optionally add iii to the table: overlap, a polyglot runtime with traces in a console; Ligand adds a typed graph as source and AI held to tests.

## 6. Open questions and unverified claims

- Funding, revenue, paying Motia Cloud users and team size: nothing found [21]. Motia Cloud's "usage-based" pricing is unverified.
- Why the licence moved MIT to ELv2 to Apache-2.0 to dual; the PRs give no reason [6].
- The founder's fuller account (X posts around 2026-06-09, a 2026-02-13 podcast) was not read; X and Discord were unreachable.
- Cause of the January 2026 download drop, and of a star burst (about +2,250, 2026-05-11 to 06-09) from snapshots the fact-check could not re-read [8].
- A newsletter says the rename happened in early June 2026 [23]; the repo redirect from March contradicts it.
- Tracxn's "closed down" profile is a different Motia (a semiconductor firm founded 2000) [21]. A reported wind-down date of 2026-09-12 (Simplify) could not be found or checked.
- Whether iii's console has replaced the Workbench and Flows view in full [16].

## References

All sources checked 2026-10-11 unless dated otherwise.

1. Motia blog, "Motia helped us build something incredible", 2026-04-16, read via http://web.archive.org/web/20260518222434/https://blog.motia.dev/motia-helped-us-build-something-incredible/ (live host dead).
2. https://motia.dev, live, and Wayback snapshots 20260414173118, 20260422112926, 20260511025036, 20260607174736.
3. GitHub API https://api.github.com/repos/MotiaDev/motia (resolves to iii-hq/iii, id 911303109); commit, issue, PR and release lists; Wayback captures of github.com/MotiaDev/motia (2026-03-03, 2026-03-07).
4. iii-hq/iii commits: 124975c1 (2026-02-24), #1220 (2026-03-03), #1487 (2026-04-15), #1497 (2026-04-17), https://github.com/iii-hq/iii/commits/main.
5. https://github.com/iii-hq/iii/pull/1512 (merged 2026-04-22).
6. https://github.com/iii-hq/iii/pull/971, /pull/1169, /pull/1262; https://github.com/iii-hq/iii/blob/main/LICENSE.spdx.
7. https://registry.npmjs.org/motia; https://api.npmjs.org/downloads/range/2025-01-01:2026-10-10/motia and the same for `iii-sdk`.
8. Star counters in Wayback snapshots of github.com/MotiaDev/motia and github.com/iii-hq/iii, 2025-02-28 to 2026-07-12.
9. https://github.com/iii-hq/iii/issues/482 (2025-07-23).
10. https://github.com/iii-hq/iii/issues/1132 (comment 2026-01-12).
11. Issues 861, 955, 1133, 1137, 1138, https://github.com/iii-hq/iii/issues; https://github.com/MotiaDev/motia-examples; web searches for case studies.
12. https://github.com/iii-hq/iii/issues/1447 (closed 2026-04-20).
13. https://iii.dev/docs/changelog/0-11-0/migrating-from-motia-js (undated).
14. https://iii.dev, https://iii.dev/blog (posts 2026-04-28 to 2026-08-05), https://iii.dev/manifesto.
15. Motia at tag v0.17.14-beta.196 (2026-01-09): `packages/core/src/types.ts`, `packages/core/src/helper/flows-helper.ts`, `packages/docs/content/docs/` (steps, flows, workbench, testing, Motia Cloud), README.
16. Motia README at tag v1.0.0-rc.26, https://raw.githubusercontent.com/iii-hq/iii/v1.0.0-rc.26/README.md (2026-02-25).
17. Wayback snapshots of motia.dev: 20250203150453, 20250402213317, 20250615184944, 2025-12-14, 20260307004428.
18. Wayback snapshot of https://motia.dev/hosting, 2025-12-05.
19. Heavybit, "Back-end engineering in the age of AI", https://www.heavybit.com/library/article/back-end-engineering-in-the-age-of-ai (2026-03-12).
20. Hacker News search, https://hn.algolia.com/api/v1/search?query=motia (items 44564695, 46938120, 47120081).
21. Funding searches; https://tracxn.com/d/companies/motia/__GFXo1DFkmek460NhbeRrHGQz6JmNoqUuCGnSB9mXda8 and https://www.bouncewatch.com/explore/startup/motia (both a different company).
22. https://github.com/iii-hq/iii-cloud-cli (created 2026-03-26).
23. Signal Over Noise, "Motia became iii", https://signalovernoise.tech/p/motia-became-iii (2026-06-14, paywalled after the intro).
24. Ligand `docs/design.md`, `GLOSSARY.md` and ADRs 0005, 0006, 0007, 0011, 0012, `docs/adr/`.
