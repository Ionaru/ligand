# EUPL for tools, MPL for runtime libraries, 0BSD for generated files

The tools (CLI, dev server, canvas and the built-in AI path) are licensed EUPL-1.2, whose copyleft also covers giving others access to the software's functionality over a network. The libraries that run inside a user's application (the runtime and Ligand's own I/O atoms) are MPL-2.0, which is copyleft per file, so users' own files stay theirs. Everything Ligand writes into a user's repository (scaffolds, generated types, installed skills) is 0BSD, so no licence terms attach to users' code.

## Consequences

- Dependencies point one way: tools may import runtime libraries, never the reverse.
- Anyone who distributes an application containing an MPL package must tell recipients where its source is. Running it on one's own servers carries no such duty.
- EUPL-1.2's compatibility clause allows a work combining EUPL and MPL code to be distributed under MPL-2.0, which has no network clause. This needs legal review if protection against hosted use is the goal.
