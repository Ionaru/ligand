# Golden runs are committed, scrubbed recordings that check what an organism sends

Marking a run as golden writes `organisms/<name>.golden/<id>.yaml`, which is committed: the habitat's payload, the start time and every I/O atom's output. Personal fields are replaced by consistent stand-ins (the same value always gets the same replacement) and secret fields by placeholders. When a golden run is checked, I/O atoms play back those outputs, and both the organism's outputs and the input to every write atom must match, because what an organism sends is its real behaviour.

## Considered options

- **Read golden runs from the run log.** Nothing extra to store, but the run log is gitignored and drops personal values after 30 days, so CI and other checkouts couldn't use them.
- **Check only the organism's outputs.** Simpler, but the listing bot's outputs say little; its behaviour is the messages it sends.

## Consequences

- Updating a golden run after a deliberate change is a proposal a developer approves. The AI may never re-bless one (ADR 0001).
- Consistent stand-ins keep equality intact, so a listing ID that appears twice still matches itself after scrubbing.
- Golden runs run in `ligand check` but don't count toward the mutation gate, which is for logic atoms.
