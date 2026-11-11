---
{
  "id": "xd-dsl-0009",
  "title": "Define arbitration between score automation and live changes",
  "status": "open",
  "labels": [
    "wayfinder:grilling"
  ],
  "parent": "xd-dsl-0001",
  "assignee": null,
  "mode": "HITL",
  "blocked_by": [
    "xd-dsl-0005",
    "xd-dsl-0007"
  ]
}
---

# Define arbitration between score automation and live changes

## Question

When a Rust caller changes a sound parameter during prepared-score playback, how does that change interact with the score's current and future automation?

Resolve the live command model, target identity, acknowledgment/error meaning, effective time, ordering, override duration, resumption of score automation, interpolation, and concurrent callers. Stress-test a live cutoff change during a ramp, two updates to the same parameter, patch loading during automation, and updates arriving after playback stops. Notes and score structure are not live-editable in the selected scope.

## Blocked by

- [Define musical time and note semantics](xd-dsl-0005-musical-semantics.md)
- [Define sound-parameter and patch control semantics](xd-dsl-0007-sound-and-patches.md)
