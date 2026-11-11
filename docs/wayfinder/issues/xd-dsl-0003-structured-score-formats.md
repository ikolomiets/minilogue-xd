---
{
  "id": "xd-dsl-0003",
  "title": "Establish structured-document and musical-time representation options",
  "status": "closed",
  "labels": [
    "wayfinder:research"
  ],
  "parent": "xd-dsl-0001",
  "assignee": "ikolomiets",
  "mode": "AFK",
  "blocked_by": [],
  "worker": "score_format_research",
  "research_branch": "research/structured-score-formats",
  "resolution": "../comments/xd-dsl-0003/0002-resolution.md"
}
---

# Establish structured-document and musical-time representation options

## Question

What do primary specifications establish about structured formats suitable for AI/application-authored scores, and what reusable representations exist for exact musical duration, ties, tuplets, pitch spelling, and validation?

Compare a purpose-built JSON document with JSON Schema, YAML as an alternate serialization, and relevant MusicXML/MNX concepts. Consult official specifications and first-party Rust library documentation only where needed to establish practical feasibility. Distinguish document validation from cross-event semantic validation; investigate exact rational timing, numeric portability, schema versioning, and unknown-field behavior. The scope is prepared scores with live parameter updates via a Rust API, not a service or a generative language.

Produce cited findings and tradeoffs. Do not choose a format, add a parser, or make human-facing design decisions.

## Activity

- [Research branch context](../comments/xd-dsl-0003/0001-research-context.md)
- [Research resolution and evidence](../comments/xd-dsl-0003/0002-resolution.md)
