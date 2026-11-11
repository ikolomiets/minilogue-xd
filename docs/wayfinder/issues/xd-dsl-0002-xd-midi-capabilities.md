---
{
  "id": "xd-dsl-0002",
  "title": "Establish XD MIDI capabilities and playback constraints",
  "status": "closed",
  "labels": [
    "wayfinder:research"
  ],
  "parent": "xd-dsl-0001",
  "assignee": "ikolomiets",
  "mode": "AFK",
  "blocked_by": [],
  "worker": "xd_midi_research",
  "research_branch": "research/xd-midi-capabilities",
  "resolution": "../comments/xd-dsl-0002/0002-resolution.md"
}
---

# Establish XD MIDI capabilities and playback constraints

## Question

Which sound, patch, and onboard-sequencer operations are actually MIDI-addressable on the Minilogue XD, how does this repository expose them, and which constraints affect performing them during host-driven playback?

Use official Korg documentation and repository source. Produce a cited capability inventory grouped by CC, NRPN, and SysEx, distinguishing verified behavior, documented behavior, library gaps, and facts requiring hardware measurement. Cover analog and digital sound parameters, effects, patch loading versus stored writes, onboard sequencing, message grouping, voice limits, clock, and transport bandwidth. Do not infer completeness from README counts or invent latency guarantees. Identify relevant known errata. Global settings, tuning-table management, and user-module management are outside the chosen control scope; explain only prerequisites they impose.

Record factual findings and unresolved evidence gaps, without selecting the DSL's policy or syntax.

## Activity

- [Research branch context](../comments/xd-dsl-0002/0001-research-context.md)
- [Research resolution and evidence](../comments/xd-dsl-0002/0002-resolution.md)
