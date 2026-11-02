# 06 — Encodings, transports and tables

Two requirements that reshape the runtime design:

1. An RPC must be callable over **HTTP**, **gRPC** and **Redis**, with **JSON**
   as a fully supported encoding on every one of them, not a debugging
   convenience.

2. **Big tables** (metrics, states, expanded graphs, captures, telemetry) must
   move between processes efficiently, and the format must be a deliberate
   choice rather than "a JSON list of row objects".

The consequence is a strict separation inside every generated SDK:

```
model types  ──codec──▶  bytes  ──transport──▶  peer
 (generated)   JSON | protobuf | arrow      HTTP | gRPC | Redis
```

Generated clients and servers never know which codec or transport they run on.
That is also what finally removes OTG's `x-stream` from the model: chunking is
a transport concern.

## 1. Encodings (codecs)

| Codec | Content type | Use | Dependency in SDK |
|---|---|---|---|
| **JSON** (canonical, specified by xapi) | `application/json` | Default for HTTP, Redis, files, humans, automations | None. The generated code includes its own encoder/decoder. |
| **Protobuf binary** | `application/x-protobuf`, gRPC default | Compact, fast, schema-evolving | protobuf runtime, optional extra |
| **Arrow IPC stream** | `application/vnd.apache.arrow.stream` | Tables only (section 3) | Arrow runtime, optional extra |
| gRPC+JSON | `application/grpc+json` | gRPC with readable payloads, debugging, polyglot bridges | gRPC custom codec, shipped in runtime |

Rules that hold for every codec:

- The **JSON encoding is normative** and specified in `spec/encoding-json.md`.
  It is not "whatever protojson does". The protobuf codec must round-trip to
  exactly the same JSON through the generated types, which the conformance
  suite checks.

- A codec is chosen per call or per client, never baked into generated types.

- The default **JSON number rule**: `i64`/`u64` are JSON numbers (matches OTG
  practice and every non-JavaScript consumer). A codec option `int64_as_string`
  exists for JavaScript peers. The IR records the type, so consumers know.

- Other row-oriented binary encodings (CBOR, MessagePack) are deliberately not
  in v1: for small messages protobuf wins, for tables Arrow wins.

### JSON encoding summary

| xapi | JSON |
|---|---|
| field names | snake_case as declared (`@json_name` overrides) |
| unset field | omitted; `normalize()` before encode produces a complete document |
| `bool`, integers, floats | JSON literals; `NaN`/`Inf` as strings `"NaN"`, `"Inf"`, `"-Inf"` |
| `string`, `mac`, `ipv4`, `ipv6`, `cidr`, `hex`, `uuid` | strings in canonical form |
| `bytes` | base64 (standard alphabet, padded) |
| `duration` | string, `"1.5s"` style, or integer nanoseconds with codec option |
| `timestamp` | RFC 3339 UTC string |
| `json` | inline JSON value |
| enum | string name |
| union | object with exactly one key (the variant); `@tagged` adds `"choice": "variant"` |
| unit variant | `{"auto": {}}` |
| `[T]` | array; `{K: V}` object with keys stringified |
| `ref[T]` | string |
| `table[T]` | object of arrays (section 3); `list[T]` is the row form |

Field order on encode is source order, so diffs of encoded documents are
stable. Decoders accept any order.

## 2. Transports

All three transports implement one runtime interface per language
(`Transport` with `call`, `stream_call`, `serve`). Generated code contains:

- a **client** per service: one method per rpc, taking a request type and
  returning the response type (or an iterator/stream);

- a **handler interface** per service for implementors;

- a **dispatcher** that routes an envelope to the handler and encodes the
  result, shared by every transport.

### HTTP

- Method and path from `@http`, default `POST /<service>/<rpc>`.

- Request `Content-Type` selects the codec; `Accept` selects the response
  codec; the server may answer in the request's codec when `Accept` is absent.

- Streaming rpcs: newline-delimited JSON (`application/x-ndjson`), or a
  length-prefixed protobuf stream, or an Arrow IPC stream for tables.

- Errors: HTTP status from the error's `code` category, body is the xapi
  `Error` in the negotiated codec.

- Large payloads are the transport's problem: HTTP has no 4 MB default limit,
  so `set_config` with a 50 MB config needs no chunking in the model.

### gRPC

- Proto services generated from `service`; default codec protobuf.

- `grpc+json` codec registered in every runtime so the same stubs can carry
  JSON (useful with gateways and for polyglot debugging).

- Big messages: the runtime raises the per-call message limit and, for
  `stream` rpcs, sends chunks. There is no `x-stream` flag on the model; a
  unary rpc with a big payload is still unary to the caller.

- Collections: an rpc whose top-level request or response is a `list`, `map`
  or `table` (or a union of them) is generated as a streaming method; each
  message is one chunk, a complete value of the same type (sketch, 8a).

### Redis

Redis is used as a message bus, in three patterns, all defined on top of
one envelope type written in xapi itself (dogfooding):

```kotlin
package xapi.transport.redis

/// One request or response carried over Redis.
model Envelope {
  /// Correlation id; a response copies the request's id.
  id:       uuid!
  /// Fully qualified rpc, for example `otg.Otg.set_config`.
  rpc:      string!
  encoding: Encoding = json
  /// Stream or list the reply must be written to (requests only).
  reply_to: string
  deadline: timestamp
  /// Sequence number and end marker for streamed responses.
  seq:      u32
  last:     bool = false
  payload:  bytes
  error:    Error
}

enum Encoding = json { json, protobuf, arrow }
```

| Pattern | Redis primitive | Used for |
|---|---|---|
| Request/reply | `XADD` to `xapi:<service>:req`, server reads with `XREADGROUP` (consumer group per service instance set), replies with `XADD` to the `reply_to` stream; client `XREAD` blocks on it. | Every unary rpc |
| Server stream | Same, with `seq`/`last` on each reply entry | `stream` responses, table batches |
| Pub/sub | `PUBLISH` on `xapi:<service>:events:<name>` | Fire-and-forget events (not request/reply) |

- Streams, not lists, so replies survive a client reconnect and work is
  acknowledged (`XACK`), which gives at-least-once delivery and horizontal
  scaling of servers through consumer groups.

- Payload size: Redis values are capped at 512 MB, but entries above a few MB
  hurt latency; tables are sent as one record batch per entry.

- Client libraries: `redis-py`, `go-redis`, `redis-plus-plus` (C++). All are
  optional extras of the runtime.

### Transport-neutral semantics

Everything a caller can observe is defined once in `spec/transport.md`:
deadlines, cancellation, error mapping, idempotency (`@idempotent` on an rpc
allows transport-level retry), streaming termination, and metadata (a
string map carried as HTTP headers, gRPC metadata or envelope fields).

## 3. Tables

### Why not "a list of row objects"

OTG `get_metrics` returns `flow_metrics: [FlowMetric]` as JSON. For 20,000
flows and 15 columns that is 300,000 key strings on the wire, parsed into
300,000 dictionary entries, then usually converted to a DataFrame by the user.
It cannot stream, and over gRPC it hits the 4 MB limit, which is where
`x-stream` came from. InfraGraph has the same problem in reverse: an expanded
graph of 100k nodes is returned as a YAML string inside a JSON string.

### `table[T]`

```kotlin
/// Per-flow traffic metrics.
model FlowMetric {
  name:      ref[Flow] key
  frames_tx: uint!  @unit(frames)
  loss:      float{0..100} @unit(percent)
  state:     TransmitState
}

union MetricsResponse {
  flow: table[FlowMetric]
  port: table[PortMetric]
}

rpc get_metrics(MetricsRequest) -> MetricsResponse
```

A table is the column-major layout of a sequence of flat rows; `list[T]` is
the row-major layout of the same values. The model is ordinary; flatness
(primitives, aliases, enums, refs, `duration`, `timestamp` only) is checked
where `table[T]` is used. Columns are nullable unless `!` or `key`; a `key`
column is unique across the table. The language-level rules, the JSON and
protobuf wire forms and the chunking rule are in the language sketch, section
8a; this document covers the codecs and the Arrow path.

### Encodings of `table[T]`

| Encoding | Shape | When |
|---|---|---|
| **Columnar JSON** | object of arrays, `null` for a missing cell, all-null columns may be omitted | the JSON codec, always |
| **Columnar protobuf** | generated `TTable` message: one packed `repeated` per column with the row model's ids, plus a validity bitmap (`bytes`, bit per row) at `1000 + id` for each nullable column that has nulls | the protobuf codec, always |
| **Arrow IPC** | record batches with the schema below; Arrow validity bitmaps carry the nulls | when negotiated; the fast path for large tables |

Row layout for the same data is spelled `list[T]` in the model, not chosen by
a codec option. A JSON decoder accepts either layout because an array and an
object cannot be confused, so a row-oriented producer and a columnar consumer
interoperate.

The Arrow mapping (xapi → Arrow): `bool`→`bool`; `int` and its aliases → the
pinned width or `int64`; `uint` and aliases → unsigned widths or `uint64`;
`float`→`double`, `f32`→`float`; `string`→`utf8` (`large_utf8` above 2 GB per
batch); `bytes`→`binary`; enum→`dictionary<int32, utf8>`; `ref[T]`→`utf8`
with field metadata `xapi.ref=T`; `duration`→`duration(ns)`;
`timestamp`→`timestamp(ns, UTC)`. Formatted string aliases such as a future
`Mac` may pin `fixed_size_binary` through `@repr`. Table-level metadata
carries the xapi type name and project version so a consumer can validate
without the IR.

Chunk size defaults to 64k rows or 4 MB, whichever comes first, overridable
with `@chunk(rows=…)`; body compression `zstd` is on by default for Redis and
HTTP, off for gRPC (which compresses itself).

### Generated table API

Every language gets a container type per table model with the same
operations:

| Operation | Python | Go | C++ |
|---|---|---|---|
| build | `FlowMetricTable.from_rows([...])`, `.append(row)` | `NewFlowMetricTable()`, `.Append(row)` | `FlowMetricTable::from_rows`, `.append` |
| read | `.rows()` iterator of typed rows, `.frames_tx` typed column | `.Rows()`, `.FramesTx() []uint64`, `.Loss() []*float64` | `.rows()`, `.frames_tx()` span |
| validate | `.validate()` (keys unique, ranges, refs) | `.Validate()` | `.validate()` |
| convert | `.to_list()` / `FlowMetricTable.from_list()` | `.ToList()` / `FromList()` | same |
| Arrow | `.to_arrow()` → `pyarrow.Table`, `.from_arrow()` | `.ToArrow()` → `arrow.Record`, `.FromArrow()` | `to_arrow()` → `std::shared_ptr<arrow::Table>` |
| frames | `.to_pandas()`, `.to_polars()` (via Arrow, zero-copy where possible) | — | — |
| chunks | iterate a chunked response, or `.collect()` | `Recv()` loop, or `Collect()` | same |

Python and C++ Arrow support is mature (`pyarrow`, Arrow C++). For Go,
`github.com/apache/arrow-go` is used; for very small C++ deployments the
`nanoarrow` C library is an alternative the C++ runtime can target with the
same generated schema. Arrow is optional in every runtime; the columnar
protobuf and columnar JSON forms need no dependency.

### Moving tables around, end to end

| Hop | Mechanism |
|---|---|
| Implementation → controller (same host, in process) | Arrow record batches in memory, zero copy |
| Controller → client over gRPC | `stream table[T]`, one batch per message |
| Controller → client over HTTP | `application/vnd.apache.arrow.stream` body, or columnar JSON when asked |
| Controller → many consumers over Redis | One `XADD` per batch on a results stream; consumers read with `XREAD`/consumer groups |
| Client → analysis | `.to_arrow()` then pandas, polars, DuckDB; Parquet via Arrow for at-rest storage |
| Client → file | `xapic`-independent: Arrow IPC file for fidelity, columnar JSON for readability, Parquet for archives |

### Consequences for OTG and InfraGraph

- OTG `Metrics.Response` and `States.Response` become unions of `table[T]`
  variants; a `list[T]` form can be kept alongside for existing clients during
  migration, while new clients get chunking and DataFrames for free.

- InfraGraph's expanded graph becomes two tables, `table[Node]` and
  `table[GraphEdge]`, returned by `get_graph` and accepted by `set_graph`, which
  also fixes the lossy round trip noted in PR #88. `networkx` import is a
  one-liner from Arrow.

## 4. Additions to earlier documents

- Roadmap: Phase 3 to 5 each include the three transports and the three codecs
  for that language; a new conformance group covers codec round trips and
  table encodings. New decisions D10 to D12.

- Language: `table[T]` collection, `@idempotent`, `@chunk`,
  `@redis(stream="...")` optional override; `stream` semantics clarified.

- Generators: runtime `Codec` and `Transport` interfaces; Arrow schema
  emitter; columnar protobuf messages; Redis envelope dogfooded from
  `xapi.transport.redis`.

- Critique: OTG B11 (row-oriented metrics) and InfraGraph C13 (graph as a
  string) are added with these fixes.
