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

```kotlin
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

## 2. Documentation and comments

```kotlin
/// A stream of packets from a transmit endpoint to one or more receive endpoints.
/// Headers are listed outermost first.
model Flow {
  // TODO: confirm with vendors whether egress_packet survives review.
  …
}
```

Two syntaxes, two roles:

- `///` is **documentation**. It is data: carried into the IR, emitted as
  docstrings in Python, Go and C++, as comments in the `.proto`, as
  descriptions in OpenAPI and the HTML docs. It is required on every model,
  union, enum, mixin, table, service, rpc, field, variant and enum value;
  a missing `///` is an error. The first sentence is the summary used where
  only a short description fits.

- `//` is a **comment**. It stays in the source file and is never emitted.

Why two syntaxes rather than one:

- The rule "everything is documented" is only enforceable if the checker can
  tell which comment counts. With a single syntax the rule degrades to "some
  comment appears above it", which is how protobuf-generated docs end up full
  of leaked implementation notes.

- Position is not a reliable signal. "A comment directly above a declaration
  is its documentation" breaks silently on a blank line, a commented-out
  field or a note written between the doc and the declaration. An explicit
  marker fails loudly instead.

Why `///` is the documentation marker and not the other way round:

- Mark the one with consequences. A `///` line is published to every SDK
  user; a `//` line is a no-op. The no-op is the safe default, the publish is
  the deliberate act.

- Commenting out code must be harmless. If `//` were documentation,
  `// mtu: u16 {64..9216}` above a field would silently become that field's
  description. With `///` for docs it is simply ignored.

- Every C-family language, protobuf included, treats `//` as text the tooling
  ignores. Reversing that would publish stray notes from any pasted snippet
  and would make editors' colouring misleading.

- The terseness gain of the reversal is one character per documentation
  line; the cost is a rare special spelling that people forget precisely
  because it is rare.

This follows Rust, C#, Dart and Swift, so it reads as intended without
explanation.

## 3. Primitive types (locked)

### Vocabulary

Two pairs of words are used throughout the specification and never mixed:

- **Primitive / derived** describe xapi types. A primitive type is defined by
  the language itself, with its own encoding rule, and cannot be built from
  other xapi types. Every other type is derived from primitives, by
  restriction (aliases) or by composition (declared and parametric types).

- **Native / generated** describe target types. A native type already exists
  in the target language or runtime (`uint16`, `str`, `nlohmann::json`,
  `google.protobuf.Value`). A generated type is emitted by `xapic`.

The two axes are independent: every primitive maps to a native type, and so do
aliases and a few well-known derived types; models, unions, enums and tables
map to generated types.

The word "scalar" is not used: one primitive is not a scalar value.

### The test

A type is primitive only if it passes this test:

> It cannot be expressed as another primitive plus a constraint without
> changing its wire encoding or its natural native type.

Six types pass.

### The six primitives

| Primitive | Values | JSON | Protobuf |
|---|---|---|---|
| `bool` | true, false | `true` / `false` | `bool` |
| `int` | integers | number | varint |
| `float` | IEEE reals | number; `"NaN"`, `"Inf"`, `"-Inf"` as strings | `double` |
| `string` | Unicode text, valid UTF-8 | string | `string` |
| `bytes` | octet sequence | base64 string | `bytes` |
| `json` | any JSON value | the value itself, inline | `google.protobuf.Value` |

Why each is irreducible:

- **`bool`** is not `int {0..1}`. JSON encodes it as a literal, not a number,
  and every language has a distinct boolean type.

- **`int`** is the only integer type. Signedness and width are a range and a
  representation hint respectively, not kinds of value (see aliases below).

- **`float`** is not `int` with a range: its value set contains non-integers.
  Precision (32 versus 64 bit) is a representation hint, not a kind.

- **`string`** is not "bytes with a UTF-8 constraint". JSON carries text
  natively and cannot carry octets, length is counted in characters, and the
  native type is text.

- **`bytes`** is not `[u8]`, which would encode as an array of numbers, and not
  `string`, because the JSON encoding must be base64 and the native type is a
  byte buffer.

- **`json`** is the one open value. Its encoding rule, "the value is its own
  encoding", cannot be derived from models, unions or maps, and its native type
  is the target's dynamic JSON type rather than a generated class. See the
  section below.

Native defaults: `int` is 64-bit signed, `float` is 64-bit. These never change
when a range changes, so generated SDK types are stable.

### `json` in detail

A `json` field carries a value whose shape the model deliberately does not
know.

```kotlin
/// A graph node after expansion.
model Node {
  id:         string key
  type:       NodeType
  /// Attributes attached by annotate_graph; their shape is owned by the caller.
  attributes: json
}
```

On the wire the value is embedded as itself:

```json
{
  "id": "server[0].xpu[1]",
  "type": "xpu",
  "attributes": { "rank": 1, "labels": ["gpu", "hbm3"], "owner": { "team": "ml" } }
}
```

In protobuf the field is `google.protobuf.Value`, which `grpcurl` and gateways
print as ordinary JSON. `Value` stores numbers as doubles; a model that needs
exact integers above 2^53 inside an open value uses `@repr(text)`, which maps
the protobuf side to a `string` holding JSON text while the JSON encoding stays
embedded.

Native types: Python `dict | list | str | int | float | bool | None`, Go
`json.RawMessage`, C++ `nlohmann::json`. Decoding checks well-formedness only;
`@schema(T)` additionally validates the value against a named xapi model or an
external JSON Schema.

Why it is not a `string` alias: a JSON document inside a `string` is encoded
as an escaped string literal, is opaque to tools, cannot be validated by a
regular expression (JSON is not a regular language), and compares as text
rather than as a value. That is InfraGraph's `Annotation.Attribute.value`
today (flaw C2).

Why it is not a map: a map is homogeneous and flat. `{string: string}`
re-creates the stringly-typed problem; `{string: Value}` with a union of
primitives encodes tagged (`"rank": {"int": 1}`) and cannot nest; a recursive
union can represent any JSON value but still encodes tagged and generates a
`Value` class instead of the native dynamic type. `json` is exactly that
recursive union with an untagged encoding and a native representation, which
is what makes it primitive.

Use it only at deliberate extension points: annotations, vendor extras, user
metadata, pass-through payloads, diagnostics on errors. `xapic lint` warns
when a `json` field is used where a declared type would do.

### Representation aliases

`uint`, `i8 i16 i32 i64`, `u8 u16 u32 u64`, `f32 f64` are aliases, not
primitives. Each is a primitive plus an implied range plus a representation
hint, and nothing more:

| Alias | Means | Native | Why it exists |
|---|---|---|---|
| `uint` | `int {0..}` | unsigned 64-bit | counters and sizes read naturally as unsigned in Go, C++ and Arrow |
| `u8 u16 u32 u64` | `int {0..2^n-1}` | unsigned n-bit | packet fields are genuinely n-bit; table columns are n/8 bytes wide |
| `i8 i16 i32 i64` | `int {-2^(n-1)..2^(n-1)-1}` | signed n-bit | same, signed |
| `f32` | `float` | 32-bit | half the memory in a column; interop with existing protobuf `float` fields |
| `f64` | `float` | 64-bit | spelled out for symmetry; identical to `float` |

Rules:

- An alias never changes validation beyond the range it implies. `u16 {64..9216}`
  and `uint {64..9216}` accept exactly the same values; only the generated
  native type differs.

- A range that exceeds the alias is a compile error.

- Protobuf: all integer forms are varints and wire-compatible with each other,
  so the backend uses the narrowest standard proto scalar. `float` versus
  `double` is not wire-compatible, so changing `f32` to `float` is reported as
  a wire break by `xapic check --against`.

- Changing an alias on a field is always reported as an SDK-breaking change.

- Width is never inferred from a range: that would change generated types
  whenever a range crossed a boundary.

### Things that are not primitives

| Earlier draft | Is actually | Where it lives now |
|---|---|---|
| `key`, `key global` | a field modifier on `string` | section 7 (`name: string key`) |
| `unit` | union syntax for a payload-less variant | section 6 |
| `Table<T>` | a collection | section 3b |
| `mac ipv4 ipv6 ip cidr hex uuid` | `string` plus a `format` constraint | constraints (section 4); named via `type` aliases, pending |
| `duration timestamp` | well-known structured values | pending the same discussion |

Examples below still use `mac`, `ipv4` and `duration` as type names. Read them
as placeholders for whatever that discussion decides.

## 3a. Derived types: the two mechanisms

Everything that is not primitive is derived, by one of two mechanisms.

**Restriction** shrinks the value set of an existing type without touching
its encoding or native shape. It produces one kind, the alias (`type`), and
constraints are its vocabulary.

**Composition** builds a new type from parts, and the encoding is computed
from the parts' encodings. It comes in two flavours:

- *Named* composition gives a nominal type: `model` (product), `union` (sum),
  `enum` (closed set of names). Two identical bodies are two different types.
  These are the only types with lockfile ids and evolution rules, because they
  are the only ones with identity.

- *Anonymous* composition gives a structural type written inline from a
  parametric constructor: the collections `[T]`, `{K: V}`, `Table<T>`, the
  reference `ref<T>`, and generic instances `G<T>`. Two identical spellings
  are the same type. They have no ids and are never declared.

Generics sit across the line: `model<T>` is a template produced by named
composition, and each instantiation `G<T>` is turned by the compiler into a
named, nominal type with its own ids and a deterministic name.

Three rules hold for the whole layer:

1. No inline named composition. Every model, union and enum has a name.

2. Presence is a field property, never a type. No `T?`, no `nullable`, no
   absent elements inside collections.

3. Encodings are compositional. The wire form of a derived type is a pure
   function of its kind and its parts. This is what makes aliases free,
   generators small and the conformance suite finite.

## 3b. Collections

### What exists elsewhere

Programming languages offer many collection shapes: list, fixed-size array,
tuple, set, map, ordered map, multimap, deque, queue, stack, linked list,
optional, bag, bitset, matrix, dataframe. Serialisation formats are far more
conservative:

| Format | Collections it can encode |
|---|---|
| JSON, CBOR, MessagePack | array, object |
| Protobuf | `repeated`, `map` |
| Avro | array, map |
| Thrift | list, set, map |
| Smithy | list, map (`set` deprecated in favour of list with `@uniqueItems`) |
| Arrow | list, fixed-size list, map, struct, union, dictionary |

Every format reduces the zoo to an ordered sequence and a keyed lookup, plus
the columnar forms Arrow adds. Thrift shipped `set` as a type; Smithy later
deprecated exactly that in favour of a constraint, which is the precedent xapi
follows.

### What xapi includes

A collection kind earns a place only if it has its own encoding or its own
native shape that cannot be obtained from another kind plus a constraint.
Three pass:

| Collection | Shape | JSON | Protobuf | Native |
|---|---|---|---|---|
| `[T]` | ordered, homogeneous, dense, duplicates allowed | array | `repeated` | list, slice, vector |
| `{K: V}` | unordered, unique keys | object | `map` | dict, map |
| `Table<T>` | ordered rows of a column-legal model, columnar | columnar object | one `repeated` per column | table container with column accessors |

`Table<T>` is a collection and not a variant of `[T]`: the two hold the same
values and differ in encoding (columnar versus array of objects), native shape
(column accessors and zero-copy Arrow versus a vector of structs) and
semantics (per-column nullability, table-wide key uniqueness). Because it is
a collection over any column-legal `model`, there is no separate `table`
declaration; column-legality is checked where `Table<T>` is instantiated, as
with any generic. Details in section 8a.

### What xapi leaves out, and why

Same bytes, different validation (a constraint, at most a `@repr` hint):

- Set: `[T]{unique}`. No format has a set encoding, and element order on the
  wire must be fixed for deterministic documents anyway.

- Fixed-size array: `[T]{len 6..6}`. Arrow fixed-size list and `std::array`
  are representation choices.

- Multimap: `{K: [V]}`. Bag: a plain list.

Better served by a model or a reference:

- Tuple: positional and heterogeneous. Protobuf cannot encode it and positions
  cannot evolve; a model with named fields and ids can.

- Ordered map: JSON objects and protobuf maps are unordered by specification.
  Order plus lookup is a list of keyed models, which is also what `ref<T>`
  resolves against.

- Tree and graph: models with `ref<T>` or recursion.

Not data shapes at all: deque, queue, stack, linked list, ring buffer and
priority queue describe how a program manipulates a sequence, and all
serialise as a list. Optional is a field property. Bitset and flags are
`[Flag]{unique}` or an integer with `@known`.

Deferred: matrix and tensor. Small ones are `[[float]]`; large numeric blocks
belong in a `Table<T>` or, if ever needed, an Arrow tensor behind `@repr`.

### Rules for lists

- Element type: any single type, including another list. Heterogeneity is a
  union element type (`[Header]`), never a mixed list.

- Dense: no absent elements.

- Constraints: `len lo..hi`, `unique`. On a list of models `unique` compares
  the `key` field and requires one; on primitives it compares values.

- Nested lists get a synthesised wrapper message on the protobuf side.

### Rules for maps

- Key type: `string`, `int` or an int alias, or an `enum`. Not `float`,
  `bytes`, `bool` or a model. Non-string keys are strings in JSON; enum keys
  are the value name in both JSON and protobuf, since protobuf forbids enum
  map keys.

- Value type: any single type except another map (deferred). Lists as values
  get a synthesised wrapper on the protobuf side.

- Values are never absent; a missing key is the only absence.

- Unordered. The JSON encoder sorts keys so documents are deterministic.

- Constraint: `len lo..hi`.

### Rules shared by all collections

- **No presence distinction.** An absent collection and an empty one are the
  same value. Protobuf cannot tell them apart for `repeated` and `map`, and
  preserving a difference that half the wire formats drop would make the
  conformance suite lie. `!` on a collection therefore means non-empty, exactly
  `len 1..`, and `xapic fmt` rewrites one into the other.

- Defaults are literals only: `= []`, `= {}`, or a list of enum values or unit
  variants such as `= [ethernet]`.

- Length sugar: a bare range directly after `]` or `}` means `len`, because
  braces attach to the thing immediately to their left and a collection has
  no other numeric constraint.

## 4. Models and fields

```kotlin
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

- `[T]` is a list and `{K: V}` a map; see section 3b for their rules. Both
  may be union payloads (the proto backend wraps them).

- Fields are optional by default and have presence. "Unset" and "set to the
  default" are distinguishable until `normalize()` is called. Collections are
  the exception: absent and empty are the same value (section 3b).

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

```kotlin
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

```kotlin
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

```kotlin
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

```kotlin
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

```kotlin
model Port with Named { … }          // Named provides `name: string key`

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

- `key` is a modifier on a `string` field: required and unique within the
  containing list. `key global` is unique across the whole document.
  Uniqueness is checked by `validate()`.

## 8. Generics

```kotlin
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

## 8a. Tables

```kotlin
/// Per-flow traffic metrics.
table FlowMetric {
  name:       ref<Flow> key
  transmit:   TransmitState
  frames_tx:  u64 @unit(frames)
  frames_rx:  u64 @unit(frames)
  loss:       f64 {0..100} @unit(percent)
  latency:    LatencyMetric          // struct column
  tagged:     [MetricTag]            // list column
}
```

- A `table` is a model restricted to column-legal types: scalars, enums, refs,
  `duration`, `timestamp`, models made of column-legal fields, and lists of
  those. Unions, maps and `json` columns are deferred to v0.2.

- Columns are nullable unless `!`; `key` columns are non-null and unique
  within the table.

- `Table<T>` is the container type for rpc results and fields. Its encodings
  (Arrow IPC, columnar JSON, row JSON, columnar protobuf) and the generated
  table API are specified in `06-encodings-transports-and-tables.md`.

- `stream Table<T>` on an rpc means a sequence of record batches; a bare
  `Table<T>` means one batch set delivered whole.

## 9. Cross-field rules (`check`)

```kotlin
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
| `@http(POST "/config")` | rpc | HTTP binding (gRPC and Redis bindings are derived; `@redis(stream="…")` overrides the stream name) |
| `@idempotent` | rpc | Transports may retry on timeout |
| `@schema("otg.capabilities.v1")` | `json` fields | Declares what the opaque value must conform to |
| `@known(EtherType)` | numeric fields and patterns | Associates a valued enum as a set of named constants |

Custom annotations are declared with `annotation name(params…) on targets` and
appear in the IR; generators ignore the ones they do not know.

## 11. Services

```kotlin
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
  /// Per-flow metrics as a stream of record batches.
  rpc get_metrics(MetricsRequest) -> stream Table<FlowMetric>
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

- Without `@http`, the default binding is `POST /<service>/<rpc>`; gRPC and
  Redis bindings are always derived from the service and rpc names.

- The encoding (JSON, protobuf, Arrow for tables) is chosen by the caller per
  client or per call; it is not part of the declaration.

- Backends: proto emits `service` and `google.api.http`; OpenAPI emits paths;
  SDKs emit a client with one method per rpc returning `(Response, error)` in
  Go, raising a typed exception in Python, and returning
  `xapi::Result<Response, Error>` in C++, plus a handler interface and a
  transport-neutral dispatcher for servers.

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

```kotlin
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

```kotlin
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
