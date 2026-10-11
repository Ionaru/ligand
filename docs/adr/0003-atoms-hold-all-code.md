# Atoms hold all code; molecules and organisms are graphs

The levels follow chemistry strictly: an organism is a graph of molecules, a molecule is a graph of atoms, and only atoms hold code. Atoms come in two kinds: a logic atom is one pure business rule with tests, and an I/O atom reads from or acts on the outside world. A lone atom sits in a molecule of one atom, the way noble gases are monatomic molecules, and the canvas draws it as just the atom.

## Considered options

- **Atoms as I/O, molecules as business rules (the first draft).** Atoms and molecules were then sibling leaves and a molecule didn't contain atoms, so the names promised a structure the framework didn't have.
- **Both leaf kinds as molecules, with atoms as the functions inside their code.** This keeps the molecule as the business-rule level, but its atoms would be invisible on the canvas and unchecked, so the containment would hold in name only.

## Consequences

- A molecule is either a single I/O atom or a graph of logic atoms only. Every multi-atom molecule is therefore pure, and every read and write the application does shows up as its own node at organism level.
- The granularity rules apply to logic atoms: one logic atom is one business rule, and programming verbs such as map and filter live inside an atom.
- A logic atom is one synchronous, default-exported function that takes an object with a field per input port and returns an object with a field per output port it emits. It may import only packages listed as pure in `ligand.yaml` and files in its own folder; shared logic is either a logic atom that both places wire to, or a pure package.
- Configuration, such as a URL or search criteria, is a constant value wired to a port in the graph file, for every kind of atom, so it is validated, logged and replayed like any other input. A constant may reference an environment variable (`${env:NOTIFY_URL:-https://example.test}`), resolved when a run starts and recorded in its first log line. Secrets are the only input that doesn't travel on a port: they are referenced by name and handed to I/O atoms through their context.
- Logic atoms can't read the clock, random numbers, environment variables, files or the network; `ligand check` rejects those in their implementation files. Every run carries its start time, which a logic atom can take as an input, and configuration also arrives on ports, so a replay always gives the same answer.
