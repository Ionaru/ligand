# HTTP habitats always answer

An organism lives in its habitats rather than being started by a trigger: the habitat feeds it input and, for HTTP, takes its answer. An HTTP habitat owns the whole exchange and sends exactly one response per request, whatever happens inside. It decodes the request with a codec chosen by `Content-Type` and encodes the answer with one chosen by `Accept`, so the organism, its examples and its golden runs only ever see typed values.

## Considered options

- **Format handling in atoms.** Atoms would receive raw bodies and parse them, so every organism would repeat the same parsing and its tests would depend on the wire format.
- **Let any emitted output answer.** ADR 0002 allows several outputs to emit in one run, which would leave the response ambiguous or dependent on timing.

## Consequences

- Before a run starts: failed authentication answers 401, an unsupported `Content-Type` 415, an unsatisfiable `Accept` 406, and input that fails its schema 400.
- An organism in an HTTP habitat must emit exactly one mapped output per run. Zero or several is a logic bug: the caller gets a 500 and the run parks with its input captured. A run that parks for any other reason also answers 500.
- The habitat waits up to 30 seconds for an answer (the response wait), then answers 504 while the run carries on. With `respond: accepted` it answers 202 with the run ID at once; an output-count error then only parks the run.
- The response mapping gives every output a status, and selects its headers from fields of the output value; the organism computes full header values, such as a `Location` URL, in a logic atom.
- The response mapping is type-checked: `ligand check` fails if an output has no status, if a header refers to a field that doesn't exist or isn't a string, or if the route's path parameters, query and named headers don't match the organism's input type.
- Every response carries a `Ligand-Run-Id` header.
- Ligand ships codecs for JSON, XML, plain text, URL-encoded and multipart forms, and bytes. XML follows the schema: arrays where the schema says array, and OpenAPI's `xml` annotation for names, namespaces and attributes. `ligand check` rejects a media type that can't represent a port's type, such as `text/plain` for an object.
