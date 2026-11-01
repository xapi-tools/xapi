# 05 — Validation, IR and automation

## Validation semantics

Validation is a property of the model, and every runtime implements the same
algorithm. The algorithm has four passes, in order, stopping at the first pass
that produces errors unless `--all` is requested:

1. **Shape.** Unknown fields (error by default, warning with `lenient`), union
   with zero or more than one variant, enum value not in set, scalar format
   (`mac`, `ipv4`, …), integer range of the declared width.
2. **Local constraints.** Required fields present, braces constraints, `unique`
   on lists, `key` uniqueness within the containing list.
3. **Document constraints.** `key global` uniqueness, `ref<T>` resolution.
4. **Checks.** `check` expressions, evaluated after defaults are applied.

`normalize()` fills defaults and canonicalises formatted scalars; it never
fails. `validate()` runs the passes above on the normalised view without
mutating the caller's object. Decoders run pass 1 only; a successfully decoded
object can still be invalid, and callers choose when to pay for passes 2 to 4.

### Error shape

```json
{
  "errors": [
    { "path": "/flows/0/size/fixed", "code": "range",
      "message": "value 20 is below minimum 64", "min": 64, "actual": 20 },
    { "path": "/flows/1/tx_rx/port/tx", "code": "unknown_ref",
      "message": "no Port named \"p9\"", "target": "otg.port.Port", "value": "p9" }
  ]
}
```

Codes are closed: `required`, `range`, `length`, `pattern`, `in`, `multiple_of`,
`unique`, `unknown_ref`, `union_multiple`, `union_none`, `enum_unknown`,
`format`, `unknown_field`, `check_failed`. Extra keys per code are specified in
`spec/semantics.md` so a UI can render them without parsing messages.

Servers return the same shape in the RPC `Error` type, so a client-side
validation failure and a server-side rejection look identical to the caller.

## The IR

`xapic ir` writes one JSON document for the whole project. It is the contract
for every downstream tool, including the generators inside `xapic`.

```json
{
  "ir_version": "0.1",
  "project": { "name": "otg", "version": "1.53.0" },
  "packages": {
    "otg.device": {
      "types": {
        "Ethernet": {
          "kind": "model",
          "doc": "An emulated Ethernet interface.",
          "mixins": ["otg.device.EthernetBase"],
          "fields": [
            { "name": "mtu", "id": 6, "type": {"scalar": "u16"},
              "required": false, "default": 1500,
              "constraints": [{"range": {"min": 64, "max": 9216}}],
              "annotations": {"unit": "bytes"},
              "doc": "Maximum transmission unit.",
              "span": {"file": "otg/device/ethernet.xapi", "line": 41} }
          ],
          "checks": [ { "expr": "...", "message": "...", "ast": { } } ]
        }
      }
    }
  },
  "services": { }
}
```

Properties:

- **Deterministic.** Sorted keys, stable field order (source order), no
  timestamps. Two runs on the same input produce identical bytes.
- **Complete.** Resolved types only; generics instantiated; mixins flattened
  but recorded; ids present; docs present; constraint ASTs present so a tool
  need not parse expressions.
- **Schema'd.** `spec/ir.schema.json` (JSON Schema 2020-12) is published and
  versioned with `ir_version`. Additive changes bump the minor; anything else
  bumps the major.
- **Self-describing paths.** Every field has a canonical path
  (`otg.device.Ethernet.mtu`) that capability declarations, lints, docs and
  MCP tools can use as a stable identifier across releases.

## Conformance suite

`conformance/cases/<name>/` contains:

- `model.xapi` — the model under test (small, focused).
- `instances/*.json` — documents, each with a sibling `*.expect.json` listing
  the exact error set (codes and paths) or `[]` for valid.
- `normalized/*.json` — expected output of `normalize()` for selected inputs.

A runner per language loads the generated SDK for `model.xapi`, runs each
instance through decode, validate and normalize, and diffs the result. The
suite is the definition of "same validation everywhere". Cases are added for
every bug fixed in any runtime.

Initial case groups: scalar formats, ranges and lengths, unions (tagged and
untagged), refs (root scope, `in` scope, multi-target), keys (list and global),
generics (`Pattern<T>` across types), checks, deprecations (warning surface),
unknown fields, defaults on nested models and union payloads.

## Automation surfaces

| Surface | Produced by | Consumer |
|---|---|---|
| `ir.json` + `ir.schema.json` | `xapic ir` | Any tool; the generators; MCP servers; diff tools |
| JSON Schema for instance documents | `xapic gen jsonschema` | CI validation of YAML configs without an SDK; editors |
| OpenAPI 3.1 | `xapic gen openapi` | Redoc docs, existing OTG tooling |
| HTML docs | `xapic gen docs` | Humans; rendered from IR so prose and rules never drift |
| Breaking-change report | `xapic check --against` | PR gates, release notes |
| Capability paths | IR field paths | Implementations advertise supported paths; clients can pre-check a config |
| LSP | Phase 7 | Editors: hover shows constraints and docs; go-to-definition across `use` |

## Lint rule sets

`xapic lint` ships opt-in rule sets so model guides become executable:

- `core`: docs present, unused types, uid gaps (informational), required with
  default, numeric field with unit-like words in docs but no `@unit`,
  description containing `MUST` (suggest `key`/`ref`/`check`).
- `otg`: snake_case fields, PascalCase types, no scalar fields on root types,
  `name` first in `Named` types, deprecated before removed.
- `infragraph`: counts have `{1..}`, every enum value documented, no string
  fields whose docs mention slice or path syntax.

## Open design questions (beyond the decisions in 00)

| # | Question | Current leaning |
|---|---|---|
| Q1 | Should `json` fields accept `@schema` pointing at an xapi type as well as external JSON Schema? | Yes: `@schema(Capabilities)` validates the opaque value against a model from the same project. |
| Q2 | Quantifiers in `check` (`all x in list: …`) in v0.1 or v0.2? | v0.2. Keep v0.1 compilable to native code with no closures. |
| Q3 | Should `ref<T>` carry an optional `?` for soft references that may dangle? | No. A dangling reference is always an error; model it as `string` if it is not a reference. |
| Q4 | Map keys of enum type on the proto side (proto forbids enum map keys)? | Encode as the enum's string name; document the loss. |
| Q5 | Per-package versions or one project version? | One project version (matches OTG and InfraGraph practice); `@since` records introduction. |
| Q6 | Where do lockfiles live for generic instantiations that cross packages? | In the instantiating package's section; the generic definition has no ids of its own. |
