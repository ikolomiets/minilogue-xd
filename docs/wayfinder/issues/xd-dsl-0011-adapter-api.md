---
{
  "id": "xd-dsl-0011",
  "title": "Define the Rust adapter and CLI boundary",
  "status": "open",
  "labels": [
    "wayfinder:grilling"
  ],
  "parent": "xd-dsl-0001",
  "assignee": null,
  "mode": "HITL",
  "blocked_by": [
    "xd-dsl-0004",
    "xd-dsl-0007",
    "xd-dsl-0008",
    "xd-dsl-0009",
    "xd-dsl-0010"
  ]
}
---

# Define the Rust adapter and CLI boundary

## Question

What public Rust API and CLI behavior must connect structured documents to validated performances and live updates?

Resolve parsing versus semantic validation versus preparation versus playback, resource/port ownership, the execution model and cancellation, live-command submission, observable state and diagnostics, and dry-run output. Define which failures occur before MIDI output and which can occur during playback. Specify the smallest CLI surface for loading and testing documents. Decide module/crate placement using the existing library without implementing it.

## Blocked by

- [Choose the structured score document shape](xd-dsl-0004-document-shape.md)
- [Define sound-parameter and patch control semantics](xd-dsl-0007-sound-and-patches.md)
- [Define the boundary between scores and onboard sequences](xd-dsl-0008-onboard-sequencer.md)
- [Define arbitration between score automation and live changes](xd-dsl-0009-live-updates.md)
- [Define scheduling and MIDI-output guarantees](xd-dsl-0010-scheduler-contract.md)
