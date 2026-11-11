---
{
  "id": "xd-dsl-0013",
  "title": "Choose the evidence standard for disputed MIDI encodings",
  "status": "open",
  "labels": [
    "wayfinder:grilling"
  ],
  "parent": "xd-dsl-0001",
  "assignee": null,
  "mode": "HITL",
  "blocked_by": [
    "xd-dsl-0002",
    "xd-dsl-0006"
  ]
}
---

# Choose the evidence standard for disputed MIDI encodings

## Question

When official Korg documentation, this library, and device/file evidence disagree about an in-scope MIDI mapping, what evidence is sufficient to specify the adapter's behavior?

The capability research has surfaced concrete conformance discrepancies. Decide the target firmware/device baseline, precedence of evidence, reproducible fixtures or captures required to resolve conflicts, how unresolved mappings are exposed to callers, and what prevents an unsupported claim of complete control coverage. Distinguish a specification decision from the later implementation work needed to reconcile library code. If hardware observation is necessary, create a bounded task or research ticket after the environment and access are known; do not resolve the discrepancy by preference or guesswork.

## Blocked by

- [Establish XD MIDI capabilities and playback constraints](xd-dsl-0002-xd-midi-capabilities.md)
- [Bound the supported performance environment](xd-dsl-0006-performance-environment.md)
