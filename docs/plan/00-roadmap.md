# 00 — Roadmap

## Vision

One source of truth for a data model and its RPCs, written in a language that is
pleasant to read in a code review, compiled to artifacts that are pleasant to
use in each target language, and exported in a form that tools can consume
without reimplementing the compiler.

## Goals

1. **Terse.** A model should take roughly a quarter of the lines of the
   equivalent OpenAPI, with no loss of information. A 25-variant union is one
   screen, not 140 lines.

2. **Semantics-first.** Required-ness, defaults, ranges, lengths, patterns,
   uniqueness, keys, references, units, lifecycle status and documentation are
   language features, not vendor extensions or prose.

3. **Stable wire identity without bookkeeping.** Field numbers are assigned by
   the compiler and pinned in a lockfile. Authors never type or reserve ids.

4. **Idiomatic generators.** Generated Python looks like typed Python, Go like
   Go, C++ like modern C++. No interface-per-message, no dynamic attribute
   magic, no 100k-line single files.

5. **One validation contract.** Every generated SDK enforces the same rules and
   reports the same error codes and paths. A cross-language conformance suite
   proves it.

6. **Automation-ready.** `xapic ir` emits a canonical JSON IR with a published
   JSON Schema. Docs, MCP servers, test generators, linters and migration tools
   build on the IR, never on source.

7. **Prove it on real models.** Import OTG and InfraGraph, regenerate their
   protobuf with identical field numbers, and show the line-count and
   defect-count delta. Then propose the fixed models.

8. **Transport-neutral RPC.** One service implementation serves HTTP, gRPC and
   Redis. JSON, Protobuf and Arrow are codecs chosen per call, never baked into
   generated types. Chunking and limits are transport concerns (no `x-stream`).

9. **Big tables are a type.** `table` declarations compile to columnar
   encodings (Arrow IPC first, columnar JSON and columnar protobuf as
   dependency-free fallbacks) with streaming, DataFrame interop and the same
   validation as everything else.

## Non-goals (for v1)

- Replacing gRPC or HTTP as transports. xapi describes RPCs; it does not define
  a new wire protocol.

- A general-purpose schema language for arbitrary JSON (no `anyOf`, no
  conditional schemas, no `additionalProperties` games). xapi models are closed.

- Server-side framework generation beyond thin interfaces and request routing.

- Languages beyond Protobuf, OpenAPI, Python, Go and C++. Rust and TypeScript
  are natural later additions; the IR is designed so they need no compiler
  changes.

## Phases

Each phase has an exit criterion that is a passing CI job, not a document.

### Phase 0 — Plan and decisions (this repository state)

Deliverable: these documents, with the decisions below confirmed.

### Phase 1 — Language core and checker

- `spec/grammar.ebnf`, `spec/semantics.md` for v0.1 of the language.

- Lexer, parser, resolver and checker in Go (`internal/`).

- `xapic check`, `xapic fmt`, `xapic ir`, `xapic lock`.

- Lockfile (`xapi.lock`) with deterministic id assignment and reservation.

- Golden tests: every construct in `03-language-sketch.md` parses, checks,
  formats round-trip, and produces stable IR.

- Diagnostics with file:line:col and a fix hint (openapiart issue #460 is
  the anti-pattern).

Exit: hand-written xapi for OTG `Flow`, `Flow.Header`, `Device.Ethernet`,
`table FlowMetric` and the whole of InfraGraph checks clean and produces IR
matching its schema. `spec/encoding-json.md` and `spec/transport.md` are
written in this phase because the IR must carry what they need.

### Phase 2 — Protobuf and OpenAPI backends, and the importer

- `gen/proto`: proto3 with `optional`, `oneof`, scoped enums, comments,
  reserved ranges, optional `buf.validate` annotations, gRPC services,
  `google.api.http` bindings from `@http`.

- `gen/openapi`: OpenAPI 3.1 for documentation and for existing OTG tooling.

- `gen/arrow`: Arrow schema (as IPC schema message and as JSON) for every
  `table`, so non-xapi consumers can validate record batches.

- `tools/import-openapiart`: openapiart-flavoured OpenAPI → xapi, seeding the
  lockfile from `x-field-uid` and `x-reserved-field-uids`.

- Wire-compatibility check: import `open-traffic-generator/models`, emit proto,
  compare descriptors against the published `otg.proto` with `buf breaking`.
  Same for `Keysight/infragraph`.

Exit: `buf breaking` reports no wire-incompatible change for either model.
Line counts of imported xapi vs. source YAML recorded in `docs/plan/metrics.md`.

### Phase 3 — Python SDK and runtime

- `runtime/python` (`xapi-runtime`): error types; `Codec` implementations
  for JSON, protobuf and Arrow; `Transport` implementations for HTTP (httpx),
  gRPC (grpcio, including the `grpc+json` codec) and Redis (redis-py, Streams
  with consumer groups); the shared dispatcher for servers.

- `gen/python`: typed classes, unions, refs, tables, `validate()`,
  `normalize()`, codec entry points, service clients and handler interfaces.

- Conformance suite runner for Python, including codec round trips and table
  encodings.

Exit: conformance suite green; generated OTG client executes the snappi
quickstart flow against ixia-c over HTTP and gRPC; the same client talks to a
generated Python server over Redis in CI; `get_metrics` streams Arrow batches
into a pandas DataFrame.

### Phase 4 — Go SDK and runtime

Same shape as Phase 3 with `net/http`, `google.golang.org/grpc`, `go-redis`
and `apache/arrow-go`. Exit: conformance green; `go vet` and `staticcheck`
clean on generated code; gosnappi quickstart equivalent runs against ixia-c;
Go client interoperates with the Python server over all three transports.

### Phase 5 — C++ SDK and runtime

C++20, header plus source per package, `std::optional`, `std::variant`,
`std::expected`-style results. Transports: gRPC (native), HTTP (libcurl or
cpp-httplib behind a CMake option), Redis (redis-plus-plus). Arrow via Arrow
C++ or nanoarrow behind a CMake option. Exit: conformance green under GCC and
Clang with `-Wall -Wextra -Werror`; cross-language interop with Python and Go.

### Phase 6 — OTG-next and InfraGraph-next proposals

- Rewrite both models in xapi applying every fix in
  `02-otg-and-infragraph-critique.md`.

- Migration guide: which changes are wire-compatible, which are not, and the
  shim strategy for the ones that are not.

- Open upstream discussions with the before/after and the metrics.

### Phase 7 — Tooling

LSP server, VS Code grammar, HTML docs generator from IR, JSON Schema export for
instance documents, lint rules (`xapic lint`), an MCP server over the IR.

## Repository layout (target)

```
xapi/
  README.md
  docs/plan/                 planning documents (this set)
  spec/                      grammar.ebnf, semantics.md, ir.schema.json, lock.md,
                             encoding-json.md, encoding-arrow.md, transport.md
  cmd/xapic/                 CLI entry point
  internal/
    lexer/ parser/ ast/      front end
    resolve/ check/          name resolution, type checking, constraint checking
    lock/                    lockfile load, assign, reserve
    ir/                      IR builder and JSON emitter
  gen/
    proto/ openapi/ arrow/ jsonschema/ python/ golang/ cpp/   backends
  runtime/
    python/ go/ cpp/         hand-written runtime libraries: errors, codecs
                             (json, protobuf, arrow), transports (http, grpc,
                             redis), dispatcher, table containers
  transport/                 xapi.transport.redis envelope model (dogfooded)
  tools/import-openapiart/   OpenAPI(openapiart) → xapi
  conformance/               language-neutral validation test cases
  examples/
    otg/ infragraph/         imported and then redesigned models
  metrics/                   line counts and defect counts, regenerated in CI
```

## Decisions to confirm

Each has a recommendation. Work proceeds under the recommendation unless you
say otherwise.

**D1. Compiler implementation language**

Recommendation: **Go**

Single static binary for CI, fast, strong templating, and we dogfood the Go backend by compiling our own output. Python would be easier for the openapiart community to contribute to but is slower and harder to distribute.

**D2. Field id strategy**

Recommendation: **Implicit, pinned in `xapi.lock`**, with `@id(n)` as an escape hatch

Removes the single biggest source of noise and error in OTG YAML. Merge conflicts in the lockfile surface exactly the wire conflicts that need a human.

**D3. Union wire form**

Recommendation: **Protobuf `oneof` and protojson** (the set key is the discriminator); `@tagged` emits an explicit `choice` field for OTG compatibility

Removes the duplicated enum-plus-siblings encoding and the offset-by-one uid problem. Imported OTG models use `@tagged` to stay wire-compatible.

**D4. Presence model**

Recommendation: **Explicit presence for every field**; defaults are applied by `normalize()`, not by construction

Callers can tell "unset" from "set to the default", which protobuf scalars cannot. Required fields are `!`; everything else is optional with an optional default.

**D5. Python SDK style**

Recommendation: **Generated plain classes** with `__slots__`, type hints and explicit `validate()`; optional pydantic export later

Zero runtime dependency, fast import, readable stack traces. Pydantic is a fine add-on but should not be the core.

**D6. Go SDK style**

Recommendation: **Structs with exported fields**, sealed-interface unions, typed string enums, `Validate() error`

Mirrors what the Go community writes; abandons gosnappi's interface-per-type and setter chains.

**D7. Constraint expression language**

Recommendation: **Small xapi expression subset** that compiles to native code in each target and to CEL for `buf.validate`

Native code keeps C++ free of a CEL runtime; CEL export keeps protovalidate users happy.

**D8. Import strategy for OTG and InfraGraph**

Recommendation: **Import first for wire compatibility, then redesign**

Separates "is the compiler correct" from "is the model good".

**D9. CLI and package names**

Recommendation: `xapic` binary, `xapi` Go module, `xapi-runtime` Python package, `xapi` C++ namespace

Short, unambiguous, matches the org name.

**D10. JSON encoding of 64-bit integers**

Recommendation: **JSON numbers**, with an `int64_as_string` codec option

Matches OTG today and every non-JavaScript consumer; protojson's string rule exists only for JavaScript.

**D11. Primary encoding for `Table<T>`**

Recommendation: **Arrow IPC stream**, with columnar JSON and columnar protobuf as dependency-free fallbacks

Zero-copy, streamable in record batches, native in Python, Go and C++, direct path to pandas, polars, DuckDB and Parquet. Row-JSON kept as a compatibility option.

**D12. Redis protocol**

Recommendation: **Redis Streams with consumer groups** for request/reply and server streams; pub/sub only for events

Streams give acknowledgement, replay after reconnect and horizontal scaling of servers; lists and pub/sub give none of those.

**D13. Codec and transport selection**

Recommendation: **Per client or per call**, never per generated type

Keeps generated types pure data; the same service object serves all transports.

