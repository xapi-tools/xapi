# 04 — xapic and the generators

## Pipeline

```
*.xapi ──lex/parse──▶ AST ──resolve──▶ typed AST ──check──▶ IR ──▶ backends
                                           │                  │
                                      xapi.lock ◀──assign ids─┘
```

1. **Lexer and parser.** Hand-written recursive descent in Go. Error recovery
   per declaration so one typo yields one diagnostic, not a cascade.

2. **Resolve.** Packages, `use`, type names, generic instantiation, mixin
   flattening. Produces a closed set of concrete types.

3. **Check.** Type rules (constraint applicability, default type matches field
   type, `ref<T>` target has exactly one `key`, `check` expressions type-check,
   required fields have no default, docs present, annotation schemas).

4. **Lock.** Load `xapi.lock`; assign ids to new elements; move deleted ids to
   reserved; fail in `--frozen` mode if anything changed (CI).

5. **IR.** Canonical JSON (`xapic ir`). Everything downstream consumes the IR,
   including the generators inside `xapic` itself. This guarantees external
   tools see exactly what the generators see.

6. **Backends.** Templates (`text/template`) plus a formatter pass:
   `buf format`, `black`, `gofmt`, `clang-format`. Output is committed to the
   consuming repo, like protoc output, and is reproducible byte for byte.

Generated code is pure data types plus thin clients and handler interfaces.
Everything that touches bytes or sockets lives in the hand-written runtime
per language, behind two interfaces:

```
Codec:      encode(value) -> bytes, decode(bytes, type) -> value      json | protobuf | arrow
Transport:  call(rpc, request bytes) -> response bytes, stream_call, serve(dispatcher)   http | grpc | redis
```

The dispatcher (generated once per service) maps an incoming rpc name to the
handler method, decodes with the negotiated codec, runs `validate()` when the
server asks for it, and encodes the result or the `Error`. See
`06-encodings-transports-and-tables.md` for the codec and transport specs.

## CLI

```
xapic check   [--frozen] [--against <git-ref>]   # parse, check, lockfile, breaking changes
xapic fmt     [--check]                          # canonical formatting
xapic lock                                       # update xapi.lock
xapic ir      -o build/ir.json                   # canonical IR
xapic gen     proto|openapi|arrow|jsonschema|python|go|cpp|docs -o <dir>
xapic lint                                       # model-guide rules (opt-in sets)
xapic import  openapiart <openapi.yaml|dir> -o <dir>   # migration
```

## Lockfile

```toml
# xapi.lock — written by `xapic lock`; commit it; do not edit by hand.
version = 1

[otg.flow.Flow]
fields = { name = 1, tx_rx = 2, packet = 3, size = 4, rate = 5, duration = 6, metrics = 7, egress_packet = 9 }
reserved = [8]

[otg.flow.Header]
variants = { custom = 1, ethernet = 2, vlan = 3, … }

[otg.device.State]
values = { up = 1, down = 2 }
```

- Keyed by fully qualified type; entries sorted; one line per type so merge
  conflicts are local and meaningful.

- New elements get `max(existing, reserved) + 1`.

- Generic instantiations are keyed by their mangled name
  (`otg.flow.Pattern<mac>`), so each instantiation keeps its own stable numbers.

- The importer seeds the lockfile from `x-field-uid` and
  `x-reserved-field-uids`, which is how wire compatibility with existing
  OTG and InfraGraph protobuf is achieved.

## Protobuf backend

| xapi | proto3 output |
|---|---|
| `package a.b` | `package a.b;` one file per xapi package, `go_package`, `java_package` options derived |
| `model` | `message`; mixin fields inlined |
| optional scalar field | `optional T f = n;` |
| required field | plain field plus `buf.validate.field.required = true` when `--validate` is on |
| `[T]` | `repeated` (wrapped in a synthesised message when inside a `oneof`) |
| `{K: V}` | `map<K,V>` (wrapped inside a `oneof`) |
| `union` | `oneof` named after the union; `@tagged` adds `Choice choice = 1;` enum, preserving OTG layout |
| unit variant | synthesised empty message `Foo.Auto {}` |
| `enum` | nested or top-level enum with `_UNSPECIFIED = 0` and value prefix |
| `i8/i16/u8/u16` | `int32`/`uint32` with range annotations and a comment |
| `mac ipv4 …` | `string` with format comment and `buf.validate` pattern |
| `duration timestamp json` | well-known types |
| `ref<T>` | `string` with a leading comment `// ref<T>` |
| constraints | comments always; `buf.validate` options with `--validate` |
| `///` docs | leading comments on every element |
| reserved ids | `reserved n, m;` |
| `service` | `service`, `stream` keywords, `google.api.http` annotations from `@http` |
| `table T` / `Table<T>` | columnar message `TTable { repeated C col; bytes col_nulls; … }` plus `TTableChunk` for streams; alternatively `bytes arrow_ipc` when the Arrow codec is negotiated |

Compatibility check in CI: `buf breaking --against` the published proto of the
imported model.

## OpenAPI backend

OpenAPI 3.1 for documentation (Redoc) and for tools that still want it. Unions
become `oneOf` with a `discriminator` when `@tagged`, or an object with
`minProperties: 1, maxProperties: 1` otherwise. `x-field-uid` is emitted for
anyone bridging back into openapiart. This backend is a convenience, not a
source of truth.

## SDK contract (all languages)

Every generated SDK offers the same operations with language-native spelling:

| Operation | Python | Go | C++ |
|---|---|---|---|
| construct | keyword args | struct literal | aggregate / designated init |
| required-ness | `validate()` | `Validate() error` | `validate() -> std::expected<void, xapi::Errors>` |
| defaults | `normalize()` | `Normalize()` | `normalize()` |
| JSON | `to_json()`, `from_json()` | `MarshalJSON`, `UnmarshalJSON` | `to_json()`, `from_json()` (nlohmann) |
| YAML | via JSON | via JSON | via JSON |
| Protobuf | `to_proto()`, `from_proto()` | `ToProto()`, `FromProto()` | `to_proto()`, `from_proto()` |
| refs | `Ref[Port]` with `.resolve(doc)` | `PortRef` string type with `Resolve(doc)` | `xapi::Ref<Port>` |
| unions | `.kind` plus typed accessors | sealed interface, one struct per variant | `std::variant` plus `kind()` |
| errors | `xapi.ValidationError` with `.items[{path, code, message}]` | `*xapi.ValidationError` implementing `error`, `Unwrap` | `xapi::ValidationError` |
| clients | `OtgClient(transport)`; sync and async | `OtgClient` with `context.Context` first | `xapi::otg::Client` |
| transports | `HttpTransport` (httpx), `GrpcTransport` (grpcio), `RedisTransport` (redis-py) | `xapihttp`, `xapigrpc`, `xapiredis` packages | gRPC native; HTTP and Redis behind CMake options |
| codecs | `JsonCodec`, `ProtoCodec`, `ArrowCodec` | same | same |
| tables | `FlowMetricTable` with `.rows()`, `.column()`, `.to_arrow()`, `.to_pandas()` | `FlowMetricTable` with typed column accessors, `ToArrow()` | `FlowMetricTable` with spans, `to_arrow()` |
| servers | `OtgHandler` protocol + `serve(handler, transport)` | `OtgHandler` interface + `Serve` | `OtgHandler` abstract class + `serve` |

Validation error codes are a closed enum shared by all runtimes (`required`,
`range`, `length`, `pattern`, `unique`, `unknown_ref`, `union_multiple`,
`union_none`, `enum_unknown`, `check_failed`, `format`). Paths are JSON
pointers. This is what the conformance suite asserts.

## Python backend

```python
@dataclass(slots=True)
class Ethernet(EthernetBase):
    """An emulated Ethernet interface."""
    connection: Connection                     # required: no default
    ipv4_addresses: list[Ipv4] = field(default_factory=list)
    ...

class Connection:
    """How an emulated interface attaches to the topology."""
    kind: Literal["port", "lag", "vxlan", "simulated_link"]
    port: Ref[Port] | None
    ...
    @classmethod
    def port_(cls, ref: Ref[Port] | str) -> "Connection": ...
```

- Plain classes with `__slots__`, full type hints, `py.typed`.

- Unions are small classes with a `kind` and exactly one populated payload;
  constructors per variant; pattern matching via `match conn.kind:`.

- No runtime dependency beyond `xapi-runtime` (which depends on nothing for
  JSON; protobuf and grpc are optional extras).

- Sync and async clients (`httpx` for HTTP, `grpcio` or `grpclib` for gRPC).

## Go backend

```go
// Ethernet is an emulated Ethernet interface.
type Ethernet struct {
    Name          string       `json:"name"`
    Mac           *net.HardwareAddr `json:"mac,omitempty"`
    Mtu           *uint16      `json:"mtu,omitempty"`     // default 1500, 64..9216, unit bytes
    Connection    Connection   `json:"connection"`        // required
    Ipv4Addresses []Ipv4       `json:"ipv4_addresses,omitempty"`
}

// Connection is how an emulated interface attaches to the topology.
type Connection interface{ isConnection() }
type ConnectionPort struct{ Port PortRef }
type ConnectionLag struct{ Lag LagRef }
...
func (e *Ethernet) Validate() error
func (e *Ethernet) Normalize()
```

- One Go package per xapi package. `go vet`, `staticcheck` and `golangci-lint`
  clean is a CI gate.

- Optional scalars are pointers; helpers `xapi.Ptr(v)` and `xapi.Or(p, def)`.

- Enums are `type State string` with constants and `Valid()`.

- Clients take `context.Context` first and return `(T, error)`; errors wrap
  `*xapi.ValidationError` or `*xapi.RemoteError`.

## C++ backend

```cpp
namespace otg::device {
/// An emulated Ethernet interface.
struct Ethernet {
  std::string name;
  std::optional<xapi::Mac> mac;
  std::optional<uint16_t> mtu;            // default 1500, 64..9216, unit bytes
  Connection connection;                  // required
  std::vector<Ipv4> ipv4_addresses;

  std::expected<void, xapi::Errors> validate() const;
  void normalize();
};
using Connection = std::variant<ConnectionPort, ConnectionLag, ConnectionVxlan, ConnectionSimulatedLink>;
}
```

- C++20, one header and one source per package, CMake package config.

- `nlohmann::json` for JSON, `std::variant` for unions, `std::expected` (or a
  shipped polyfill) for results, protobuf conversion behind a CMake option.

- No exceptions in the generated validation path.

## Importer (`xapic import openapiart`)

Maps openapiart conventions back to language features:

| openapiart | xapi |
|---|---|
| `choice` + `x-enum` + sibling properties | `union` with `@tagged` |
| `x-field-uid`, `x-reserved-field-uids` | lockfile entries |
| `x-include` | `mixin` |
| `x-constraint: [/components/schemas/Port/properties/name]` | `ref<Port>` |
| `x-status: deprecated/under_review/obsolete` | `@deprecated`, `@status(...)` |
| `x-field-pattern` | `Pattern<T>` family with `@proto_name` pinning legacy names |
| `x-constants` | valued `enum` plus `@known` |
| `format: global-unique / list-unique` | `key global` / `key` |
| `minimum/maximum/minLength/maxLength/minItems/pattern` | brace constraints |
| `x-stream` | dropped; noted in a migration report |
| `required: [...]` | `!` |
| `description` | `///` |

The importer emits a report of everything it could not express or that it
suspects is wrong (uid gaps, duplicate descriptions, prose rules containing
`MUST`), which feeds Part E of the critique.

## Testing strategy

- **Golden tests** per backend: fixture `.xapi` in, expected files out,
  diffed byte for byte.

- **Compile tests**: generated Python is imported and `mypy --strict` clean;
  Go is `go build` and `go vet` clean; C++ compiles under GCC and Clang with
  warnings as errors; proto passes `buf lint` and `buf breaking`.

- **Conformance suite** (see 05) runs the same cases through every runtime,
  including codec round trips (JSON ↔ protobuf ↔ Arrow produce identical
  normalised JSON) and table encodings.

- **Interop matrix**: every client language against every server language
  over every transport, with a fixed set of rpcs including a streamed table.

- **End-to-end**: the generated OTG Python and Go clients run a traffic test
  against ixia-c in CI (ports, one flow, metrics), proving real-world
  interoperability with existing implementations.
