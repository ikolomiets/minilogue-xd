---
{
  "id": "xd-dsl-0012",
  "title": "Define the specification handoff and acceptance evidence",
  "status": "open",
  "labels": [
    "wayfinder:grilling"
  ],
  "parent": "xd-dsl-0001",
  "assignee": null,
  "mode": "HITL",
  "blocked_by": [
    "xd-dsl-0004",
    "xd-dsl-0005",
    "xd-dsl-0006",
    "xd-dsl-0007",
    "xd-dsl-0008",
    "xd-dsl-0009",
    "xd-dsl-0010",
    "xd-dsl-0011"
  ]
}
---

# Define the specification handoff and acceptance evidence

## Question

What evidence makes the resulting DSL specification and adapter contract complete enough to implement without unresolved product decisions?

Agree a traceability checklist from musical and control requirements to worked valid/invalid documents, expected event timelines, expected MIDI transactions, and diagnostic outcomes. Cover feature completeness, notation edge cases, live-update arbitration, onboard sequencing, timing/load behavior, and stop/disconnection recovery. Separate deterministic mock-output checks from required hardware experiments; define how documented limitations appear in the handoff. This ticket decides acceptance criteria and required specification artifacts; it does not implement the adapter.

## Blocked by

- [Choose the structured score document shape](xd-dsl-0004-document-shape.md)
- [Define musical time and note semantics](xd-dsl-0005-musical-semantics.md)
- [Bound the supported performance environment](xd-dsl-0006-performance-environment.md)
- [Define sound-parameter and patch control semantics](xd-dsl-0007-sound-and-patches.md)
- [Define the boundary between scores and onboard sequences](xd-dsl-0008-onboard-sequencer.md)
- [Define arbitration between score automation and live changes](xd-dsl-0009-live-updates.md)
- [Define scheduling and MIDI-output guarantees](xd-dsl-0010-scheduler-contract.md)
- [Define the Rust adapter and CLI boundary](xd-dsl-0011-adapter-api.md)
