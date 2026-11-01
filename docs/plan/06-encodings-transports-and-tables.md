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
| `ref<T>` | string |
| `Table<T>` | columnar object (section 3), or row array with the `rows` codec option |

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
- Tables: a `stream Table<T>` rpc sends one record batch per message.

### Redis

Redis is used as a message bus, in three patterns, all defined on top of
one envelope type written in xapi itself (dogfooding):

```xapi
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

### The `table` declaration

```xapi
/// Per-flow traffic metrics.
table FlowMetric {
  name:            ref<Flow> key
  transmit:        TransmitState
  frames_tx:       u64 @unit(frames)
  frames_rx:       u64 @unit(frames)
  bytes_tx:        u64 @unit(bytes)
  bytes_rx:        u64 @unit(bytes)
  frames_tx_rate:  f64 @unit(fps)
  loss:            f64 {0..100} @unit(percent)
  latency:         LatencyMetric          // struct column
  tagged:          [MetricTag]            // list column
}

service Otg {
  rpc get_metrics(MetricsRequest) -> stream Table<FlowMetric>
}
```

A `table` is a model with restrictions that guarantee a columnar layout:

- Column types: scalars, enums, refs, `duration`, `timestamp`, models whose
  fields are themselves column-legal (struct columns), lists of column-legal
  types. No unions, no `json`, no maps in v1 (Arrow can do all three; they are
  deferred, not refused).
- Columns are nullable unless `!`; `key` columns are non-null and unique.
- `Table<T>` is a built-in generic container usable only where `T` is a
  `table`. It is the type of rpc results and of fields (`metrics: Table<FlowMetric>`).

### Encodings of `Table<T>`

| Encoding | Shape | When |
|---|---|---|
| **Arrow IPC stream** (primary) | Record batches with a schema derived mechanically from the table declaration | Anything above a few thousand rows; all `stream Table<T>` rpcs by default; Redis and HTTP and gRPC alike |
| **Columnar JSON** | `{"schema": "otg.FlowMetric", "rows": 3, "columns": {"name": [...], "frames_tx": [...]}}`, nulls as `null`, struct columns as nested columnar objects | Human-readable, automation-friendly, dependency-free; default for the JSON codec |
| **Row JSON** | `[{"name": "f1", ...}, ...]` | Compatibility with existing OTG consumers; codec option `rows` |
| **Columnar protobuf** | Generated message with one `repeated` field per column plus a null bitmap per nullable column | gRPC without an Arrow dependency; still 10 to 50 times fewer allocations than repeated row messages |

The Arrow mapping (xapi → Arrow): `bool`→`bool`; `i8..i64`/`u8..u64`→ matching
width ints; `f32`/`f64`→`float`/`double`; `string`→`utf8` (or `large_utf8`
above 2 GB per batch); `bytes`→`binary`; `mac`→`fixed_size_binary(6)`;
`ipv4`→`fixed_size_binary(4)`; `ipv6`→`fixed_size_binary(16)`;
`ip`/`cidr`/`uuid`/`hex`→`utf8` (uuid as `fixed_size_binary(16)` is a codec
option); enum→`dictionary<int32, utf8>`; `ref<T>`→`utf8` with field
metadata `xapi.ref=T`; `duration`→`duration(ns)`; `timestamp`→`timestamp(ns, UTC)`;
model→`struct`; `[T]`→`list<T>`. Table-level metadata carries the xapi type
name and project version so a consumer can validate without the IR.

Batch size default 64k rows or 4 MB, whichever first; body compression
`zstd` on by default for Redis and HTTP, off for gRPC (which compresses
itself).

### Generated table API

Every language gets a `Table` type with the same operations:

| Operation | Python | Go | C++ |
|---|---|---|---|
| build | `FlowMetricTable.from_rows([...])`, `.append(row)` | `NewFlowMetricTable()`, `.Append(row)` | `FlowMetricTable::from_rows`, `.append` |
| read | `.rows()` iterator of typed rows, `.column("frames_tx")` typed array | `.Rows()`, `.FramesTx() []uint64` | `.rows()`, `.frames_tx()` span |
| validate | `.validate()` (keys unique, ranges, refs) | `.Validate()` | `.validate()` |
| Arrow | `.to_arrow()` → `pyarrow.Table`, `.from_arrow()` | `.ToArrow()` → `arrow.Record`, `.FromArrow()` | `to_arrow()` → `std::shared_ptr<arrow::Table>` |
| frames | `.to_pandas()`, `.to_polars()` (via Arrow, zero-copy where possible) | — | — |
| JSON | `.to_json(columnar=True)` | `MarshalJSON` columnar | `to_json()` |

Python and C++ Arrow support is mature (`pyarrow`, Arrow C++). For Go,
`github.com/apache/arrow-go` is used; for very small C++ deployments the
`nanoarrow` C library is an alternative the C++ runtime can target with the
same generated schema.

### Moving tables around, end to end

| Hop | Mechanism |
|---|---|
| Implementation → controller (same host, in process) | Arrow record batches in memory, zero copy |
| Controller → client over gRPC | `stream Table<T>`, one batch per message |
| Controller → client over HTTP | `application/vnd.apache.arrow.stream` body, or columnar JSON when asked |
| Controller → many consumers over Redis | One `XADD` per batch on a results stream; consumers read with `XREAD`/consumer groups |
| Client → analysis | `.to_arrow()` then pandas, polars, DuckDB; Parquet via Arrow for at-rest storage |
| Client → file | `xapic`-independent: Arrow IPC file for fidelity, columnar JSON for readability, Parquet for archives |

### Consequences for OTG and InfraGraph

- OTG `Metrics.Response`, `States.Response` and capture metadata become
  `Table<...>` declarations; the row-JSON codec option keeps existing clients
  working while new clients get streaming and DataFrames for free.
- InfraGraph's expanded graph becomes two tables, `Table<Node>` and
  `Table<GraphEdge>`, returned by `get_graph` and accepted by `set_graph`, which
  also fixes the lossy round trip noted in PR #88. `networkx` import is a
  one-liner from Arrow.

## 4. Additions to earlier documents

- Roadmap: Phase 3 to 5 each include the three transports and the three codecs
  for that language; a new conformance group covers codec round trips and
  table encodings. New decisions D10 to D12.
- Language: `table` declaration, `Table<T>` built-in, `@idempotent`,
  `@redis(stream="...")` optional override; `stream` semantics clarified.
- Generators: runtime `Codec` and `Transport` interfaces; Arrow schema
  emitter; columnar protobuf messages; Redis envelope dogfooded from
  `xapi.transport.redis`.
- Critique: OTG B11 (row-oriented metrics) and InfraGraph C13 (graph as a
  string) are added with these fixes.
