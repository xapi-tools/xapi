# xapi

**xapi** is a small, terse interface-definition language for data models and
RPCs. **xapic** is its compiler: it checks a model, assigns stable wire ids,
emits a machine-readable intermediate representation (IR), and generates
Protobuf, OpenAPI, Python, Go and C++ artifacts that read like code a careful
engineer would write by hand in each language. Generated services are
transport-neutral (HTTP, gRPC, Redis) with JSON, Protobuf and Arrow encodings,
and big result sets are first-class `table` types rather than lists of rows.

xapi exists because the two mainstream options force a bad trade:

- **OpenAPI** is verbose, has no stable field identity, treats RPCs and
  streaming as afterthoughts, and its JSON-Schema composition keywords are
  hostile to code generation. Projects such as OTG end up banning most of it
  and re-inventing unions, composition and references as `x-` extensions.
- **Protobuf** is wire-first. It has no defaults, no constraints, no typed
  references, no composition, no generics, awkward enums and awkward unions.
  Everything a data model needs beyond serialisation is bolted on.

xapi is semantics-first. Validation (required, defaults, ranges, lengths,
patterns, uniqueness, referential integrity, cross-field rules) is part of the
language, enforced identically by every generated SDK, and exported in the IR
so automations can reason about a model without parsing source.

## Status

Planning. Nothing is implemented yet. Start with
[docs/plan/00-roadmap.md](docs/plan/00-roadmap.md).

| Document | What it covers |
|---|---|
| [00-roadmap.md](docs/plan/00-roadmap.md) | Goals, non-goals, phases, repository layout, decisions to confirm |
| [01-why-not-protobuf-or-openapi.md](docs/plan/01-why-not-protobuf-or-openapi.md) | The specific defects xapi is designed to remove |
| [02-otg-and-infragraph-critique.md](docs/plan/02-otg-and-infragraph-critique.md) | Flaws observed in the OTG and InfraGraph models, each mapped to a fix |
| [03-language-sketch.md](docs/plan/03-language-sketch.md) | The xapi language: syntax, types, constraints, unions, refs, generics, services |
| [04-xapic-and-generators.md](docs/plan/04-xapic-and-generators.md) | Compiler architecture, lockfile, per-language idiom contracts |
| [05-validation-ir-and-automation.md](docs/plan/05-validation-ir-and-automation.md) | Validation semantics, the IR, conformance suite, automation surfaces |
| [06-encodings-transports-and-tables.md](docs/plan/06-encodings-transports-and-tables.md) | JSON, Protobuf and Arrow codecs; HTTP, gRPC and Redis transports; the `table` type for big data |
