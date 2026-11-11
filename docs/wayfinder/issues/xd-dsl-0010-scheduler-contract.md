---
{
  "id": "xd-dsl-0010",
  "title": "Define scheduling and MIDI-output guarantees",
  "status": "open",
  "labels": [
    "wayfinder:grilling"
  ],
  "parent": "xd-dsl-0001",
  "assignee": null,
  "mode": "HITL",
  "blocked_by": [
    "xd-dsl-0002",
    "xd-dsl-0005",
    "xd-dsl-0006",
    "xd-dsl-0008",
    "xd-dsl-0009"
  ]
}
---

# Define scheduling and MIDI-output guarantees

## Question

What observable timing, ordering, overload, and recovery guarantees should the host-driven adapter contract make?

Define musical-time conversion to a monotonic clock, tempo transitions, optional MIDI clock output and transport behavior, ordering at identical deadlines, indivisible CC/NRPN groups, SysEx interaction, automation sampling/coalescing, late-event policy, and bandwidth admission. Resolve stop/pause/resume semantics, note cleanup, device loss, and error visibility. Choose measurable latency/jitter acceptance targets against the agreed environment, distinguishing desired targets from hardware-verified guarantees.

## Blocked by

- [Establish XD MIDI capabilities and playback constraints](xd-dsl-0002-xd-midi-capabilities.md)
- [Define musical time and note semantics](xd-dsl-0005-musical-semantics.md)
- [Bound the supported performance environment](xd-dsl-0006-performance-environment.md)
- [Define the boundary between scores and onboard sequences](xd-dsl-0008-onboard-sequencer.md)
- [Define arbitration between score automation and live changes](xd-dsl-0009-live-updates.md)
