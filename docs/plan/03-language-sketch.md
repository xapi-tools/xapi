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

- **`bytes`** is not `list[u8]`, which would encode as an array of numbers, and not
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
| `Table<T>`, `table` declaration | the collection `table[T]` | sections 3b and 8a |
| `mac ipv4 ipv6 ip cidr hex uuid` | `string` aliases with a pattern and `@repr` | section 4 (pattern); the standard library |
| `duration timestamp` | `int` aliases with `@unit` | the standard library |

Examples below use `mac`, `ipv4` and `duration` as if the standard library
already declared them.

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
  parametric constructor: the collections `list[T]`, `map[K, V]`, `table[T]`,
  the reference `ref[T]`, and generic instances `G[T]`. Two identical
  spellings are the same type. They have no ids and are never declared.

Generics sit across the line: `model Counter[T]` is a template produced by
named composition, and each instantiation `Counter[uint]` is turned by the
compiler into a named, nominal type with its own ids and a deterministic name.

All type application uses one shape, `name[args]`. There are no angle
brackets: `>` is then only ever a comparison operator in `check`
expressions, and the `>>` lexing problem that C++ and Java had to special-case
never arises.

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
| `list[T]` | ordered, homogeneous, dense, duplicates allowed | array of values | `repeated T` | list, slice, vector |
| `map[K, V]` | unordered, unique keys | object of values | `map<K, V>` | dict, map |
| `table[T]` | ordered rows of a flat model, stored by column | object of arrays | generated message, one `repeated` per column | table container with column accessors |

Collections are spelled as words with square brackets rather than as symbols:
the words are self-describing, extensible without inventing bracket
combinations, searchable, free of collisions with the brace constraint
syntax, and familiar from Python (`list[int]`, `dict[str, int]`). The JSON
shape each one mirrors is recorded in the table above rather than in the
syntax.

`list[T]` and `table[T]` hold the same values, an ordered sequence of rows,
and every SDK converts between them. They differ in layout. A list is
row-major: each row is an object or a message, and every key or tag is
repeated per row. A table is column-major: keys appear once per column,
numeric columns are packed, and the native container exposes columns directly
and converts to Arrow and DataFrames without transposing. Use `table[T]` for
anything measured in thousands of rows (metrics, states, expanded graphs),
`list[T]` for configuration, where order and per-item references matter and
rows are few. Rules in section 8a.

### What xapi leaves out, and why

Same bytes, different validation (a constraint, at most a `@repr` hint):

- Set: `list[T] {unique}`. No format has a set encoding, and element order on the
  wire must be fixed for deterministic documents anyway.

- Fixed-size array: `list[T]{6..6}`. Arrow fixed-size list and `std::array`
  are representation choices.

- Multimap: `map[K, list[V]]`. Bag: a plain list.

Better served by a model or a reference:

- Tuple: positional and heterogeneous. Protobuf cannot encode it and positions
  cannot evolve; a model with named fields and ids can.

- Ordered map: JSON objects and protobuf maps are unordered by specification.
  Order plus lookup is a list of keyed models, which is also what `ref[T]`
  resolves against.

- Tree and graph: models with `ref[T]` or recursion.

Not data shapes at all: deque, queue, stack, linked list, ring buffer and
priority queue describe how a program manipulates a sequence, and all
serialise as a list. Optional is a field property. Bitset and flags are
`list[Flag]{unique}` or an integer with `@known`.

Deferred: matrix and tensor. Small ones are `list[list[float]]`; large
numeric blocks belong in a `table[T]` or, if ever needed, an Arrow tensor
behind `@repr`.

### Rules for lists

- Element type: any single type, including another list. Heterogeneity is a
  union element type (`list[Header]`), never a mixed list.

- Dense: no absent elements.

- Constraints: a range (number of elements), `unique` (primitives, aliases,
  enums, refs only; models are unique by `key`).

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

- Constraint: a range (number of entries).

### Rules for tables

- Element type: a flat model only. Its fields must be primitives, aliases,
  enums, refs, or the well-known `duration` and `timestamp`. No nested models,
  lists, maps, unions or `json`: nesting defeats the columnar layout and
  DataFrame interop, and the relational answer (a second table with a `ref`
  column as foreign key) already exists.

- Columns are nullable unless the field is `!` or `key`. A `key` column is
  unique across the whole table.

- Same values as `list[T]`; `rows()` and `columns()` convert in every SDK.

- Encodings, chunking and the generated API are in section 8a.

### Rules shared by all collections

- **No presence distinction.** An absent collection and an empty one are the
  same value. Protobuf cannot tell them apart for `repeated` and `map`, and
  preserving a difference that half the wire formats drop would make the
  conformance suite lie. `!` on a collection therefore means non-empty, exactly
  `{1..}`, and `xapic fmt` rewrites one into the other.

- Defaults are literals only: `= []`, `= {}`, or a list of enum values or unit
  variants such as `= [ethernet]`.

- A range on a collection is its number of entries (`list[Header] {1..}`);
  see the range rule in section 4.

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
  ipv4_addresses: list[Ipv4]
  ipv6_addresses: list[Ipv6]
  dhcpv4_clients: list[Dhcpv4Client]
  dhcpv6_clients: list[Dhcpv6Client]

  check connection is simulated_link || mac != null
      : "mac is required unless the connection is a simulated link"
}

/// Fields shared by every Ethernet-like interface.
mixin EthernetBase with Named {
  mac:   mac
  mtu:   u16 {64..9216} = 1500   @unit(bytes)
  vlans: list[Vlan]
}
```

Field syntax: `name: Type` followed optionally by `!`, `{constraints}`,
`= default` and `@annotations`, in that order.

- `!` marks a required field. A required field cannot have a default (lint).

- `list[T]`, `map[K, V]` and `table[T]` are the collections; see section 3b
  for their rules. All may be union payloads (the proto backend wraps them).

- Fields are optional by default and have presence. "Unset" and "set to the
  default" are distinguishable until `normalize()` is called. Collections are
  the exception: absent and empty are the same value (section 3b).

- `mixin` contributes fields to the including type. The including type owns
  their ids. A mixin may include other mixins. No diamond conflicts: duplicate
  field names across mixins are an error.

- `check <expr> : "message"` is a cross-field rule (section 9).

### Constraints (locked)

Inside braces, comma-separated, all must hold. Three constraints exist.

| Constraint | Applies to | Example |
|---|---|---|
| range: `a..`, `..b`, `a..b`, `a..a`, `a` | every type with a magnitude | `{64..9216}`, `{1..}`, `{6}` |
| `/re/` | string | `{/[a-z][a-z0-9_]*/}` |
| `unique` | lists of primitives, aliases, enums or refs | `{unique}` |

**Range (locked).** Both ends inclusive; an open end is unbounded; `a..a` is
exactly `a`, and a standalone `a` is sugar for `a..a`. A range constrains the type's natural magnitude: the value of
an `int`, `float` or `duration`; the size of a `string` (Unicode scalar
values), `bytes` (octets), list or map (entries). There is no separate
length keyword, because no type has both a value range and a size. A range
must be well-formed at compile time (`a <= b`, within the bounds of the
type's kind). On `float`, a range also rejects NaN and infinities, since
they compare false against any bound.

Whitespace between the type and the brace is free; `xapic fmt` emits one
space: `uint {64..9216}`.

**Pattern (locked).** A regex literal between slashes, no keyword: it is
self-identifying and raw (regex escapes only, `\/` for a literal slash).
Applies to `string` only. Full match: the whole value must match, so authors
write no anchors, and the JSON Schema backend emits `^(?:…)$` because JSON
Schema's `pattern` is a search. RE2 syntax, verified at compile time; inline
flags such as `(?i)` replace a flag suffix; lookaround and backreferences are
errors. Matching is over Unicode scalar values, consistent with string length.

There is no `format` constraint. A well-known text shape is an alias with a
pattern and a `@repr` naming the native type, for example
`type Mac = string {/(?i)([0-9a-f]{2}:){5}[0-9a-f]{2}/} @repr(mac)`; the
pattern validates, the native type canonicalises on encode where a runtime
has it. `duration` and `timestamp` are integers with `@unit`, not formats.

**Unique (locked).** No two elements equal; order is preserved. Applies only
to lists whose element type is a primitive, alias, enum or `ref`. On a list of
models it is an error: uniqueness of models is what `key` means. Maps are
unique by construction. There is no `set` type: the wire has no set encoding
and order keeps mattering (Smithy removed `set` for the same reasons); a native
set is reachable with `@repr(set)` if wanted. Error path points at the second
occurrence.

Removed from earlier drafts, with the reason: `len` (a range already means
size on a sized type), `in` (a valued enum adds a name, docs, ids and a native
type), `multiple_of` (arithmetic belongs in `check`), `format` (a pattern plus
`@repr` plus normalisation, not a constraint).

**One brace per field.** Constraints appear once, after the whole type, and
apply to it. Type arguments are bare types: `list[uint {1..}]` is an error.
A constrained element, map value or generic argument gets a named alias,
which also gives it documentation and a place for `@unit`:

```kotlin
type Speed = uint {1..}  @unit(mbps)

speeds: list[Speed] {1..}     // each element at least 1; at least one element
```

### Defaults

```kotlin
mtu:      u16 = 1500
state:    State = up
size:     Size = fixed(64)             // union default: variant and payload
dst:      AutoPattern[mac] = "00:00:00:00:00:00"   // literal selects `value`
packet:   list[Header]{1..} = [ethernet]   // list default of unit-ish variants
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
  in `pattern[u16 | EtherType]` so a field accepts either raw numbers or names.

- A valued enum is also how a closed set of legal numbers is expressed
  (`enum Speed: uint { mbps_10 = 10, mbps_100 = 100 }`). There is no `in`
  constraint: an enum adds a name, per-value documentation, stable ids and a
  native type, and a legal value with no name usually has no documented
  meaning either.

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
  port:           ref[Port]
  lag:            ref[Lag]
  vxlan:          ref[VxlanV4Tunnel | VxlanV6Tunnel]
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
  tx:  ref[Port]!
  rx:  list[ref[Port]]{1..}
}
```

- `ref[T]` is a string on the wire whose value must equal the `key` of some `T`
  reachable from the document root. `T` must have exactly one `key` field.

- `ref[A | B]` resolves against any of the listed types.

- Resolution scope is the document root by default. `ref[T in parent]` limits
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
union Pattern[T] = value {
  value:     T
  values:    list[T]{1..}
  increment: Counter[T]
  decrement: Counter[T]
}

/// A pattern that may also be auto-assigned by the implementation.
union AutoPattern[T] with Pattern[T] {
  auto
}

/// A pattern whose values can be tagged in metrics.
model Tagged[T] {
  pattern:     Pattern[T]!
  metric_tags: list[MetricTag]
}

/// An arithmetic progression.
model Counter[T] {
  start: T!
  step:  T = 1
  count: uint {1..} = 1
}
```

- Type parameters are instantiated at compile time (monomorphised). Instantiated
  names are deterministic: `Pattern[mac]` becomes `Pattern_Mac` in proto and
  `PatternMac` in Go unless `@proto_name("PatternFlowEthernetDst")` pins a
  legacy name.

- A `with` on a union adds the variants of another union (union mixin).

- A type parameter is used bare inside the generic body, like any type
  argument; constraints on it come from the alias passed at instantiation
  (`Counter[Speed]`), never from the generic itself.

This single feature removes the `x-field-pattern` meta-schema and hundreds of
generated OTG messages.

## 8a. Tables

```kotlin
/// Per-flow traffic metrics.
model FlowMetric {
  name:      ref[Flow] key
  frames_tx: uint!          @unit(frames)
  loss:      float{0..100}  @unit(percent)
  state:     TransmitState
}

/// Metrics selected by the request.
union MetricsResponse {
  flow: table[FlowMetric]
  port: table[PortMetric]
}

rpc get_metrics(MetricsRequest) -> MetricsResponse
```

`table[T]` is a collection over an ordinary flat model; nothing in the model
says "table". The compiler checks flatness where `table[T]` is used, as it
checks any generic argument. The same `FlowMetric` can appear in a `list` or
as a single value elsewhere.

### JSON

One array per column, all the same length. For three rows where `f2` has no
loss value and no state:

```json
{
  "flow": {
    "name":      ["f1", "f2", "f3"],
    "frames_tx": [1000, 1000, 1000],
    "loss":      [0.0, null, 2.5],
    "state":     ["started", null, "stopped"]
  }
}
```

- A nullable cell is `null`. A nullable column with no values at all may be
  omitted. A non-null column must be present whenever there are rows.

- Row count is the length of any present column; no columns means no rows.

- A decoder accepts the row layout too (an array instead of an object cannot
  be confused with it), so row-oriented producers interoperate; what the model
  says governs what is emitted.

### Protobuf

Each flat model used as a table gets one generated message, with the row
model's field ids as column ids:

```proto
message FlowMetric {                        // the row model, as always
  string name = 1;
  uint64 frames_tx = 2;
  optional double loss = 3;
  optional TransmitState state = 4;
}

message FlowMetricTable {                   // table[FlowMetric]
  repeated string        name      = 1;     // same ids as the row model
  repeated uint64        frames_tx = 2;     // packed
  repeated double        loss      = 3;     // packed; a null cell holds 0
  repeated TransmitState state     = 4;     // packed; a null cell holds the zero value
  bytes loss_nulls  = 1003;                 // validity bitmap: bit i set = row i is null
  bytes state_nulls = 1004;                 // omitted when the column has no nulls
}
```

A `repeated` field cannot hold a null, so every nullable column carries a
validity bitmap: one bit per row, least-significant bit first, packed into
`ceil(rows / 8)` bytes, at id `1000 + column id`, a range the lockfile keeps
clear. The column keeps a placeholder at a null position so the columns stay
aligned. Non-null columns never have a bitmap. This is the same scheme Arrow
uses, so converting a protobuf column to an Arrow array copies the values and
the bitmap with no per-cell work. The alternatives were rejected: wrapper
messages per cell double the size and allocate per cell; a `repeated bool`
costs a byte per row instead of a bit; a sparse index list is not what Arrow
does.

### Arrow

When the Arrow codec is negotiated, a table is a record batch with the
mechanically derived schema (`name: utf8`, `frames_tx: uint64 not null`,
`loss: double`, `state: dictionary<int32, utf8>`), and Arrow's own validity
bitmaps replace the `*_nulls` fields. Mapping details are in document 06.

### Chunking

Collections are closed under chunking: any row subset of a `table[T]` is a
`table[T]`, any sub-list of a `list[T]` is a `list[T]`, any disjoint sub-map
of a `map[K, V]` is a `map[K, V]`. A chunk is a complete, decodable value of
the same type as the whole, and concatenating the chunks in order gives the
whole. Models do not have this property.

The rule that follows: **a collection at the top level of an rpc signature,
or a union whose variants are all collections, is one logical value that
transports may deliver as an ordered sequence of chunks.** No keyword is
needed.

- gRPC: the method is generated as a server stream (or client stream for a
  request) of the collection's message; a small result is a one-message
  stream. See document 06 for how gRPC frames messages.

- HTTP: `application/x-ndjson`, one chunk per line; a client that asks for
  `application/json` receives the concatenation as a single value.

- Arrow: the IPC stream format is already a schema followed by record
  batches.

- Redis: one stream entry per chunk, framed by the envelope's `seq` and `last`.

The producer chooses chunk boundaries, defaulting to 64k rows or 4 MB,
overridable with `@chunk(rows=…)` on the rpc. Receivers decode each chunk
independently and run every local constraint on it; collection-wide
constraints (`key` uniqueness, `unique`, `len` of the whole) are checked
incrementally and finalised at the last chunk. Generated clients offer both
views: iterate chunks as they arrive, or `collect()` into the whole.

Within a chunk there is no streaming in any format: a columnar JSON object,
a protobuf message and an Arrow record batch must each be received whole
before any row is usable. Chunk size is the knob that trades latency and
memory against per-message overhead.

### `stream` versus chunking

`stream T` keeps its gRPC meaning: a sequence of independent, complete values
of `T`, possibly unbounded. Live telemetry is a stream of models that happen
to contain tables, not a chunked table:

```kotlin
model MetricsUpdate {
  at:    timestamp!
  kind:  UpdateKind = snapshot        // snapshot | delta
  flows: table[FlowMetric]
}

rpc watch_metrics(MetricsRequest) -> stream MetricsUpdate
```

Each message is one complete batch; keys are unique per message, not across
the stream. A model that merely contains a collection, such as
`MetricsResponse { flows: table[FlowMetric], ports: table[PortMetric] }`
written as a model rather than a union, is one message and cannot be split;
`xapic lint` flags it on a unary rpc and suggests the union form or returning
the collection directly. That shape is where OTG's `x-stream` came from.

## 9. Cross-field rules (`check`)

```kotlin
model Counter[T] {
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
- comparisons `== != < <= > >=`, arithmetic `+ - * / %`, boolean `&& || !`,
  `in [..]`

Arithmetic rules such as alignment live here rather than in the constraint
set: `check step % 8 == 0 : "step must be a multiple of 8"`.
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
| `@chunk(rows=65536)` | rpc returning or accepting a collection | Producer-side chunk size hint (section 8a) |
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
  /// Metrics for the flows or ports named in the request; chunked in transit.
  rpc get_metrics(MetricsRequest) -> MetricsResponse
  /// Captured packets, streamed as they become available.
  rpc get_capture(CaptureRequest) -> stream CaptureChunk
}

/// Structured failure.
error Error {
  code:  ErrorCode!
  items: list[ErrorItem]{1..}
}

model ErrorItem {
  /// JSON pointer into the request, for example `/flows/0/size/fixed`.
  path:    string
  message: string!
}
```

- `rpc name(Request?) -> Response [throws Err]`. Omitting `throws` uses the
  service default. `-> unit` for no response body.

- `stream` on the request or response side means a sequence of independent
  values. A top-level collection (or union of collections) needs no `stream`:
  it is one value that transports chunk (section 8a).

- Without `@http`, the default binding is `POST /<service>/<rpc>`; gRPC and
  Redis bindings are always derived from the service and rpc names.

- The encoding (JSON, protobuf, Arrow for tables) is chosen by the caller per
  client or per call; it is not part of the declaration. Layout (row versus
  column) is part of the declaration: `list[T]` versus `table[T]`.

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
  port:           ref[Port]
  /// The LAG the interface is connected to.
  lag:            ref[Lag]
  /// The VXLAN tunnel the interface rides on.
  vxlan:          ref[VxlanV4Tunnel | VxlanV6Tunnel]
  /// A link simulated between two emulated interfaces.
  simulated_link: SimulatedLink
}
```

InfraGraph `Infrastructure.Edge` and `Device.Edge` today: two types, two
endpoint types, string endpoints with slice syntax, about 130 lines. xapi:

```kotlin
/// Connects two endpoints with a link using a wiring scheme.
model Edge[E] {
  ep1:    E!
  ep2:    E!
  /// How ep1 instances are wired to ep2 instances.
  scheme: Scheme = one2one
  link:   ref[Link]!
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
  instance:        ref[Instance]!
  index:           Slice
  component:       ref[Component in Device]!
  component_index: Slice
}

model Infrastructure with Named {
  description: string
  devices:     list[Device]
  links:       list[Link]
  instances:   list[Instance]
  edges:       list[Edge[InstanceEndpoint]]
}
```
