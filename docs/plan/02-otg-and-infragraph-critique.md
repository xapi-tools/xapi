# 02 — OTG and InfraGraph: observed flaws and the fix for each

Sources read for this document (as of 2026-11-01): `open-traffic-generator/models`
(`MODEL-GUIDE.md`, `flow/flow.yaml`, `flow/packet-headers/header.yaml`,
`flow/packet-headers/ethernet.yaml`, `device/ethernet.yaml`,
`device/bgp/bgpv4.yaml`, `api/api.yaml`), `open-traffic-generator/openapiart`
(`MODELGUIDE.md`, issue tracker), `Keysight/infragraph` (all 14 files under
`schema/`, `docs/src/model.md`, `query.md`, `blueprints.md`, `version.md`,
`generate.py`, PR #88). Both projects are built with openapiart, so most
flaws are shared and most trace back to the tooling rather than to the
authors. That is the point: the language shaped the models.

Severity: **structural** changes the wire or the model shape, **quality** is
fixable without a wire change, **tooling** is a generator or process defect.

## Part A — Flaws shared by both (inherited from openapiart)

### A1 (structural)

Unions are a `choice` string enum plus one optional sibling property per variant. Nothing in the schema makes the variants exclusive; every variant name appears twice; the enum uids and the property uids are numbered independently and drift (in `Flow.Header`, `ethernet` is enum uid 2 and property uid 3).

- Evidence: `header.yaml`: 25 variants, 140 lines. `link.yaml` Bandwidth and Latency: uids 1-3 vs 2-4.

- xapi fix: `union` with one line per variant and one id per variant. Exclusivity is a type property.

### A2 (structural)

Field ids are hand-assigned, with gaps, out-of-order values and manual `x-reserved-field-uids`.

- Evidence: `Device.Ethernet` has no uid 1; `Bgp.V4Peer` has uid 16 out of sequence; `Flow.egress_packet` is 9 after `payload` 8.

- xapi fix: Compiler assigns ids into `xapi.lock`; deletion reserves automatically.

### A3 (structural)

References are strings with an `x-constraint` yamlpath. SDKs did not enforce them for years (openapiart #325, #326, #345). Reference intent is encoded in the field name suffix `_name`/`_names`.

- Evidence: `Ethernet.Connection.port_name`, `Bgp.V4Interface.ipv4_name`, `Flow.TxRx.Port.tx_name`.

- xapi fix: `ref<Port>`, `ref<Ipv4 | Ipv4Loopback | Dhcpv4Client>`, `[ref<Flow>]`. Checked by every SDK; the type, not the name, says it is a reference.

### A4 (structural)

Composition via `x-include` instead of `allOf`, implemented by a bundler. Includes drag in APIs they should not (openapiart #501); backwards compatibility of include broke (#495); the include target must keep the same property name.

- Evidence: `Device.Ethernet` includes `mac`, `mtu`, `vlans`, `name` from `Device.EthernetBase`.

- xapi fix: `mixin` flattened by the compiler, with ids owned by the including type.

### A5 (quality)

Normative rules live in descriptions ("MUST be unique", "MUST exist in", "MUST be configured for all connection types except simulated_link"). None is machine-checked.

- Evidence: `Device.EthernetBase.mac`, `Instance.device`, `Infrastructure.Endpoint.instance`.

- xapi fix: `key`, `ref<T>`, and `check <expr> : "message"` cross-field rules.

### A6 (quality)

Required-ness and defaults are governed by prose rules in the guide ("Do not mark properties as required if they can hold sensible defaults") that tooling cannot verify.

- Evidence: OTG `MODEL-GUIDE.md`.

- xapi fix: `!` and `=` are the only two mechanisms; `xapic lint` enforces the guide's rules mechanically (e.g. "required field with a default" is an error).

### A7 (tooling)

Validation errors from the generator point nowhere useful.

- Evidence: openapiart #460.
- xapi fix: Diagnostics carry file:line:col and a fix hint; the IR carries source spans.

### A8 (tooling)

Each lifecycle status is an `x-status` object; `deprecated` has no "since" or "use instead"; `under_review` is spelt both `under_review` and `under-review` in the same guide.

- Evidence: openapiart `MODELGUIDE.md`.

- xapi fix: `@deprecated("use x", since="1.30")`, `@status(under_review)`, `@since("1.2")` with one spelling, checked.

### A9 (tooling)

Everything is in one bundled `openapi.yaml`; the Go SDK is one package with thousands of types, the Python SDK is a dynamically built module.

- Evidence: gosnappi, snappi.
- xapi fix: Packages map to Go packages, Python modules and C++ namespaces.

### A10 (tooling)

Extensions (`x-field-pattern`, `x-device-pattern`, `x-constants`, `x-stream`, `x-enum`, `x-unique`, `x-include`, `x-constraint`, `x-status`, `x-field-uid`) are documented inconsistently; the guide's `x-device-pattern` block is labelled `x-field-pattern` and its sample uses an undefined `count` key.

- Evidence: openapiart `MODELGUIDE.md`.

- xapi fix: No extensions. Each concept is a keyword or a declared annotation with a schema.


## Part B — OTG-specific flaws

### B1 (structural)

Packet-field patterns are a meta-schema (`x-field-pattern` with `format`, `length`, `features`) that the generator expands into one concrete type per field: `PatternFlowEthernetDst`, `PatternFlowEthernetSrc`, and so on, hundreds of near-identical messages.

- Evidence: `ethernet.yaml`; generated `otg.proto`.

- xapi fix: Generic `Pattern<T>`, `AutoPattern<T>`, `MetricPattern<T>` unions. One definition; monomorphised by the compiler with stable names (`@proto_name` keeps existing names for wire compatibility).

### B2 (quality)

Well-known constants are `x-constants` maps on a pattern; they do not generate typed constants in every SDK.

- Evidence: `ether_type` constants `ipv4: 2048`.

- xapi fix: Valued `enum EtherType: u16 { ipv4 = 0x0800 … }` used as `pattern<u16 | EtherType>` or `@known(EtherType)`.

### B3 (structural)

Time and size units are in descriptions only ("in seconds", "in bytes").

- Evidence: `Flow.Duration`, `Flow.Rate`, `Metrics` latency fields.

- xapi fix: `@unit(ns)`, `@unit(bytes)`, `@unit(pps)`. Lint: a numeric field whose doc mentions a unit must carry `@unit`.

### B4 (structural)

Error handling is a `200` plus a `default` failure response. Warnings are a list of strings. No error codes, no field paths. openapiart #500 shows `2xx` cannot even be expressed.

- Evidence: `api/api.yaml`, `result/request.yaml`.

- xapi fix: `rpc set_config(Config) -> Warnings throws Error`; `Error { code: ErrorCode, kind, items: [ErrorItem{ path, message }] }`. Validation errors are structured and carry JSON-pointer paths.

### B5 (structural)

`x-stream: client/server` is a transport workaround for large payloads over gRPC (chunked bytes). It leaks into the model and generated signatures.

- Evidence: `set_config`, `get_config`, `get_metrics`.

- xapi fix: Transport chunking is a runtime concern. `stream` in xapi means a semantic stream of messages (for example streaming metrics), not a chunked blob.

### B6 (quality)

Deep nesting forced by the guide rule "avoid simple datatypes at top level". It is a good rule, but with unions expensive to write, it produced wrappers like `Flow.Size.choice = fixed` plus `fixed: integer`.

- Evidence: `flow.size.fixed = 128`, `flow.duration.fixed_packets.packets`.

- xapi fix: Keep the rule; make unions cheap: `size: Size = fixed(64)` where `union Size = fixed { fixed: u32{1..}, increment: SizeIncrement, random: SizeRandom, weight_pairs: … }`.

### B7 (quality)

Metric and state queries take names as `[string]` with the referential rule in prose.

- Evidence: `Metrics.Request.Flow.flow_names`.
- xapi fix: `[ref<Flow>]`.

### B8 (quality)

Implementations reject unsupported features only at `set_config` time; there is no model-level capability discovery.

- Evidence: OTG guide and vendor READMEs.

- xapi fix: Model-level: `rpc get_capabilities() -> Capabilities`; language-level: every field has a stable IR path, so `Capabilities` can list supported paths.

### B9 (quality)

`Device.Ethernet.mac` is optional but mandatory unless `connection` is `simulated_link`. Stated twice in prose.

- Evidence: `device/ethernet.yaml`.

- xapi fix: `check connection is simulated_link || mac != null : "mac is required unless the connection is a simulated link"`

### B10 (quality)

`name` uniqueness scope is a string `format` (`global-unique`, `list-unique`) and not enforced historically.

- Evidence: openapiart `MODELGUIDE.md`.
- xapi fix: `name: string key` (unique in containing list) or `name: string key global`.

### B11 (structural)

Metrics and states are row-oriented lists of objects (`flow_metrics: [FlowMetric]`). Every key is repeated per row, nothing streams, DataFrame conversion is the caller's problem, and the gRPC 4 MB limit is what produced `x-stream`.

- Evidence: `result/metrics.yaml`, `api/api.yaml`.

- xapi fix: `table FlowMetric` and `rpc get_metrics(...) -> stream Table<FlowMetric>`; Arrow IPC batches on the wire, row-JSON kept as a codec option for old clients (see 06).

### B12 (tooling)

The API is bound to one transport style (HTTP paths with gRPC derived by the generator). There is no way to serve the same model over a message bus.

- Evidence: `api/api.yaml`.

- xapi fix: Transport-neutral services: HTTP, gRPC and Redis from one `service` declaration (see 06).


## Part C — InfraGraph-specific flaws

InfraGraph is small (roughly 1,100 lines of YAML in the nine schema files read, 14 files in all) so the problems
are easy to see in full.

### C1 (structural)

Endpoints and node names are strings carrying an undocumented, inconsistent mini-language: Python slice notation `cx5[0:2]`, dotted composition `dgx.cx5[0]`, juxtaposition `server[0]xpu[0]`, and the older dotted form `xpu.0.pcie.nic.0` still shown in `model.md` and `example.yaml`. The schema cannot validate any of it.

- Evidence: `edge.yaml`, `annotation.yaml`, `query.yaml`, `docs/src/model.md`, `schema/example.yaml`.

- xapi fix: Structured `Endpoint { instance: ref<Instance>, index: Slice, component: ref<Component>, component_index: Slice }` with `model Slice { start: u32 = 0, stop: u32 }`. A compact string form, if wanted, is a documented `@compact` serialisation of that structure, parsed and validated by the runtime.

### C2 (structural)

Structured data is stuffed into strings because the tooling forbids `oneOf` and free-form objects. Annotations are `attribute: string, value: string`; `GraphResponseContent.networkx` is "a networkx node-link graph serialized as YAML" in a `string`; PR #88 keeps document `value` as a string for the same reason.

- Evidence: `annotation.yaml`, `graph.yaml`, PR #88 description.

- xapi fix: `json` scalar type (proto `google.protobuf.Value`/`Struct`, Python `dict`, Go `json.RawMessage`, C++ `nlohmann::json`) with optional `@schema(ref)`; attribute values typed as `union Value { string, i64, f64, bool, json }`.

### C3 (structural)

Component kinds are empty placeholder messages (`Component.Cpu`, `Component.Xpu`, `Component.Nic`, `Component.Memory`, `Component.Port`, `Component.Switch` have no properties). The union exists only to carry a tag. `Component.Custom.type` reintroduces an untyped string tag. Issue #11 says the `npu` choice was "too narrow".

- Evidence: `component.yaml`.

- xapi fix: `union Kind { cpu, xpu, nic, memory, port, switch, device: ref<Device>, custom: string }` with unit variants. When attributes appear, variants grow payloads without a wire break.

### C4 (structural)

Quantities with units are unions of unit-named numbers: `Link.Physical.Bandwidth` is `gigabits_per_second` or `gigabytes_per_second` or `gigatransfers_per_second`; `Latency` is `ms`/`us`/`ns`. Two typos in the enum docs.

- Evidence: `link.yaml`.

- xapi fix: `bandwidth: u64 @unit(bps)`, `latency: u64 @unit(ns)`. One canonical unit; the SDK offers converters.

### C5 (quality)

No defaults anywhere in the schema, and almost no constraints. `Component.count` and `Instance.count` are `int32` with no minimum though the docs say count is at least 1.

- Evidence: `component.yaml`, `instance.yaml`.
- xapi fix: `count: u32{1..} = 1`.

### C6 (quality)

Referential rules are prose: `Instance.device` "MUST exist in the array of Infrastructure devices"; `Infrastructure.Edge.link` "must exist in the infrastructure's links array"; `Device.Endpoint.device` must match a device and that device must match an instance.

- Evidence: `instance.yaml`, `edge.yaml`.
- xapi fix: `device: ref<Device>!`, `link: ref<Link>!`.

### C7 (quality)

Duplicate near-identical shapes: `Device.Edge` vs `Infrastructure.Edge`, `Device.Endpoint` vs `Infrastructure.Endpoint`; `Edge.Common.scheme` shared by include. `ep1` and `ep2` descriptions are copy-pasted ("the other endpoint" on both). `Infrastructure.Edge.ep2` inlines `required: [device, component]` though the referenced type requires `instance`.

- Evidence: `edge.yaml`.

- xapi fix: `model Edge<E> { ep1: E!, ep2: E!, scheme: Scheme = one2one, link: ref<Link>! }` used as `Edge<DeviceEndpoint>` and `Edge<InstanceEndpoint>`. The inline-required inconsistency is unrepresentable.

### C8 (quality)

`scheme` values `one2one`, `many2many`, `ring` have no documented semantics.

- Evidence: `edge.yaml`.

- xapi fix: Mandatory `///` docs on every enum value; lint fails otherwise (the repo already sets `strict_description_validation="all"`, which shows the appetite).

### C9 (quality)

Expansion by `count` plus slice indexing means per-index attributes can only be expressed through annotations after `set_graph`, and the graph cannot be reloaded from `get_graph` output (reviewer comment on PR #88).

- Evidence: PR #88 review thread.

- xapi fix: Make the expanded graph a first-class model (`Graph { nodes: [Node], edges: [GraphEdge] }`) that `get_graph` returns and `set_graph` accepts, so a round trip is lossless. The compact `Infrastructure` form compiles to it.

### C10 (quality)

The query API is an ad hoc filter object; later work adds JSONPath strings. Shortest-path endpoints "should be an EXACT match" in prose.

- Evidence: `query.yaml`, PR #88.

- xapi fix: Typed `NodeSelector` union (`id: NodeId`, `slice: Endpoint`, `attributes: AttributeFilter`), so exactness is a type not a sentence.

### C11 (tooling)

Every RPC is `POST` with `x-stream: client`, including `get_graph`.

- Evidence: `api.yaml`.
- xapi fix: `rpc get_graph(GraphRequest) -> Graph @http(GET "/graph")`; no stream flag.

### C12 (tooling)

`$ref :` with a space before the colon in `graph.yaml`; `Graph.Type` is unreferenced; uid gaps.

- Evidence: `graph.yaml`, `component.yaml` (uids 4 → 10).
- xapi fix: `xapic fmt` and `xapic lint` (unused type is a warning).

### C13 (structural)

The expanded graph, which can reach hundreds of thousands of nodes and edges, is returned either as the compact `Infrastructure` plus annotations or as a networkx node-link document serialised to YAML inside a JSON string. Neither streams, and the second is opaque to every non-Python consumer.

- Evidence: `graph.yaml` (`GraphResponseContent.networkx: string`).

- xapi fix: `get_graph` returns `Table<Node>` and `Table<GraphEdge>` as Arrow batches; networkx, pandas and DuckDB load them directly (see 06).


## Part D — What is right and should be preserved

- The OTG guide's principles: model use-cases not features, one way to get a
  piece of information, automation-first, every element documented, snake_case
  fields and PascalCase types, `name` as primary key, deprecate before remove.

- OTG's separation of config, control and monitor RPCs, and the
  `Warnings` return on mutating calls.

- InfraGraph's core idea: a compact blueprint (`Device`, `Instance`, `Edge`)
  that compiles into an expanded graph; annotations layered on top; a query API.
  The flaws are in the encoding, not the idea.

- InfraGraph's semantic versioning policy and single version location.

## Part E — Metrics to publish (Phase 2 and Phase 6)

For each of OTG and InfraGraph, regenerated in CI into `metrics/`:

| Metric | How measured |
|---|---|
| Lines of model source | `wc -l` of YAML vs. imported xapi vs. redesigned xapi |
| Number of generated protobuf messages | descriptor count; expect a large drop from generics |
| Prose-only rules | count of `MUST` in descriptions vs. count of `key`, `ref`, `check` in xapi |
| Wire compatibility | `buf breaking` against the published `.proto` |
| Validation coverage | conformance cases that the old SDKs do not catch and the new ones do |
