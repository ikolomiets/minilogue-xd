---
{
  "id": "xd-dsl-0007",
  "title": "Define sound-parameter and patch control semantics",
  "status": "open",
  "labels": [
    "wayfinder:grilling"
  ],
  "parent": "xd-dsl-0001",
  "assignee": null,
  "mode": "HITL",
  "blocked_by": [
    "xd-dsl-0002",
    "xd-dsl-0013"
  ]
}
---

# Define sound-parameter and patch control semantics

## Question

How should the DSL name and type every in-scope MIDI-addressable sound parameter, and distinguish live sound changes, patch preparation, edit-buffer loading, stored-program selection, and persistent patch writes?

Use the capability inventory to decide units and ranges, enum representation, validation versus clamping, dependencies between settings, unsupported operations, and treatment of operations that have only SysEx access. Decide whether values describe physical units or hardware/normalized values, how patch defaults and partial patches behave, and when operations may occur during a performance. Completeness needs an explicit capability mapping, not a generic raw-MIDI escape hatch.

## Blocked by

- [Establish XD MIDI capabilities and playback constraints](xd-dsl-0002-xd-midi-capabilities.md)
- [Choose the evidence standard for disputed MIDI encodings](xd-dsl-0013-encoding-evidence.md)
