# One value per wire per run, in acyclic graphs

A run is one firing of an organism's habitat. Graphs are acyclic and each node runs at most once per run; a wire carries one value per run, and that value may be a list. A node may emit on any subset of its outputs, and a node runs only when all its required inputs have a value, so an output that isn't emitted skips everything downstream of it. We chose this over item-by-item message streaming (as in Node-RED) because it keeps atoms plain function calls, keeps every run replayable, and keeps the graph readable.

## Consequences

- An empty list is a value and does start downstream nodes. Skipping everything downstream of an output is an explicit choice by a node: it doesn't emit on that output.
- A node that splits items between outcomes emits on several outputs in the same run, e.g. 18 priced listings on the main output and 2 on `noPrice`.
- Repetition such as pagination lives inside an atom, not as a loop in the graph.
- Order comes only from data; there are no control-only wires. An atom that must run after another takes that atom's output as input: Save seen IDs saves the IDs that Send Telegram actually sent.
- An input can be optional (`required: false`). A node runs once every node feeding it has finished or been skipped, all its required inputs have values, and at least one input has a value. An absent input is an omitted field, in examples and in the implementation alike.
- An input port takes exactly one wire, and merging values is the job of a logic atom with several inputs. An output port can feed any number of wires.
