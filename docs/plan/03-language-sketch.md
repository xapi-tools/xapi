# 03 — Language sketch (xapi v0.1)

This is a proposal to react to, not a final grammar. Everything here is meant to
survive into `spec/grammar.ebnf` and `spec/semantics.md` in Phase 1 unless a
decision in `00-roadmap.md` changes it.

Design rules:

1. One line per field. Name, type, constraints, default, annotations, in that
   order, left to right.
2. Everything that is a rule is a token the compiler checks. Prose is for
   meaning, never for constraints.
3. Nothing is numbered by hand.
4. Source order is documentation order. Wire order comes from the lockfile.

## 1. Files, packages, imports

```xapi
// otg/flow/flow.xapi
package otg.flow

use otg.common.{Named}
use otg.flow.headers.*
```

- One package per directory. A package may span several files.
- Names resolve within the package, then through `use`. Fully qualified
  `otg.port.Port` always works.
- A manifest `xapi.toml` at the repository root names the project, its version
  and its dependencies (other xapi projects, pinned by version).

## 2. Documentation

```xapi
/// A stream of packets from a transmit endpoint to one or more receive endpoints.
/// Headers are listed outermost first.
model Flow { … }
```

`///` is required on every model, union, enum, mixin, service, rpc, field,
variant and enum value. Missing docs are an error, not a warning. The first
sentence is the summary used by generators for short docstrings.

## 3. Scalars

| xapi | Meaning | Proto | Python | Go | C++ |
|---|---|---|---|---|---|
| `bool` | | `bool` | `bool` | `bool` | `bool` |
| `i8 i16 i32 i64` | signed, range-checked | `int32`/`int64` | `int` | `int8…int64` | `int8_t…int64_t` |
| `u8 u16 u32 u64` | unsigned, range-checked | `uint32`/`uint64` | `int` | `uint8…uint64` | `uint8_t…uint64_t` |
| `f32 f64` | | `float`/`double` | `float` | `float32/64` | `float/double` |
| `string` | UTF-8 | `string` | `str` | `string` | `std::string` |
| `bytes` | | `bytes` | `bytes` | `[]byte` | `std::vector<std::byte>` |
| `mac` | `xx:xx:xx:xx:xx:xx` | `string` | `str` | `net.HardwareAddr` | `xapi::Mac` |
| `ipv4` `ipv6` `ip` | textual address | `string` | `ipaddress.*` | `netip.Addr` | `xapi::IpAddr` |
| `cidr` | address/prefix | `string` | `ipaddress.*Network` | `netip.Prefix` | `xapi::Cidr` |
| `hex` | `0x…` or bare hex string | `string` | `str` | `string` | `std::string` |
| `uuid` | RFC 4122 | `string` | `uuid.UUID` | `uuid.UUID` | `xapi::Uuid` |
| `duration` | signed nanoseconds | `google.protobuf.Duration` | `timedelta` | `time.Duration` | `std::chrono::nanoseconds` |
| `timestamp` | UTC instant | `google.protobuf.Timestamp` | `datetime` | `time.Time` | `std::chrono::sys_time` |
| `json` | opaque structured value | `google.protobuf.Value` | `Any` (`dict`/`list`/…) | `json.RawMessage` | `nlohmann::json` |
| `key` | `string`, required, unique in containing list | `string` | `str` | `string` | `std::string` |
| `key global` | as `key`, unique in the whole document | | | | |
| `unit` | payload-less (only in unions) | empty message | marker class | empty struct | `std::monostate` |

Formatted scalars are validated on `normalize()`/`validate()` and on decode.
Their canonical string form is defined by the language (lower-case hex MAC,
RFC 5952 IPv6), so round trips are stable across languages.

## 4. Models and fields

```xapi
/// A test port.
model Port with Named {
  /// Where the traffic generator is reached, for example `10.0.0.1;1;2`.
  location: string
}

/// An emulated Ethernet interface.
model Ethernet with EthernetBase {
  connection:     Connection!
  ipv4_addresses: [Ipv4]
  ipv6_addresses: [Ipv6]
  dhcpv4_clients: [Dhcpv4Client]
  dhcpv6_clients: [Dhcpv6Client]

  check connection is simulated_link || mac != null
      : "mac is required unless the connection is a simulated link"
}

/// Fields shared by every Ethernet-like interface.
mixin EthernetBase with Named {
  mac:   mac
  mtu:   u16 {64..9216} = 1500   @unit(bytes)
  vlans: [Vlan]
}
```

Field syntax: `name: Type[!] [{constraints}] [= default] [@annotations…]`.

- `!` marks a required field. A required field cannot have a default (lint).
- `[T]` is a list. `{K: V}` is a map with `K` in `string`, integer types or an
  enum. Lists of lists are allowed; maps of lists are allowed; maps as union
  payloads are allowed (the proto backend wraps them).
- Fields are optional by default and have presence. "Unset" and "set to the
  default" are distinguishable until `normalize()` is called.
- `mixin` contributes fields to the including type. The including type owns
  their ids. A mixin may include other mixins. No diamond conflicts: duplicate
  field names across mixins are an error.
- `check <expr> : "message"` is a cross-field rule (section 9).

### Constraints

Inside braces, comma-separated. Applicable set depends on the type.

| Constraint | Applies to | Example |
|---|---|---|
| `lo..hi`, `lo..`, `..hi` | integers, floats, `duration` | `{64..9216}`, `{1..}` |
| `len lo..hi` | string, bytes, lists, maps | `{len 1..64}`, `{len 1..}` |
| `pattern /re/` | string | `{pattern /^[a-z][a-z0-9_]*$/}` |
| `unique` | lists of scalars or of models with a `key` | `{unique}` |
| `in [a, b, c]` | scalars | `{in [1, 2, 4, 8]}` |
| `multiple_of n` | integers | `{multiple_of 8}` |

Sugar: a bare range on a list constrains its length (`[Header]{1..}` is
`{len 1..}`), because that is what authors mean nine times out of ten.

### Defaults

```xapi
mtu:      u16 = 1500
state:    State = up
size:     Size = fixed(64)             // union default: variant and payload
dst:      AutoPattern<mac> = "00:00:00:00:00:00"   // literal selects `value`
packet:   [Header]{1..} = [ethernet]   // list default of unit-ish variants
metrics:  Metrics = {}                 // present with its own defaults
```

Defaults are data in the IR. Generators do not bake them into constructors;
they apply them in `normalize()`, which also runs before serialisation if the
caller asks for a "complete" document.

## 5. Enums

```xapi
/// Administrative state of a link.
enum State = up { up, down }

/// Well-known EtherType values.
enum EtherType: u16 {
  ipv4 = 0x0800
  ipv6 = 0x86DD
  arp  = 0x0806
  vlan = 0x8100
}
```

- Scoped: `State.up`. The proto backend emits `STATE_UP` style names and a
  zero `UNSPECIFIED` sentinel that xapi code never sees.
- Closed: unknown values fail decode with a structured error.
- Optional backing type (`: u16`) makes a valued enum; such an enum can be used
  in `pattern<u16 | EtherType>` so a field accepts either raw numbers or names.
- Default variant after `=`. Doc comments per value are required.

## 6. Unions

```xapi
/// One protocol header. Exactly one variant is present.
union Header = ethernet {
  custom:                Custom
  ethernet:              Ethernet
  vlan:                  Vlan
  vxlan:                 Vxlan
  ipv4:                  Ipv4
  ipv6:                  Ipv6
  pfcpause:              PfcPause
  ethernetpause:         EthernetPause
  tcp:                   Tcp
  udp:                   Udp
  gre:                   Gre
  gtpv1:                 Gtpv1
  gtpv2:                 Gtpv2
  arp:                   Arp
  icmp:                  Icmp
  icmpv6:                Icmpv6
  ppp:                   Ppp
  igmpv1:                Igmpv1
  mpls:                  Mpls
  snmpv2c:               Snmpv2c
  rsvp:                  Rsvp
  macsec:                Macsec
  lacp:                  Lacp
  ipv6_extension_header: Ipv6ExtensionHeader
  cfm:                   Cfm
}
```

28 lines against 140 in `header.yaml`, with the same information plus an
exclusivity guarantee.

```xapi
/// How an emulated interface attaches to the topology.
union Connection {
  port:           ref<Port>
  lag:            ref<Lag>
  vxlan:          ref<VxlanV4Tunnel | VxlanV6Tunnel>
  simulated_link: SimulatedLink
}

/// Packet size.
union Size = fixed(64) {
  fixed:        u32 {64..65535}
  increment:    SizeIncrement
  random:       SizeRandom
  weight_pairs: SizeWeightPairs
}

/// Where a value comes from.
union Source {
  auto                      // unit variant, no payload
  value: u32
}
```

- Variants may carry any type, including scalars, lists, maps, refs, and `unit`.
- `@tagged` on a union emits an explicit discriminator field named `choice` on
  the wire (needed for OTG wire compatibility). Default is untagged: the present
  key is the discriminator, matching protobuf `oneof` and protojson.
- Target idioms: Python has a `.kind` property and typed accessors; Go uses a
  sealed interface with one struct per variant; C++ uses `std::variant`.

## 7. References and keys

```xapi
model Port with Named { … }          // Named provides `name: key`

model TxRxPort {
  tx:  ref<Port>!
  rx:  [ref<Port>]{1..}
}
```

- `ref<T>` is a string on the wire whose value must equal the `key` of some `T`
  reachable from the document root. `T` must have exactly one `key` field.
- `ref<A | B>` resolves against any of the listed types.
- Resolution scope is the document root by default. `ref<T in parent>` limits
  resolution to the nearest enclosing instance of the referenced container
  (for example a component name resolving within its own device).
- Generated code exposes refs as a distinct type (`PortRef` in Go,
  `Ref[Port]` in Python, `xapi::Ref<Port>` in C++) with a `resolve(doc)` helper.
- A `key` field is required and unique within its containing list. `key global`
  is unique across the whole document. Uniqueness is checked by `validate()`.

## 8. Generics

```xapi
/// A fixed value, a list of values, or a counter.
union Pattern<T> = value {
  value:     T
  values:    [T]{1..}
  increment: Counter<T>
  decrement: Counter<T>
}

/// A pattern that may also be auto-assigned by the implementation.
union AutoPattern<T> with Pattern<T> {
  auto
}

/// A pattern whose values can be tagged in metrics.
model Tagged<T> {
  pattern:     Pattern<T>!
  metric_tags: [MetricTag]
}

/// An arithmetic progression.
model Counter<T> {
  start: T!
  step:  T = 1
  count: u32 {1..} = 1
}
```

- Type parameters are instantiated at compile time (monomorphised). Instantiated
  names are deterministic: `Pattern<mac>` becomes `Pattern_Mac` in proto and
  `PatternMac` in Go unless `@proto_name("PatternFlowEthernetDst")` pins a
  legacy name.
- A `with` on a union adds the variants of another union (union mixin).
- Constraints can reference `T` where the type permits (`T {1..}` is allowed
  only when `T` is instantiated with a numeric type; the compiler checks at
  instantiation).

This single feature removes the `x-field-pattern` meta-schema and hundreds of
generated OTG messages.

## 9. Cross-field rules (`check`)

```xapi
model Counter<T> {
  start: T!
  step:  T = 1
  count: u32 {1..} = 1
  check step != 0 : "step must be non-zero"
}

model Ethernet with EthernetBase {
  …
  check connection is simulated_link || mac != null
      : "mac is required unless the connection is a simulated link"
}
```

Expression subset (deliberately small, compiled to native code and to CEL):

- field paths: `a.b.c`, `list[i]`, `len(x)`, `x != null`
- union tests: `u is variant`, `u.variant` (payload)
- comparisons `== != < <= > >=`, boolean `&& || !`, `in [..]`
- literals of all scalar types

No function calls beyond `len`, no quantifiers in v0.1 (`all`/`any` over lists
is a v0.2 candidate).

## 10. Annotations

Annotations are declared, not ad hoc. v0.1 built-ins:

| Annotation | On | Meaning |
|---|---|---|
| `@deprecated("use x", since="1.30")` | any declaration | Emits deprecation in every target; lint warns on use |
| `@status(under_review)` / `@status(obsolete)` | any declaration | Lifecycle, same vocabulary as OTG |
| `@since("1.2")` | any declaration | Version introduced, for docs and capability tooling |
| `@unit(bps)` | numeric fields | Carried into docs, IR, and generated helper names |
| `@id(n)` | fields, variants, enum values | Pin a wire id (escape hatch; the lockfile is preferred) |
| `@proto_name("X")`, `@json_name("x")` | declarations, fields | Wire-name overrides for compatibility |
| `@tagged` | unions | Emit an explicit `choice` discriminator |
| `@http(POST "/config")` | rpc | HTTP binding |
| `@schema("otg.capabilities.v1")` | `json` fields | Declares what the opaque value must conform to |
| `@known(EtherType)` | numeric fields and patterns | Associates a valued enum as a set of named constants |

Custom annotations are declared with `annotation name(params…) on targets` and
appear in the IR; generators ignore the ones they do not know.

## 11. Services

```xapi
/// Open Traffic Generator control API.
service Otg {
  errors Error                                     // default error type

  /// Replace the configuration. Returns warnings about implicit changes.
  rpc set_config(Config) -> Warnings               @http(POST "/config")
  /// Current configuration as the implementation holds it.
  rpc get_config() -> Config                       @http(GET "/config")
  /// Merge a partial configuration.
  rpc update_config(ConfigUpdate) -> Warnings      @http(PATCH "/config")
  rpc set_control_state(ControlState) -> Warnings  @http(POST "/control/state")
  rpc get_metrics(MetricsRequest) -> MetricsResponse
  /// Captured packets, streamed as they become available.
  rpc get_capture(CaptureRequest) -> stream CaptureChunk
}

/// Structured failure.
error Error {
  code:  ErrorCode!
  items: [ErrorItem]{1..}
}

model ErrorItem {
  /// JSON pointer into the request, for example `/flows/0/size/fixed`.
  path:    string
  message: string!
}
```

- `rpc name(Request?) -> Response [throws Err]`. Omitting `throws` uses the
  service default. `-> unit` for no response body.
- `stream` on the request or response side means a sequence of messages.
- Without `@http`, the default binding is `POST /<service>/<rpc>` with a JSON
  body; gRPC is always generated.
- Backends: proto emits `service` and `google.api.http`; OpenAPI emits paths;
  SDKs emit a client with one method per rpc returning `(Response, error)` in
  Go, raising a typed exception in Python, and returning
  `xapi::Result<Response, Error>` in C++.

## 12. Wire ids and evolution

- `xapic lock` assigns ids to new fields, variants and enum values and writes
  `xapi.lock`. `xapic check` fails if the lockfile is stale, so CI enforces it.
- Deleting a declaration moves its id into the type's reserved set.
- Renaming is `@renamed_from("old")`, which keeps the id and emits aliases where
  the target can (JSON decode accepts both; proto keeps the number).
- Changing a field's type is a wire break unless the proto encoding is
  identical; `xapic check --against <git-ref>` reports breaks the way
  `buf breaking` does, but at the xapi level.

## 13. Side by side

OTG `Ethernet.Connection` today (abridged; roughly 40 lines in `device/ethernet.yaml`):

```yaml
Ethernet.Connection:
  type: object
  properties:
    choice:
      type: string
      x-enum:
        port_name: { x-field-uid: 1 }
        lag_name: { x-field-uid: 2 }
        vxlan_name: { x-field-uid: 3 }
        simulated_link: { x-field-uid: 4 }
      x-field-uid: 1
    port_name:
      description: Name of the port that the interface is connected to.
      type: string
      x-constraint: [ /components/schemas/Port/properties/name ]
      x-field-uid: 2
    lag_name: …
    vxlan_name: …
    simulated_link: …
```

xapi:

```xapi
/// How an emulated interface attaches to the topology.
union Connection {
  /// The test port the interface is connected to.
  port:           ref<Port>
  /// The LAG the interface is connected to.
  lag:            ref<Lag>
  /// The VXLAN tunnel the interface rides on.
  vxlan:          ref<VxlanV4Tunnel | VxlanV6Tunnel>
  /// A link simulated between two emulated interfaces.
  simulated_link: SimulatedLink
}
```

InfraGraph `Infrastructure.Edge` and `Device.Edge` today: two types, two
endpoint types, string endpoints with slice syntax, about 130 lines. xapi:

```xapi
/// Connects two endpoints with a link using a wiring scheme.
model Edge<E> {
  ep1:    E!
  ep2:    E!
  /// How ep1 instances are wired to ep2 instances.
  scheme: Scheme = one2one
  link:   ref<Link>!
}

/// Wiring scheme between two expanded endpoint sets.
enum Scheme = one2one {
  /// ep1[i] connects to ep2[i]; the sets must have equal size.
  one2one
  /// Every ep1 connects to every ep2.
  many2many
  /// ep1[i] connects to ep2[(i+1) mod n].
  ring
}

/// A contiguous index range, half-open like Python slices.
model Slice { start: u32 = 0, stop: u32 }

/// A component (or range of components) inside an instance (or range of instances).
model InstanceEndpoint {
  instance:        ref<Instance>!
  index:           Slice
  component:       ref<Component in Device>!
  component_index: Slice
}

model Infrastructure with Named {
  description: string
  devices:     [Device]
  links:       [Link]
  instances:   [Instance]
  edges:       [Edge<InstanceEndpoint>]
}
```
