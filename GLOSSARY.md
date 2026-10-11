# Ligand

Ligand is a framework in which a backend is a graph of typed nodes, each held to the tests that define its behaviour. This glossary fixes the words used to describe those graphs and the work around them.

## Levels

**Atom**:
The smallest node and the only kind that holds code; either a logic atom or an I/O atom.
_Avoid_: Function, step, primitive

**Logic atom**:
An atom that holds one business rule and has no contact with the outside world, not even the clock.
_Avoid_: Pure function, rule node

**I/O atom**:
An atom that reads from or writes to the outside world, such as fetching a page or sending a message.
_Avoid_: Adapter, connector, effect node

**Molecule**:
A graph of atoms with its own ports: either a single I/O atom or logic atoms only. A molecule of a single atom is drawn as that atom.
_Avoid_: Sub-flow, group, composite

**Organism**:
A graph of molecules that lives in one or more habitats, which start its runs.
_Avoid_: Flow, workflow, pipeline

**Ecosystem**:
An application: a graph of organisms wired together by events.
_Avoid_: App, system, service

**Biosphere**:
The largest level: a set of ecosystems that work together.
_Avoid_: Platform, landscape

## Nodes and wiring

**Node**:
Anything in a graph that has a contract.
_Avoid_: Step, block, component, function

**Contract**:
A node's name, description and typed ports: what it promises to the rest of the graph. Its tests sit beside it and pin its behaviour.
_Avoid_: Spec, interface, signature

**Implementation**:
Whatever fulfils a node's contract, written by hand or by AI. It can be replaced without changing the contract.
_Avoid_: Body, logic

**Type**:
A named description of the shape of a value on a port, shared across nodes, in which fields can be marked personal or secret.
_Avoid_: Schema, model, interface

**Port**:
A named, typed input or output of a node.
_Avoid_: Pin, socket, parameter

**Main output**:
The output port a node uses for its normal result.
_Avoid_: Return value, default output

**Outcome**:
An extra, named output port for a business case the graph must show, such as a listing without a price.
_Avoid_: Error, exception, branch

**Wire**:
The path along which a value travels from an output port of one node to an input port of another.
_Avoid_: Edge, link, connection

**Mapping**:
A selection, renaming or nesting of fields applied to a value as it travels along a wire.
_Avoid_: Transform, glue, expression

**Pure package**:
A package that logic atoms may import.
_Avoid_: Allowed dependency, whitelisted package

## Runs

**Habitat**:
The surroundings an organism lives in, which feed it the input for each run: a schedule, an HTTP route or an event. An HTTP habitat also takes the organism's answer and always responds.
_Avoid_: Trigger, environment, cage, entry point

**Overlap policy**:
The rule a habitat applies when new input arrives while a run it started is still going: skip it, queue it, or allow both runs at once.
_Avoid_: Concurrency mode, reentrancy

**Run**:
An organism's handling of one input from one of its habitats, from start to finish.
_Avoid_: Execution, job, invocation, instance

**Event**:
An output of an organism, published so that other organisms can start a run from it.
_Avoid_: Message, signal, notification

**Run log**:
The record of what every atom did in every run: its outputs, timing and attempts.
_Avoid_: Journal, trace, history

**Parked run**:
A run stopped at an atom that couldn't continue, waiting for a developer to fix and resume it, rerun it or discard it.
_Avoid_: Dead letter, failed job

**Store**:
The key-value storage Ligand provides for data that has to outlive a run, such as the listings already seen.
_Avoid_: Database, cache, state

**Secret**:
A credential, such as an API token, that Ligand looks up by name and hands only to I/O atoms.
_Avoid_: Credential, key, password

**Personal field**:
A field holding data about a person. Its values are kept only for a limited time and replaced with generated values in fixtures; values that can't be replaced automatically are flagged for a developer to scrub.
_Avoid_: PII, sensitive field

**Secret field**:
A field whose value is never written to the run log.
_Avoid_: Sensitive field

**Recording**:
One saved real exchange between an I/O atom and the outside world, a request and its response, played back in the atom's tests and in development.
_Avoid_: Mock, stub, cassette

**Dry run**:
A run in development in which write atoms send nothing and get recorded or synthetic responses instead.
_Avoid_: Simulation, test run

## Tests

**Example**:
One row of a node's tests: an input and the exact set of output values it must produce, and nothing else.
_Avoid_: Test case, sample

**Fixture**:
A saved real-world input, such as a results page, paired with the output it must produce.
_Avoid_: Snapshot, sample file

**Property**:
A rule that every output of a node must satisfy, checked against many generated inputs.
_Avoid_: Invariant, assertion

**Golden run**:
A recorded run, scrubbed and kept with the organism as one of its tests: I/O atoms play back their recorded outputs, and both what the organism outputs and what it sends must match.
_Avoid_: Snapshot, baseline, replay test

## Tools

**Canvas**:
The local visual editor where graphs are drawn and edited, and where each node's test status and last-run values are shown.
_Avoid_: Workbench, designer, editor

## AI

**AI task**:
One unit of AI work: generating, describing, fixing or improving a node, or diagnosing a parked run.
_Avoid_: Job, prompt

**Diagnosis**:
An AI's short report on why a run parked when no code is at fault, attached to the parked run.
_Avoid_: Postmortem, analysis

**Fix loop**:
The path from a parked run to a fix: its failing input becomes an approved example, then an AI implementation change that passes all tests, or an escalation.
_Avoid_: Self-healing, auto-fix

**Proposal**:
An example, a contract description or an implementation change suggested by an AI, waiting for a developer to approve or reject it.
_Avoid_: Suggestion, draft, patch

**Escalation**:
An AI's report that it couldn't produce an implementation that passes all tests, with the reason, handed back to a developer in place of a fix.
_Avoid_: Failure, give-up, abort
