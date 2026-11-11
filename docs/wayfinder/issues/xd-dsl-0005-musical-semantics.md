---
{
  "id": "xd-dsl-0005",
  "title": "Define musical time and note semantics",
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

# Define musical time and note semantics

## Question

What exact musical meanings must the score preserve for notes, chords, rests, rational beat durations, meter, tempo changes, ties, tuplets, velocity, and timed synth automation?

Resolve the beat unit, absolute versus relative positions, pitch spelling and octave convention, tempo-change interpretation, tie and tuplet behavior, note duration versus gate, simultaneous events, and overlapping voices. Stress-test a tempo change inside a sustained note, a tuplet across a barline, tied versus retriggered notes, and overlapping identical pitches. Determine which expressive distinctions belong in the document even when MIDI cannot preserve them. The initial scope does not require generative transformations or microtonal notation.
