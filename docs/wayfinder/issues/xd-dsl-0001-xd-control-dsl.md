---
{
  "id": "xd-dsl-0001",
  "title": "Find the way to an XD control DSL and real-time adapter contract",
  "status": "open",
  "labels": [
    "wayfinder:map"
  ],
  "parent": null,
  "assignee": null,
  "mode": "map",
  "blocked_by": []
}
---

# Find the way to an XD control DSL and real-time adapter contract

## Destination

Reach an implementation-ready DSL specification and real-time MIDI adapter contract for the Korg Minilogue XD, including worked musical examples and acceptance criteria. The map is complete when no decisions remain before implementation and the linked decisions establish the specification handoff.

## Notes

- Primary authors are AI systems and applications generating structured control documents; syntax and serialization remain undecided.
- Prepared scores play while callers can make live sound-parameter changes. Live editing of notes or score structure is excluded.
- Musical scope: notes, chords, rests, rational beat durations, meter, tempo changes, ties, tuplets, velocity, and timed synth automation.
- Control scope: all MIDI-addressable sound parameters, including analog oscillators, digital multi-engine and effects, plus patches and onboard sequencing.
- Integration: a Rust library API and a CLI for loading and testing documents.
- The computer owns playback timing; sending MIDI clock to the XD is optional. External-clock following is excluded from the first version.
- These scope choices were confirmed by ikolomiets during charting; they are standing preferences, not resolved child tickets.
- Follow `wayfinder`, `grilling`, and `domain-modeling` each decision session; also follow the skill associated with the ticket type. Read [Repository Guidelines](../../../AGENTS.md) and the [domain glossary](../../../CONTEXT.md).
- Use the [local Markdown tracker](../README.md). Claim a ticket as `ikolomiets` before work; record the working agent separately. Keep resolutions in comment files and index them here. Do not publish tracker content to GitHub unless requested.
- Existing `RealtimeController` emits immediately; onboard `SequenceBuilder` and host playback are different paths. Repository reconnaissance is context, not a verified full hardware capability inventory.
- Planning only: no parser, scheduler, or production adapter implementation during this map. Research and throwaway examples are supporting assets.

## Decisions so far

<!-- Append one named link and one-line gist per closed child ticket. -->

- [Establish structured-document and musical-time representation options](xd-dsl-0003-structured-score-formats.md): representation and validation tradeoffs are documented; syntax remains a human decision.

- [Establish XD MIDI capabilities and playback constraints](xd-dsl-0002-xd-midi-capabilities.md): capability groups and conformance gaps are documented; hardware behavior and timing remain unverified.

## Not yet specified

- Further decisions exposed by reviewing concrete combined score, patch, live-control, and onboard-sequencer scenarios; avoid predicting them before those scenarios exist.

## Out of scope

- Implementing or shipping the DSL parser, runtime scheduler, CLI, or adapter; this effort establishes their specification and contract.
- Live rewriting of musical structure, a generative programming language, and mandatory microtonal notation.
- A network service, graphical editor, notation engraving, and audio synthesis on the computer.
- Following external MIDI clock in the first version.
- General global-settings administration, tuning-table management, and user-module installation/management. Sound parameters of installed digital modules remain in scope.
