---
{
  "id": "xd-dsl-0006",
  "title": "Bound the supported performance environment",
  "status": "open",
  "labels": [
    "wayfinder:grilling"
  ],
  "parent": "xd-dsl-0001",
  "assignee": null,
  "mode": "HITL",
  "blocked_by": []
}
---

# Bound the supported performance environment

## Question

What environment must the first adapter contract support: operating systems, MIDI connection types, device count/routing, and access to hardware for acceptance testing?

The user has chosen a Rust library with a CLI and computer-owned timing, optionally emitting MIDI clock to the XD. Resolve single versus multiple devices, USB MIDI versus DIN expectations, device selection, exclusive versus shared port access, and whether a DAW participates without being the timing authority. Record which real hardware and connection can verify timing; do not promise hard real-time behavior before measurement.
