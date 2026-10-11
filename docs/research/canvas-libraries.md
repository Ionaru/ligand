# Libraries for the canvas

Research date: 2026-10-12.

## 1. Question and constraints

Which libraries should the canvas be built with? The design doc says the canvas is "built on React Flow" [36]; this note confirms or challenges that provisional choice. The search started from the two inputs given, the awesome-node-editors list [1] and the Stack Overflow question on node-editor libraries [2], and went wider: other curated lists, GitHub topic pages and discussions [3], the source of about 30 real products [5], an npm keyword search whose 938 leftover packages were each classified [4], and layout and routing libraries. That gave 130 distinct candidates. Nine editors and five layout engines were evaluated in depth, plus two late npm finds (@gravity-ui/graph, Grafloria) and two extra layout engines (ggn-layout, libavoided-js).

Each deep evaluation built a Ligand-shaped snippet (a node with inputs `item` and `price`, outputs `main` and `noPrice`, one wire per input, a cycle check, a wire label, a molecule with an atom inside, state held outside the library), typechecked it with TypeScript 7.0.2 in strict mode, and ran what could run in Node, jsdom or happy-dom. Of the editors, only the two late candidates ran in a (headless) browser. **Browser performance was not measured for any editor**: the user chose to skip those checks. Layout engines were benchmarked locally [31].

Constraints, from Ligand's design, glossary and ADRs [36]:

- The YAML files (graphs, contracts, habitats, constants) are the source of truth; the canvas reads and writes them without running user code, and they can change on disk while the canvas is open (ADR 0006).
- Node positions live in a separate layout file per graph; graphs may have no layout yet.
- Named, typed ports; a main output plus outcomes; one wire per input, any number per output; acyclic; type-checked wires flagged while drawn; mappings and constants on wires and ports; a warning above three inputs.
- Levels: ecosystem (derived, read-only) > organism > molecule > atom, with drill-down; a molecule of one atom is drawn as the atom.
- Overlays: test status, last-run value per wire, dry-run marking, AI tasks and proposals. Nodes and wires need rich, interactive DOM.
- Sizes: tens of nodes per organism, up to low hundreds in the ecosystem view.
- Nice to have: keyboard use, accessibility, theming, undo/redo, minimap, copy/paste.
- Licence: the canvas ships in `@ligand/cli` under EUPL-1.2 (ADR 0007). No commercial, source-available, key-gated or paid-tier-gated dependency.

## 2. Recommendation

**Keep React Flow (`@xyflow/react`, pinned exactly at 12.12.0) with React as the canvas framework, elkjs in a worker for automatic layout, and Ligand's own files as the only persisted state.** The facts that decide it:

1. **It is the only candidate that combines all of these**: a stable MIT release with nothing behind a paid tier; controlled nodes, wires and viewport with a change-event stream; a synchronous `isValidConnection` called on every pointer move while a wire is drawn; nodes and wire labels as plain React HTML (not SVG `foreignObject` or canvas drawing); built-in accessibility basics (focusable nodes and wires, an aria-live region); clean published types; about 61 KB gzip [6][7]. Other libraries match single criteria (Section 3), but none matches the set.
2. **The rival with more built-ins is not a safer base.** Grafloria has single-wire ports, undo and copy/paste built in, but is one author's 81-day-old 0.x project [23]. X6 and FlowGram are uncontrolled models that fight files-as-truth [16][19]. JointJS sells its editing UX separately [18].
3. **Its gaps are work Ligand does with any library**: a reconcile layer for files changing on disk scored partial for every candidate, and drill-down, keyboard wiring and file-based undo exist in none (Section 3).
4. **What React Flow saves** is about 126 KB of minified, tested code (`@xyflow/react` 80.9 KB plus `@xyflow/system` 45.2 KB) [6]: pan and zoom, node drag, the wire-drawing gesture with snapping and validation, node and port measurement, wire paths and HTML wire labels, selection box, multi-select, delete with a veto, keyboard node moves, focusable nodes and wires, minimap, controls, toolbars and about 85 CSS variables with colour modes [6][7]. Building all of this from scratch (as Node-RED, Rivet, Flyde and Typebot did [5]) was judged, not costed.

Its production record is broad (13.8M weekly downloads [6]), but the closest editing analogue is Langflow, whose typed ports are validated in `isValidConnection`; Airflow 3, Supabase Studio and Mastra use it too, but whether they edit wires was not checked [5]. Langflow still ships v11 and v12 side by side, and Dify and Flowise are still on v11 two years after v12.0.0, which shows what a major-version migration costs [5][6].

Conditions:

1. **Two layers.** The persisted model (graph YAML, contracts, layout file) stays separate from view state (`selected`, `measured`, `dragging`), which only Ligand's adapter holds. Node key = node name (unique in its graph [36]); wire key = target node plus target port, unique because an input takes one wire; a duplicate arriving from a file gets a suffix by file order and is flagged.
2. **Reconcile, don't rebuild.** On a debounced file change, diff by key, reuse the previous object when nothing changed, carry `selected`, `measured` and `dragging` over, and skip the position of a node being dragged; React Flow wants stable objects (#6022) [6][10]. Call `updateNodeInternals` only for nodes whose port list changed, and heal nodes that have `measured` but no `handleBounds` (open bug #4888, where wires vanish) [10]. Mount one React Flow per open graph, keyed by graph and instance path, and keep each viewport in Ligand's store.
3. **Flag, never drop.** A wire whose port no longer exists in its contract is reported by React Flow as error 008 and not drawn [6]; render a placeholder port so the wire stays visible and flagged. Invalid wires from files are never rewritten.
4. **Split the wire rules.** While drawing, reject only structural faults: direction, self-loop, an occupied input (ignoring the wire being reconnected) and a cycle, read from a reachability index in Ligand's store. A type mismatch is soft: the wire is accepted, flagged and offered the mapping editor, because a mapping is added after the wire exists [36]. The dev server fills a compatibility cache per (output, input) pair of the open graph using the same generated-TypeScript check as ADR 0006, recomputes it when a contract or type file changes, and reports "unknown" while stale. This is not a JSON Schema compatibility checker. Its cost (outputs times inputs per graph) is unmeasured.
5. **Writes and undo.** Every canvas write is a comment- and key-order-preserving YAML edit (ADR 0006); the layout file is written on a position change with `dragging` false. The canvas ignores watcher echoes of its own writes by the hash of the content it last wrote. Undo/redo are inverse operations on files, layout moves included, invalidated when an external change touches the same nodes; not snapshots, which would undo an agent's edits. A rename on the canvas rewrites graph and layout file together; an external rename loses the position and the placer places the node again; layout entries for missing nodes are dropped the next time the canvas writes that file.
6. **Levels.** One flat graph per file is the editing model, swapped on drill-down. Drill-down carries the instance path (for last-run values, which the run log keys by the path of node names [36]; test status belongs to the node itself), a notice that a molecule is shared wherever it is used by name, boundary nodes for the molecule's own ports, and habitats and constants drawn as source nodes. The adapter turns a one-atom molecule into the atom. The design's worked example draws a molecule expanded inside its organism [36], so an expanded read-only view (a parent node whose children take their positions from the molecule's own layout file, since child coordinates are parent-relative) is the second step, not optional. The ecosystem view has no layout file and is laid out by ELK on open.
7. **Layout.** elkjs as in Section 5; positions in the layout file are final; a computed layout is written only on a user action (a move or Tidy), so opening a graph never creates a diff.
8. **Licence hygiene.** Keep the attribution badge visible [9]. Copy no Pro example code, and keep those building the canvas from opening Pro examples (the Pro licence is proprietary and non-transferable [8]). Do not copy React Flow UI components until their licence is confirmed [12]. Ship elkjs unmodified as a separate file under EPL-2.0 (Section 5).
9. **Containment.** Every React Flow import stays in one adapter module; Ligand's model, validator, layout and overlays carry no xyflow types. Confirm the canvas bundler builds 12.12.0 (Vite 8 with rolldown breaks on zustand 4's CommonJS shim, #5734, fixed only in v13; esbuild built it) [6][10]. Avoid APIs already marked to change: `expandParent` (#5916), group nodes (#5937), the deprecated `useHandleConnections` [10]. Pin exactly, as Ligand pins Effect [36], because xyflow publishes no stability policy [6]. Re-evaluate v13 before the canvas ships [11].
10. **Gate.** Before this becomes an ADR, a browser spike draws, validates and reconnects wires, applies a file change mid-drag, drills down with async ELK (the #4888 race), renders the ecosystem view at about 150 nodes with wire labels, and measures performance.

Choosing React also decides the rest of the canvas: React has the larger JSON Schema form ecosystem (Svelte's `@sjsf/form` has about 0.75% of rjsf's downloads [37]), though no form library was checked against Ligand's dialect (Draft 2020-12, cross-file `$ref`, `x-ligand-*`). Canvas tests use Playwright, as the docs recommend; jsdom needs shims [7]. React 19.3 server rendering worked; the React Compiler was not tried [6].

## 3. Comparison

How it was checked: T typecheck, R ran in Node/jsdom/happy-dom, B ran in headless Chromium (behaviour only), S source.

| Candidate | Rendering | One wire per input | Validate while drawing | Controlled state | Nesting | Keyboard, a11y | Editing built-ins | Gzip | Licence | Maintenance | Editing use |
|---|---|---|---|---|---|---|---|---|---|---|---|
| React Flow 12.12.0 [6] | React HTML nodes and labels, SVG wires | Userland: `isConnectable` or `isValidConnection` (T, R) | Sync, every pointer move; `connectingto`/`valid` classes (T, S) | Yes, with change events (T) | Parent-child positioning (T) | Nodes and wires focusable; ports not (S) | Select, delete veto, arrows, snap, minimap; no undo, copy/paste (S) | 61 KB | MIT; Pro sells examples | Funded company, ~2 core maintainers; v13 prerelease | Langflow |
| Svelte Flow 1.7.0 [13] | Same engine, Svelte DOM | Userland, boolean only (T) | Yes (S) | Partial: bound arrays, no change events in 1.x (S) | Same as React Flow | Same as React Flow | Same as React Flow | 59 KB | MIT | Same team; v2 prerelease | Windmill (drag off) |
| Vue Flow 1.48.2 [14] | Vue DOM | Built in, `connectable="single"` (T) | Yes (S) | Partial, deprecated API (T) | Parent-child | v11-era, English-only strings | v11-era | 73 KB | MIT | Solo, no code since 2026-01; v2 as `@xyflow/vue` prerelease | n8n |
| Rete.js 2.0.6 [15] | DOM per render plugin, SVG wires | Replaces, not rejects; model unenforced (R) | Partial (T) | None (T) | Only a CC-BY-NC-SA plugin | None (S) | History, minimap plugins | 24 to 36 KB | MIT, two NC plugins | One maintainer; core idle since 2025-06 | Discourse workflows |
| AntV X6 3.1.8 [16] | SVG plus `foreignObject` | Hook (R) | Yes, highlight (R) | No: mutable model, mirror needed (R) | Parent-child (R) | None (R) | All MIT: history, clipboard, keyboard, minimap, snaplines | 175 KB | MIT | One active maintainer; npm account compromised 2026-05 [17] | DolphinScheduler (1.x) |
| JointJS 4.3.3, @joint/react 4.3.6 [18] | SVG plus `foreignObject` | Hook (T) | Yes (T) | Partial: `syncCells` with echo filter; React controlled mode (R) | Embedding (R) | None (S) | None in core; paid JointJS+ | 142 to 173 KB | MPL-2.0 core | ~3 funded maintainers | pcui-graph |
| FlowGram 1.0.15 [19] | React DOM | Hook; loading bypasses it (R) | Yes (T) | No: library-owned; changed wire data ignored on reload (R) | Containers on one canvas | None (S) | Undo, snap, minimap, dagre, containers | 223 to 246 KB | MIT, no LICENSE in tarballs | 4 people, ~3 commits a month | Coze Studio (0.1.28) |
| Sprotty 1.4.0 [20] | SVG (snabbdom) plus `foreignObject` | On drop only (R) | No, drop-time | Model outside by design; edits not written back (R) | Real containment, ELK (R) | None (S) | Little | 118 KB | EPL-2.0 or GPL-2.0+CE | No release since 2024-12; v2 pending | Axon Ivy |
| GLSP 2.8.0 [21] | As Sprotty | Server check (T) | Yes, server round trip | Server-owned (T) | Containment; nested ELK bugs | Keyboard module, thin ARIA | Undo, copy/paste, marquee | 264 KB | EPL-2.0 or GPL-2.0+CE | 2 to 3 people; repo consolidating | Axon Ivy |
| @gravity-ui/graph 1.12.2 [22] | Canvas; React blocks only at zoom >= 0.7 | Host code (B) | Snapping only; invalid wires still created (B) | Imperative store (B) | Canvas groups (B) | None (B) | Minimap, groups, layered layout; no undo, no Delete key (B) | 70 KB | MIT | ~2 maintainers, v2 rewrite | Read-only viewers |
| Grafloria 0.13.0 [23] | HTML layer over SVG | Built in, `toMaxLinks` (B) | Yes, with reasons (B) | Partial, reconcile gaps (B) | Groups, collapse with proxy wires (B) | Claimed; absent in React (B) | Undo, copy/paste, delete, minimap, ELK (B) | 355 KB | MIT | One author, 0.x | None found |

Layout and routing, local run on Node 24.18.1 (medians) with two graphs: the listing bot (11 nodes, 12 wires, one group) and a synthetic DAG (150 nodes, 293 wires, 10 groups) [31]:

| Engine | Layered left to right | Port side and row | Groups | Keeps stored positions | Wire routes | Gzip | Bot / 150 nodes | Licence |
|---|---|---|---|---|---|---|---|---|
| elkjs 0.12.0 [24] | Yes | Yes, `FIXED_POS`: 693 of 693 ports | Yes, `INCLUDE_CHILDREN`, 60 cross-group wires | No | Orthogonal, 0 off-port ends | 462 KB worker | 10 / 173 ms flat, 18 / 200 ms grouped | EPL-2.0 (GPL-3.0 conditional) |
| dagre 3.1.1 [26] | Yes | No | Yes | No | To node border | 17 KB | 4 / 126 ms; 3.1.x throws on parallel wires | MIT |
| d3-dag 1.2.2 [27] | Yes | No | No | No (layers pinnable) | Centre to centre | 37 KB | 1 / 331 to 4964 ms | MIT |
| MSAGL 1.1.24 [28] | Yes (groups only top-to-bottom) | Router only | Partial | No | Rectilinear, spline | 188 KB | 2 / 56 ms | MIT |
| libavoid-js 0.5.0-beta.5 [29] | Router only | Yes, pins | No | Yes, by design | Orthogonal, avoids nodes | 17 KB + 181 KB wasm | 9 ms; 368 ms at 60 nodes, 6.3 s at 80 | LGPL-2.1+ |
| ggn-layout 0.1.2 [30] | Yes | `FIXED_POS` only | Yes | Not tested | As elkjs | 136 to 188 KB | 4 / 31 ms | MIT AND EPL-2.0 |

## 4. Candidates

**React Flow.** Ports are `Handle` ids you choose, so they are as stable as contract port names [6]. There is no connection limit prop; the free Connection Limit example and `isValidConnection` cover it [6][7]. The validator receives ids only and cannot return a reason; `useConnection()` drives a reason tooltip [6]. Wire labels scale with zoom, so last-run values need a compact form at low zoom, and a mapping editor belongs in a popover [6]. Pro (from $169 a month) sells examples, templates and support and gates no code; every capability Ligand wants beyond the basics (undo/redo, copy/paste, helper lines, auto layout, expand/collapse, selection grouping, wire routing) is a Pro example, so Ligand writes them clean-room [8]. Hiding the badge is legal under MIT but against xyflow's stated policy without Pro, and has logged a development warning since 12.11.4 [9]. v13 (13.0.0-next.0, 2026-10-11) is a draft with no date: zustand 5, React >= 18, ESM-only, a changeset class for changes, `useStore` renamed, `colorMode` dropped, and still no port connection limit [11].

**Svelte Flow.** The same `@xyflow/system` engine in Svelte 5, released together from one monorepo with independent version numbers [13]. 1.x has no change-event stream, so file sync means vetoing the library's own edits; Svelte Flow 2 (prerelease) adds `onnodeschange` [13]. Only Windmill uses 1.x in production, with dragging off [13]. The right choice only if the canvas moves to Svelte, after Svelte Flow 2 is stable. It is no hedge against xyflow stalling: same team.

**Vue Flow and `@xyflow/vue`.** The only xyflow-family library with a built-in one-wire input, and n8n proves it as a full editor [14]. But 1.x is one person's frozen project (last code change 2026-01-23), and its successor `@xyflow/vue` 2.0 is a prerelease published by the xyflow team, authored by Vue Flow's maintainer [14].

**Rete.js.** Good port model and React rendering, Discourse's 2026 workflow editor as a close analogue, and a small core [15]. Against it: the only nesting plugin is CC-BY-NC-SA-4.0 and cannot ship under EUPL [15]; no controlled mode, no node measurement, no accessibility; one maintainer, core idle since June 2025 [15].

**AntV X6.** The most complete open toolkit: every editing plugin is MIT [16]. But the model is mutable and uncontrolled, nodes are SVG `foreignObject`, there is no accessibility, one maintainer made most commits and nothing landed since 2026-08-11, and a maintainer account published credential-stealing versions 3.2.7 and 3.3.7 on 2026-05-19 [16][17]. Its MIT undo and clipboard matter little once Ligand undoes on files.

**JointJS core.** The strongest wiring core outside xyflow: `syncCells` with echo filtering, typed events, graph helpers for cycles, MPL-2.0 (listed in the EUPL Appendix), about three funded maintainers, and a controlled mode in `@joint/react` [18][32]. Undo, selection, clipboard, keyboard, minimap and snaplines are in paid JointJS+ [18]; `@joint/react` was in beta until mid-2026; no accessibility. **The fallback outside xyflow**, ahead of X6 on maintenance and model.

**FlowGram.ai.** Covers much of the checklist out of the box, but its defects hit files-as-truth directly: changed wire data is ignored on reload, removing a port deletes its wires, and loading bypasses `canAddLine` (all ran) [19]. Heavy and declining in activity [19]. Out.

**Sprotty and GLSP.** Sprotty shares Ligand's view-model philosophy and has real containment with ELK [20], but validates only on drop, never writes edits back, and has had no release since 2024-12 with a breaking 2.0 pending [20]. GLSP adds the editing (server-owned model, undo, keyboard module) at 264 KB gzip, a JSON-RPC server, a hard pin to Sprotty 1.4.0 and open nested-layout bugs [21]. Out.

**@gravity-ui/graph.** Yandex's canvas-first editor with React blocks in its detailed zoom tier [22]. `snapCondition` only gates snapping: type-mismatched, occupied-input and same-node drops still created wires until every rule was repeated in the creation handler (headless browser run) [22]. No keyboard or accessibility, wires and groups canvas-only, React peer range ^17 or ^18, a breaking v2 in progress, and only read-only viewers in production [22]. Its strength, thousands of nodes, is not a Ligand need. Watch for v2.

**Grafloria.** Covers more of the rubric out of the box than React Flow: per-port wire limits, validators with reasons, valid-target highlighting, port-aware ELK with placement of new nodes only, undo/redo, copy/paste, collapse with proxy wires, a run overlay [23]. The same runs showed that its React binding lacks the accessibility it claims, that data updates reach cards one render late, that port and wire-endpoint changes are ignored, and that port ids must be unique across the whole diagram [23]. One author made 99.7% of 1,874 commits; npm history is 81 days; no production user found [23]. The strongest watch candidate; copy its validation design.

## 5. Layout and routing

**elkjs layered** is the engine: direction right, `portConstraints: FIXED_POS` with port positions at the UI rows, orthogonal routing, and `hierarchyHandling: INCLUDE_CHILDREN` only where expanded molecules are drawn (the default lost 60 of 293 wires) [24][31]. Load `elk-api.js` with `elk-worker.min.js` as a static worker (`workerUrl`): `elk.bundled.js` works on the main thread but throws inside a Web Worker under any bundler (elkjs #141, #359 open; the fix in PR #378 is unreleased) [24]. Browser worker timings: 31 ms for the bot, 364 ms for 150 nodes in groups [31].

**No engine keeps stored positions.** ELK's interactive options pin layers and order but recompute coordinates: with positions on every node, 147 of 150 still moved (141, mean 184 px, after removing a global shift) [31]. Its `layerChoiceConstraint` and `positionChoiceConstraint` are only read by a Java-only visitor [25]. `elk.algorithm: fixed` cannot route wires, and elkjs has no standalone routing (eclipse-elk/elk#315 was closed by a Libavoid bridge that elkjs does not ship) [24][25]. So ELK runs only for graphs without a layout, for the ecosystem view and for an explicit Tidy; a greedy placer puts new nodes to the right of their upstream neighbours (0 existing nodes moved, 2.7 ms on 150 nodes, crossings 732 to 753 versus 803 for a full relayout) [31]. The placer ignores groups: it can drop a node inside a foreign group's rectangle and does not grow groups [31].

**Wires** are React Flow's own paths (smooth-step) for v1. libavoid-js gives the best obstacle-avoiding routes with exact pins over fixed positions, but takes 0.4 s at 60 nodes and seconds at 80, lacks junctions in the published build (bindings merged in PR #36, unreleased), has an open wasm abort (#23), and needs an explicit wasm URL under Vite [29][31]. Later work, organism-sized graphs only.

**Not used:** dagre ignores ports, and 3.1.0 and 3.1.1 throw or return NaN points on some parallel wires (#521; PR #522 open), so it would need merged wires or 3.0.0 [26]; ELK already handles the bot in 31 ms, so a second engine mainly doubles the test surface. d3-dag has no ports or groups [27]. MSAGL's layered layout ignores ports and initial positions, and its constraints throw "not implemented" (#68) [28]. ggn-layout matched elkjs's output on `FIXED_POS` graphs at about a seventh of the time but is one day old, and its EPL-2.0 part has no GPL option [30][31].

**Licences.** elkjs is declared "EPL-2.0 OR GPL-3.0-or-later" [24]. The EUPL-1.2 Appendix lists EPL v1.0, not 2.0 (it lists GPL v2 and v3, AGPL v3, LGPL v2.1 and v3, MPL v2) [32]; a 2020 Commission overview says EPL-2.0 is covered, which contradicts the binding text [33]. The GPL route is conditional on EPL-2.0 section 3.2(a), so it is not a plain dual licence [34]. The safer reading: ship `elk-worker.min.js` unmodified as a separate file under EPL-2.0 with its LICENSE.md, a source location (EPL-2.0 section 3.1), and an Apache-2.0 notice for the compiled Guava code it contains [24][34]. The Commission reads unmodified, linked components as keeping their own licence, but says there is no case law, and bundling is a grey area [33]. This joins the EUPL questions design.md already marks for legal review [36]. libavoid-js is LGPL-2.1-or-later: section 6 would need the licence text, a notice, an unmodified replaceable wasm, and source or a written offer (6a, 6c or 6d; 6b's shared-library route does not literally fit a shipped wasm) [35]. Not legal advice.

## 6. Risks and containment

| Risk | Containment |
|---|---|
| xyflow stalls (two core maintainers, docs source not public [6]) | Thin adapter; Svelte Flow shares `@xyflow/system`; a fork means `@xyflow/react` plus `@xyflow/system` (0.0.x, pinned exactly by React Flow); JointJS core is the outside fallback |
| v13 migration as costly as v11 to v12 [5][6] | Pin 12.12.0 exactly; avoid APIs marked to change; re-evaluate v13 once stable with a migration guide [11] |
| Wires vanish (#4888) or flicker on re-adoption (#6022) [10] | Identity-preserving reconcile, heal hook, one instance per graph, hide wires until measured; confirm in the spike |
| Toolchain: #5734 (Vite 8 with rolldown), #6038 (`exactOptionalPropertyTypes` errors in published types) [10] | Choose a bundler shown to build 12.12.0; `skipLibCheck` if needed |
| Ports not keyboard-operable; focus order follows the DOM (#5189) [10] | A command-style "connect output X to input Y" palette using the same validator; PR #6012 (keyboard connections) is open, not merged [10] |
| Pro-derived code reaching an EUPL repository [8] | Clean-room rule; written permission from xyflow otherwise |
| elkjs EPL-2.0 against EUPL-1.2 [32][33] | Unmodified separate file with notices; legal review |
| Molecule expansion and drill-down are all custom [6] | Flat graph per file; expanded view read-only first |
| Performance at low hundreds of nodes unmeasured | Measure in the spike; memoised nodes, `onlyRenderVisibleElements` [7] |

## 7. Ruled out

From the triage's 124 entries and the npm sweep's 30 small finds [1]-[5]. Vue Flow, GLSP, ggn-layout and libavoided-js were ruled out in triage but evaluated anyway (Sections 3 and 5); libavoided-js did not avoid obstacles in its runs [31].

| Reason | Libraries |
|---|---|
| Licence: commercial, source-available, non-commercial or none | JointJS+ (paid tier of JointJS), jsPlumb Toolkit / VisuallyJs, GoJS, yFiles for HTML, Syncfusion Diagram, MindFusion JS Diagram, DHTMLX Diagram, KeyLines / ReGraph / Ogma, tldraw SDK (licence key), node-flow-elements (no licence), @metadev/daga-angular, avoid-nodes-edge (non-standard licence, unmaintained), rete-scopes-plugin and rete-structures (CC-BY-NC-SA-4.0), about 12 commercial npm diagram packages (DevExpress, Kendo and others) |
| Unmaintained (no release or push for about two years, archived or deprecated) | react-flow-editor, lit-node-editor, yew-flow, egui_node_graph, litegraph.js, ComfyOrg litegraph fork, Wire, jsPlumb Community, react-digraph, beautiful-react-diagrams, Butterfly (Alibaba), nodl, awslabs diagram-maker, Function Junctions, solid-graph / Odysea, behave-graph / behave-flow, d3-flextree, entitree-flex, @zxch3n/tidy, klayjs |
| Stagnant or tiny, though relevant | Reaflow (built-in ELK and nesting; no commits in 12 months), React Diagrams (projectstorm), Drawflow (no typed ports), react-dag-editor, Svelvet (Svelte 4 era), Flyde (AGPL editor, idle since 2025-07), Flume, oura-node-editor, @blackprint/sketch, ueblueprint (Unreal format) |
| Immature (weeks or months old, few users) | lit-flow, xyflow-vue, ngDiagram, Xenolith graph, AlpineFlow, RealFlow, canvas-harness, flow-connect, sun-hierarchy, buchheim, system-canvas, @hidemikimura/canvas-flow, @graph-giraffe/core, @cascivo/flow, @c2n/flow, @quantumwake/kgraph, @snomiao/rgui, @vanduo-oss/vd3-flowchart, @syzgyn/litegraph.js, @statelyai/flow, obstacle-router (LGPL libavoid port), @knowvah/dot-engine |
| Covered by an evaluated library (old name, port, wrapper, plugin or same approach) | React Flow v11 (`reactflow`), solid-flow, Workflow Builder (Synergy Codes), @tisoap/react-flow-smart-edge, baklavajs, maxGraph, diagram-js (bpmn.io; also a watermark clause to check), LogicFlow, Foblex Flow, ngx-vflow, @ng-draw-flow/core (Angular only), @joint/react (evaluated with JointJS), @xyflow/vue (with Vue Flow), @jalez/react-flow-automated-layout, react-flow-auto-layout, @grkmr/freeflow, react-flow-edge-overlapping, @synergycodes/overflow-ui, @flowdrop/flowdrop, @rozie-ui/rete-react, @foblex/flow-elk-layout, @unovis/dagre-layout, JointJS layout (@joint/layout-directed-graph), rete-auto-arrange-plugin, cytoscape-elk, cytoscape-dagre, @liveblocks/react-flow (needs a hosted service) |
| Not an editor (viewer, renderer, runtime, whiteboard, scene graph) | NoFlo, AntV G6, Cytoscape.js, Sigma.js, cosmos.gl, Langium, Excalidraw, react-flow-builder, ngx-graph, Sequential Workflow Designer, nice-dag, Blockly, vis-network, force-graph, Konva, PixiJS, Fabric.js, graphology, Mermaid, Reactodia, @plait/draw, flowfunc (a Dash component), Eclipse Sirius Web (needs a Java server) |
| Layout engines of the wrong shape (tree-only, force-directed, or Graphviz without pinned positions or port-aware orthogonal routing) | d3-hierarchy, WebCola, @hpcc-js/wasm-graphviz, @viz-js/viz, d3-graphviz, @antv/layout, d3-force, ngraph.forcelayout |
| Products, used as evidence only | noflo-ui, CodeWire, unit, bpmn-js / Camunda Modeler, Node-RED, n8n, Dify, Langflow, Flowise, Windmill, Kestra, Supabase Studio, Apache Airflow UI, Hatchet, Budibase, Sim Studio, ChainForge, Automatisch, ComfyUI frontend, Rivet, Directus Flows, Typebot, Babylon.js node editor, sunag/flow, cables.gl, Dagster and Argo viewers |
| Not libraries | GitHub node-editor topic pages, the React, Svelte and Vue Flow layouting guides |

## 8. Open questions and unverified claims

- **React Flow was never run in a browser.** Drag-time feedback, label scaling, flicker, #4888, a file change mid-drag, Tab order and screen-reader behaviour rest on source, typechecks and Node runs [6]. The spike in condition 10 settles them.
- **Editor performance at Ligand's sizes was not evaluated** (the user chose to skip it), and xyflow publishes no limits [7]. Layout timings above are local runs.
- Whether the canvas bundler builds 12.12.0 (#5734) [10].
- The cost of the type-compatibility cache per graph, and whether to replace rather than reject a wire dropped on an occupied input.
- Whether the ecosystem view gets its own layout file (next to `ligand.yaml`) or is always laid out by ELK.
- elkjs under EUPL-1.2, the Guava notice, and libavoid-js's LGPL duties: legal review [32][33][34][35].
- The licence of React Flow UI components [12]; whether xyflow would permit Pro-derived code in an EUPL repository [8].
- Whether any React JSON Schema form library handles Ligand's dialect; React Compiler behaviour [6].
- The greedy placer with groups; ELK port constraints in Grafloria and Gravity beyond the cases run [23][22][31].
- v13's final contents and date; Svelte Flow 2 and `@xyflow/vue` 2 stability [11][13][14].
- Editing use of Airflow 3, Supabase Studio and Mastra; production users listed but not re-checked [5].
- Firefox and Safari behaviour of every library; all browser runs used Chromium [22][23][31].

## References

All sources checked 2026-10-12 unless dated otherwise. "Local run" means a run on this machine with the versions named.

1. flowrails, awesome-node-editors, https://github.com/flowrails/awesome-node-editors (2026-10-11).
2. Stack Overflow question 72164885, https://stackoverflow.com/questions/72164885, with answers and comments (2026-10-11).
3. GitHub topic pages (node-editor, node-graph, graph-editor, flowchart-editor, diagram-editor, diagramming-library, visual-programming, workflow-editor, flow-based-programming, node-based-ui), https://github.com/topics; xyflow's awesome list and other awesome lists found by GitHub search (2026-10-11).
4. npm registry search by keyword (node-editor, flow-editor, graph-editor, diagram, flowchart, visual-programming, workflow-editor, react-flow, svelte-flow, vue-flow, blueprint, dataflow and others), https://registry.npmjs.org, and https://api.npmjs.org/downloads; 938 packages classified locally (2026-10-11).
5. Product sources at default-branch HEAD via the GitHub API: package.json and canvas code of Langflow (`src/frontend`, `utils/reactflowUtils.ts`), Apache Airflow (`airflow-core/src/airflow/ui`), Supabase Studio, Mastra, Dify (`pnpm-workspace.yaml`), Flowise, n8n, Kestra, Windmill, Discourse (`plugins/discourse-workflows`), Coze Studio, Node-RED, Rivet, Flyde, Typebot and others.
6. `@xyflow/react` 12.12.0 and `@xyflow/system` 0.0.83, npm tarballs and source at tag `@xyflow/react@12.12.0` (commit 3d35b57), incl. `packages/system/src/xyhandle/XYHandle.ts`, `packages/system/src/utils/edges/positions.ts`, `packages/react/src/components/Handle/index.tsx`, CHANGELOG.md; local run: Ligand snippet typechecked with TypeScript 7.0.2 (strict, `skipLibCheck: false`), validator, reconcile and server render run in Node with React 19.3.0, esbuild 0.28.2 bundle.
7. React Flow docs, https://reactflow.dev/learn (accessibility, theming, sub-flows, layouting, performance, state management, migrate to v12) and https://reactflow.dev/examples.
8. React Flow Pro, https://reactflow.dev/pro (incl. FAQ); xyflow Pro License 1.0, https://xyflow.com/pro-license (updated 2026-09-10); Pro markers on the example pages.
9. https://reactflow.dev/learn/troubleshooting/remove-attribution; `@xyflow/react` CHANGELOG 12.11.4 (#5962) and 12.11.6 (#5997).
10. xyflow issues #4888, #6022, #6044, #5189, #5620, #5734, #6038, #4500, #4919 and pull requests #5916, #5937, #6012, https://github.com/xyflow/xyflow.
11. `@xyflow/react` 13.0.0-next.0 on npm (2026-10-11); https://github.com/xyflow/xyflow/pull/5866 (draft, updated 2026-10-09).
12. React Flow UI, https://ui.reactflow.dev (registry items carry no licence).
13. `@xyflow/svelte` 1.7.0 and 2.0.0-next.4, npm tarballs; https://svelteflow.dev; local run: snippet checked with svelte-check, esbuild bundle.
14. `@vue-flow/core` 1.48.2 and `@xyflow/vue` 2.0.0-next.18, npm; https://github.com/bcakmakoglu/vue-flow commits; local typecheck.
15. rete 2.0.6 and plugins (area 2.3.2, connection 2.0.5, react 2.1.2, history 2.2.0), npm; https://retejs.org/docs/licensing; https://github.com/retejs/rete/pull/747; local run.
16. `@antv/x6` 3.1.8 and `@antv/x6-react-shape` 3.0.1, npm; https://github.com/antvis/X6 commits; local jsdom run.
17. GHSA-45hr-5496-7mgx / MAL-2026-3839, "Malware in @antv/x6" (2026-05-19), https://github.com/advisories/GHSA-45hr-5496-7mgx.
18. `@joint/core` 4.3.3, `@joint/react` 4.3.6, npm; https://www.jointjs.com/comparison; https://docs.jointjs.com; local typecheck and run.
19. `@flowgram.ai/*` 1.0.15, npm; https://github.com/bytedance/flowgram.ai; local headless run of the core.
20. sprotty 1.4.0 (2024-12-17), npm; `packages/sprotty/CHANGELOG.md` on main, https://github.com/eclipse-sprotty/sprotty; local run in Node and happy-dom.
21. `@eclipse-glsp/client` and `@eclipse-glsp/server` 2.8.0, npm; https://github.com/eclipse-glsp/glsp-core; GLSP issues 1740, 1820, 1598, https://github.com/eclipse-glsp/glsp/issues.
22. `@gravity-ui/graph` 1.12.2, npm and source at tag v1.12.2, https://github.com/gravity-ui/graph (issues #43, #63, #353); ydb-embedded-ui and ytsaurus-ui sources; local run with React 19.3.0 in headless Chromium (behaviour only).
23. `@grafloria/react` 0.13.0, `@grafloria/engine` 0.6.0, `@grafloria/renderer` 0.7.0, npm; https://github.com/grafloria/grafloria at c010d8a3; local run with React 19.3.0 in headless Chromium (behaviour only).
24. elkjs 0.12.0 (2026-07-17), npm tarball (LICENSE.md, `lib/elk-worker.js`); https://github.com/kieler/elkjs README and issues #141, #346, #359, #377, pull request #378.
25. ELK Layered reference, https://eclipse.dev/elk/reference/algorithms/org-eclipse-elk-layered.html; `InteractiveLayeredGraphVisitor.java` and issue #315, https://github.com/eclipse-elk/elk.
26. `@dagrejs/dagre` 3.1.1 and 3.0.0, npm; https://github.com/dagrejs/dagre/issues/521 and /pull/522.
27. d3-dag 1.2.2, npm, https://github.com/erikbrinkman/d3-dag.
28. `@msagl/core` 1.1.24, npm; https://github.com/microsoft/msagljs/issues/68.
29. libavoid-js 0.5.0-beta.5 (2026-02-23), npm; https://github.com/Aksem/libavoid-js issues #23, #35 and pull request #36.
30. ggn-layout 0.1.2, npm (created 2026-10-11), https://github.com/0xlxx/ggn-layout.
31. Local layout benchmark: Node 24.18.1 with elkjs 0.12.0, `@dagrejs/dagre` 3.1.1 and 3.0.0, d3-dag 1.2.2, `@msagl/core` 1.1.24, libavoid-js 0.5.0-beta.5, ggn-layout 0.1.2, libavoided-js 0.1.2; browser worker runs from Vite 8.3.4 and 6.4.4 builds in Chromium; repeated by an independent check.
32. EUPL-1.2 official text, Article 5 and Appendix, https://joinup.ec.europa.eu/sites/default/files/custom-page/attachment/2020-03/EUPL-1.2%20EN.txt.
33. Interoperable Europe: EUPL FAQ, https://interoperable-europe.ec.europa.eu/collection/eupl/faqs; compatible licences, https://interoperable-europe.ec.europa.eu/collection/eupl/eupl-compatible-open-source-licences; EUPL overview 2020 (PDF).
34. Eclipse Public License 2.0, as shipped in elkjs 0.12.0 LICENSE.md.
35. GNU LGPL 2.1, https://www.gnu.org/licenses/old-licenses/lgpl-2.1.txt.
36. Ligand `docs/design.md`, `GLOSSARY.md` and ADRs 0003, 0005, 0006, 0007, 0008, `docs/adr/`.
37. npm downloads API for `@rjsf/core` and `@sjsf/form`, https://api.npmjs.org/downloads.
