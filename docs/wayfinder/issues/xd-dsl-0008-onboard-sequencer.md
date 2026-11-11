---
{
  "id": "xd-dsl-0008",
  "title": "Define the boundary between scores and onboard sequences",
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
    "xd-dsl-0007"
  ]
}
---

# Define the boundary between scores and onboard sequences

## Question

How should prepared host scores and XD onboard sequences coexist without conflating their timing, note capacity, or patch state?

Decide whether onboard sequences are explicitly authored resources, compilation targets for compatible score fragments, or both. Resolve start/stop and host-clock behavior, ownership of notes and automation while the onboard sequencer runs, motion-sequence representation, and the response when requested music exceeds hardware limits. Do not silently reduce a general score to the onboard step format.

## Blocked by

- [Establish XD MIDI capabilities and playback constraints](xd-dsl-0002-xd-midi-capabilities.md)
- [Define musical time and note semantics](xd-dsl-0005-musical-semantics.md)
- [Define sound-parameter and patch control semantics](xd-dsl-0007-sound-and-patches.md)
