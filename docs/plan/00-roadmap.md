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

Exit: hand-written xapi for OTG `Flow`, `Flow.Header`, `Device.Ethernet` and
the whole of InfraGraph checks clean and produces IR matching its schema.

### Phase 2 — Protobuf and OpenAPI backends, and the importer

- `gen/proto`: proto3 with `optional`, `oneof`, scoped enums, comments,
  reserved ranges, optional `buf.validate` annotations, gRPC services,
  `google.api.http` bindings from `@http`.
- `gen/openapi`: OpenAPI 3.1 for documentation and for existing OTG tooling.
- `tools/import-openapiart`: openapiart-flavoured OpenAPI → xapi, seeding the
  lockfile from `x-field-uid` and `x-reserved-field-uids`.
- Wire-compatibility check: import `open-traffic-generator/models`, emit proto,
  compare descriptors against the published `otg.proto` with `buf breaking`.
  Same for `Keysight/infragraph`.

Exit: `buf breaking` reports no wire-incompatible change for either model.
Line counts of imported xapi vs. source YAML recorded in `docs/plan/metrics.md`.

### Phase 3 — Python SDK and runtime

- `runtime/python` (`xapi-runtime`): error types, JSON codec, proto bridge,
  HTTP and gRPC client bases.
- `gen/python`: typed classes, unions, refs, `validate()`, `to_json()`,
  `from_json()`, `to_proto()`, `from_proto()`, service clients.
- Conformance suite runner for Python.

Exit: conformance suite green; generated OTG client executes the snappi
quickstart flow against ixia-c in CI.

### Phase 4 — Go SDK and runtime

Same shape as Phase 3. Exit: conformance green; `go vet` and `staticcheck`
clean on generated code; gosnappi quickstart equivalent runs against ixia-c.

### Phase 5 — C++ SDK and runtime

C++20, header plus source per package, `std::optional`, `std::variant`,
`std::expected`-style results. Exit: conformance green under GCC and Clang
with `-Wall -Wextra -Werror`.

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
  spec/                      grammar.ebnf, semantics.md, ir.schema.json, lock.md
  cmd/xapic/                 CLI entry point
  internal/
    lexer/ parser/ ast/      front end
    resolve/ check/          name resolution, type checking, constraint checking
    lock/                    lockfile load, assign, reserve
    ir/                      IR builder and JSON emitter
  gen/
    proto/ openapi/ python/ golang/ cpp/   backends (template + post-format)
  runtime/
    python/ go/ cpp/         small hand-written runtime libraries
  tools/import-openapiart/   OpenAPI(openapiart) → xapi
  conformance/               language-neutral validation test cases
  examples/
    otg/ infragraph/         imported and then redesigned models
  metrics/                   line counts and defect counts, regenerated in CI
```

## Decisions to confirm

Each has a recommendation. Work proceeds under the recommendation unless you
say otherwise.

| # | Decision | Recommendation | Why |
|---|---|---|---|
| D1 | Compiler implementation language | **Go** | Single static binary for CI, fast, strong templating, and we dogfood the Go backend by compiling our own output. Python would be easier for the openapiart community to contribute to but is slower and harder to distribute. |
| D2 | Field id strategy | **Implicit, pinned in `xapi.lock`**, with `@id(n)` as an escape hatch | Removes the single biggest source of noise and error in OTG YAML. Merge conflicts in the lockfile surface exactly the wire conflicts that need a human. |
| D3 | Union wire form | **Protobuf `oneof` and protojson** (the set key is the discriminator); `@tagged` emits an explicit `choice` field for OTG compatibility | Removes the duplicated enum-plus-siblings encoding and the offset-by-one uid problem. Imported OTG models use `@tagged` to stay wire-compatible. |
| D4 | Presence model | **Explicit presence for every field**; defaults are applied by `normalize()`, not by construction | Callers can tell "unset" from "set to the default", which protobuf scalars cannot. Required fields are `!`; everything else is optional with an optional default. |
| D5 | Python SDK style | **Generated plain classes** with `__slots__`, type hints and explicit `validate()`; optional pydantic export later | Zero runtime dependency, fast import, readable stack traces. Pydantic is a fine add-on but should not be the core. |
| D6 | Go SDK style | **Structs with exported fields**, sealed-interface unions, typed string enums, `Validate() error` | Mirrors what the Go community writes; abandons gosnappi's interface-per-type and setter chains. |
| D7 | Constraint expression language | **Small xapi expression subset** that compiles to native code in each target and to CEL for `buf.validate` | Native code keeps C++ free of a CEL runtime; CEL export keeps protovalidate users happy. |
| D8 | Import strategy for OTG and InfraGraph | **Import first for wire compatibility, then redesign** | Separates "is the compiler correct" from "is the model good". |
| D9 | CLI and package names | `xapic` binary, `xapi` Go module, `xapi-runtime` Python package, `xapi` C++ namespace | Short, unambiguous, matches the org name. |
