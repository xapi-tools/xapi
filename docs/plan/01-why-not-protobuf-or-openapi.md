# 01 — Why not Protobuf or OpenAPI

This document lists the concrete defects xapi is designed to remove. Each row
names the defect, the symptom it produces in real models, and the xapi feature
that addresses it. The features are specified in `03-language-sketch.md`.

## Protobuf (proto3)

Protobuf is an excellent serialisation format and a poor modelling language. The
defects below are not bugs; they are consequences of being wire-first.

**No default values**

Zero means both "unset" and "zero". Every API re-invents defaults in prose and in each SDK.

xapi answer: `= value` on any field; defaults are in the IR and applied by `normalize()` identically everywhere.

**No required fields**

Required-ness lives in documentation; servers validate differently from clients.

xapi answer: `field: Type!`. Required is a type property checked by every SDK.

**Presence is an afterthought**

`optional` keyword added in 3.15; scalars had no presence for years; wrappers everywhere.

xapi answer: Every field has presence. Proto backend emits `optional` for scalars automatically.

**No constraints**

No min, max, length, pattern, uniqueness. protovalidate exists but is a separate language (CEL) and not available in every runtime.

xapi answer: Constraints in braces after the type. Compiled to native checks and, optionally, to `buf.validate` CEL.

**No typed references or keys**

A "port name" is a `string`. Referential integrity is a convention.

xapi answer: `key` fields and `ref<T>` types. Resolved and checked at validation time.

**Enums must start at zero, share C scope, are open**

`FOO_UNSPECIFIED = 0` noise; prefixes on every value; unknown values silently accepted.

xapi answer: Enums are scoped and closed. The backend emits the zero sentinel and prefixes; authors never see them. Defaults on enums are allowed.

**`oneof` restrictions**

No repeated or map members, no payload-less variants without defining empty messages, no default variant.

xapi answer: `union` allows lists, unit variants, defaults. Backend synthesises wrapper messages where proto needs them.

**No small integer types, no fixed arrays**

`uint8` becomes `uint32` with a comment; MAC addresses become `string` or `bytes` with no format.

xapi answer: `u8 u16 i8 i16`, formatted scalars `mac ipv4 ipv6 cidr hex uuid`, `bytes{len n}`. Backend widens and validates.

**No composition**

Shared fields are copy-pasted across messages; no mixins.

xapi answer: `mixin` with flattening at compile time.

**No generics**

A "pattern over T" needs one message per T per field. OTG has hundreds of `PatternFlow*` messages.

xapi answer: Parametric `model<T>` and `union<T>`, monomorphised with stable names.

**Comments are not data**

Documentation is lost on the wire and in descriptors unless source info is retained.

xapi answer: `///` docs are mandatory, in the IR, in generated docstrings.

**Hand-numbered fields**

Humans assign, reserve and audit numbers. Mistakes are silent until an incompatible release.

xapi answer: Compiler-assigned ids pinned in `xapi.lock`.

**JSON mapping quirks**

int64 as string, lowerCamel by default, enum names vs numbers, `Any` type URLs.

xapi answer: xapi JSON is defined by the language: snake_case, numbers as numbers within safe range, strings for 64-bit. Proto backend maps accordingly.

**Services are transport-shaped**

Unary/stream only; HTTP mapping needs `google.api.http`; errors are status codes.

xapi answer: `service`/`rpc` with typed error, `stream`, `@http`. Backends emit gRPC and HTTP bindings.

**No units or quantities**

`uint64 bandwidth` with a comment, or three fields for three units (InfraGraph).

xapi answer: `@unit(...)` annotation carried into docs, IR and generated names.


## OpenAPI (3.0 / 3.1)

OpenAPI is a good HTTP documentation format and a poor source of truth for
typed SDKs.

**Verbosity**

Each field is 4 to 8 lines. OTG `Flow.Header` (25 variants) is 140 lines.

xapi answer: One line per field. `Flow.Header` is 28 lines.

**No stable field identity**

Protobuf generation is impossible without `x-field-uid` on every property and enum value.

xapi answer: Lockfile ids.

**JSON-Schema composition**

`oneOf`, `anyOf`, `allOf`, `not`, `if/then` are undecidable for generators. OTG bans all of them and invents `choice` and `x-include`.

xapi answer: `union` and `mixin` are first-class and closed.

**Unions have no good encoding**

`discriminator` is optional and inconsistently supported; the OTG `choice` pattern duplicates every variant name twice and numbers them twice.

xapi answer: One declaration per variant.

**HTTP-path-centric**

RPC identity is `operationId`, an optional string. Streaming is absent. OTG adds `x-stream`.

xapi answer: `rpc` is the unit; `@http` is a binding; `stream` is a keyword.

**No module system**

`$ref` is a file path. Namespaces are dotted strings by convention. Bundling is a separate step with its own bugs (openapiart #495, #501).

xapi answer: `package` and `use`.

**Extensions are unregulated**

Every `x-` key is tool-specific, undocumented to other tools, and unvalidated.

xapi answer: Annotations are declared with a schema; unknown annotations are errors.

**`nullable`, `required`, `default` interplay**

Three partially overlapping ways to say "may be absent".

xapi answer: One: presence plus `!` plus `=`.

**References and keys**

None. `x-constraint` with yamlpath strings.

xapi answer: `key` and `ref<T>`.

**Responses by status code**

A `200` and a `default`; everything else is prose. openapiart issue #500 is about this.

xapi answer: `rpc f(Req) -> Resp throws Err`. Status codes are a binding detail.

**Descriptions carry normative rules**

"MUST be unique", "MUST exist in devices", "MUST be configured unless simulated_link". None is checked.

xapi answer: Those are `key`, `ref<T>` and `check` respectively.


## What xapi keeps from each

- From Protobuf: numeric field identity, `oneof` semantics, proto3 scalars as
  the wire baseline, gRPC services, the idea that a schema evolves by
  addition and reservation.

- From OpenAPI: human-readable documentation output, the HTTP binding model
  (method, path, body), JSON as the readable interchange form.

- From openapiart and the OTG model guide: snake_case fields, PascalCase types,
  `name` as primary key, the discipline that every element is documented, the
  lifecycle statuses `current`, `deprecated`, `obsolete`, `under_review`.
