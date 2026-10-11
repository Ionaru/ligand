# Organisms start one another through their outputs

Any output of an organism can be published as an event, and another organism subscribes to it by living in an event habitat such as `event: poll-listings.main`. An ecosystem is therefore a graph of organisms with events as its wires, drawn and read the same way as the levels below it. In v1, events are delivered in-process and at least once: the publishing run starts each subscriber run with an execution key made of the emitting run, the port, the subscriber organism and its event habitat, so when a crash forces a replay, the start is issued again and is deduplicated against the execution keys recorded in the run log (ADR 0012).

## Considered options

- **A separate "emit event" I/O atom.** More explicit inside an organism, but events would then be hidden one level down instead of being the wires of the ecosystem graph.

## Consequences

- The event habitat in the subscriber's organism file is the only place a subscription is declared. `ligand.yaml` lists the organisms, and the canvas derives the ecosystem graph from their habitats.
- No message broker in v1: the run log records every event and the execution key of every run it started.
- Atoms in a subscriber run can still repeat after a crash, so I/O atoms pass their idempotency keys on. Event habitats queue overlapping runs by default.
