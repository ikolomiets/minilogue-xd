# Structured score formats and exact musical time

Research for **Establish structured-document and musical-time representation options**, 13 September 2026. This report supplies evidence for later decisions; it chooses no DSL syntax or adapter behavior.

## Findings

A purpose-built structured document can represent the agreed scope without adopting an entire notation interchange format. JSON plus a schema is feasible; YAML adds serialization policy choices. MusicXML supplies established notation/performance distinctions, while MNX supplies relevant JSON modeling examples but remains a draft. These are research conclusions, not format selections.

### JSON and JSON Schema

JSON has ordered arrays and unordered objects. Scheduling must therefore not derive meaning from object-member order. Duplicate names have inconsistent receiver behavior; rejecting them needs a parsing policy, before a map-like representation loses the duplicates. JSON number syntax does not itself guarantee arbitrary precision. Integers within `[-(2^53)+1, (2^53)-1]` interoperate exactly with binary64-based consumers. JSON has no fraction literal. These facts favor explicitly bounded integer components or an explicitly parsed string when exact fractions are required. [RFC 8259, sections 4–6](https://www.rfc-editor.org/rfc/rfc8259)

JSON Schema 2020-12 describes types, required fields, enums, numeric bounds and array constraints. Its `integer` type means a number with zero fractional part, not necessarily an integer-shaped token: accepting `1.0` in the schema but rejecting it in Rust can create disagreement. `format` is normally annotation unless the relevant assertion vocabulary is required. [Validation specification](https://json-schema.org/draft/2020-12/json-schema-validation)

`$schema` identifies the schema dialect; `$id` identifies a schema resource. Neither automatically defines the DSL's document version or migration policy. `additionalProperties: false` closes a simple object; `unevaluatedProperties: false` can close objects composed across subschemas. Allowing unspecified properties is distinct from deliberately providing an extension namespace. [Core specification](https://json-schema.org/draft/2020-12/json-schema-core)

**Inference:** ordinary portable schema keywords cannot generally perform arbitrary rational arithmetic across events or follow instance note IDs to validate musical relationships. A second semantic-validation stage must check denominator normalization/overflow, measure totals, ID uniqueness by field, tie targets and temporal adjacency, parameter compatibility, and automation conflicts. `uniqueItems` compares entire values, not one selected ID field. The schema can reject malformed shapes before this stage. [Validation keyword definitions](https://json-schema.org/draft/2020-12/json-schema-validation)

### YAML as alternate serialization

YAML 1.2.2 offers comments and several scalar styles, but also tags, aliases, non-string mapping keys and schema-dependent scalar resolution. Its JSON and Core schemas are different scalar-resolution choices; the Core schema includes representations beyond JSON such as infinities. YAML mapping keys must be unique. [YAML 1.2.2 specification, sections 3 and 10](https://yaml.org/spec/1.2.2/)

**Inference:** YAML could encode the same DSL data model if a documented loader profile restricts it to JSON-compatible trees, string keys and accepted finite numeric values, with explicit duplicate, alias and tag handling. That adds authoring convenience and conformance work; it does not remove the need for the DSL schema or semantic validator. No Rust YAML implementation was selected or tested.

### Exact musical duration options

The following are candidate representations, not proposed syntax:

| Representation | Useful property | Contract still needed |
| --- | --- | --- |
| Integer pair/object such as `[1, 3]` | Exact, individually schema-checkable components | Unit, bounds, positive denominator, normalization, arithmetic overflow |
| String such as `"1/3"` | Exact across numeric runtimes | Grammar, parser, bounds, canonical spelling |
| Integer duration with shared divisions | Compact common grid | Resolution selection, changes of grid, maximum denominator growth |
| Written note value plus tuplet context | Retains notated rhythmic structure | Conversion authority and nested-ratio rules |

MusicXML's divisions count subdivisions of a quarter note; duration uses those divisions. This preserves fractional rhythm through a shared scale, including tuplets. Its MIDI-file compatibility advice about division limits is not a requirement to quantize this DSL's host scheduler. [MusicXML divisions](https://www.w3.org/2021/06/musicxml40/musicxml-reference/elements/divisions/)

**Arithmetic illustration:** with a quarter note as one unit, an eighth-note triplet lasts `1/3`; three sum exactly to one. Keeping fractions through score compilation avoids rounding each event independently. Converting an absolute rational position through the tempo map to a host deadline requires a separate rounding contract. Clock ticks and score-event resolution need not be the same grid. These are derived engineering implications, not claims of measured scheduler precision.

### Notation and performance semantics

MusicXML distinguishes intended duration from notation, and its `tie` affects sound while `tied` specifies notation. Its `time-modification` represents the actual/normal duration relationship; nested tuplets and presentation need additional notation information. [Duration](https://www.w3.org/2021/06/musicxml40/musicxml-reference/elements/duration/), [Tie](https://www.w3.org/2021/06/musicxml40/musicxml-reference/elements/tie/), [Time modification](https://www.w3.org/2021/06/musicxml40/musicxml-reference/elements/time-modification/)

MusicXML pitch separates diatonic step, alteration and octave. **Inference:** a MIDI note number alone cannot preserve enharmonic spelling, even when two spellings produce the same MIDI pitch. [Pitch](https://www.w3.org/2021/06/musicxml40/musicxml-reference/elements/pitch/)

MNX note values retain a base value and augmentation dots. Its tuplets contain events and define outer/inner durations, allowing nested contexts and an explicit content-total consistency check. These demonstrate that exact playback duration and written grouping can coexist. [Note value](https://mnx.formats.music/docs/mnx-reference/objects/note-value/), [Tuplet](https://mnx.formats.music/docs/mnx-reference/objects/tuplet/)

MNX ties can refer to a target note ID; source and target must have the same sounded pitch but may use enharmonic spellings. This is a useful reference model for separating identity and pitch, not a requirement to support all MNX tie variants. [Tie](https://mnx.formats.music/docs/mnx-reference/objects/tie/)

### Maturity and Rust feasibility

MusicXML 4.0 is a dated Final Community Group Report, published in 2021; it is not a W3C Recommendation. MNX's current home calls itself a 1.0 draft and explicitly says it has no stable implementation standard. Its integer document-version field aims at backward compatibility but does not make the draft stable. Reuse of concepts is less coupled to upstream changes than adopting its evolving schema wholesale. [MusicXML 4.0 status](https://www.w3.org/2021/06/musicxml40/), [MNX status](https://mnx.formats.music/docs/), [MNX version metadata](https://mnx.formats.music/docs/mnx-reference/objects/mnx/)

Serde supports typed deserialization and configurable enum representations. Unknown fields are ignored by default for self-describing formats; `deny_unknown_fields` opts into rejection and is incompatible with `flatten`. These choices must agree with the published schema. `serde_json::Value` provides a dynamic JSON representation, but using it is not itself domain validation. [Serde container attributes](https://serde.rs/container-attrs.html), [serde_json Value](https://docs.rs/serde_json/latest/serde_json/value/enum.Value.html)

Repository observation: [minilogue-xd Cargo manifest](../../../crates/minilogue-xd/Cargo.toml) currently has no direct Serde, JSON Schema, YAML or rational-number dependency. Rust integration is feasible in principle; this investigation did not benchmark parsers, select crates, change feature boundaries, or compile a prototype.

## Questions now ready for decision

- Is the authoritative document a purpose-built JSON model, with YAML excluded or strictly profiled?
- Which exact time unit and fraction encoding are canonical, and which numeric/size bounds apply?
- Must pitch spelling and written tuplet/tie grouping survive normalization, and which fields are authoritative when redundant representations disagree?
- What are the version, extension and unknown-field policies, and what semantic errors must preflight reject before playback?

These refine the existing document-shape and musical-semantics decisions rather than mandate new tickets.

## Evidence limits and completion

The research question is answered at decision-support fidelity. No consumer interoperability, AI generation reliability, crate compatibility or hardware timing was measured. The MNX rhythmic-position detail page could not be fetched reliably after the documentation moved from W3C GitHub hosting to `mnx.formats.music`; no current normative fraction-field claim depends on it. Exact duration options above follow arithmetic and the successfully retrieved specifications. Syntax selection, full schema design and acceptance fixtures remain human-facing decision work.
