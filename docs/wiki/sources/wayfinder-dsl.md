---
title: Wayfinder — XD control DSL and real-time adapter
category: source
sources:
  - docs/wayfinder/README.md
  - docs/wayfinder/dependencies.md
  - docs/wayfinder/issues/xd-dsl-0001-xd-control-dsl.md
  - docs/wayfinder/issues/xd-dsl-0002-xd-midi-capabilities.md
  - docs/wayfinder/issues/xd-dsl-0003-structured-score-formats.md
  - docs/wayfinder/issues/xd-dsl-0004-document-shape.md
  - docs/wayfinder/issues/xd-dsl-0005-musical-semantics.md
  - docs/wayfinder/issues/xd-dsl-0006-performance-environment.md
  - docs/wayfinder/issues/xd-dsl-0007-sound-and-patches.md
  - docs/wayfinder/issues/xd-dsl-0008-onboard-sequencer.md
  - docs/wayfinder/issues/xd-dsl-0009-live-updates.md
  - docs/wayfinder/issues/xd-dsl-0010-scheduler-contract.md
  - docs/wayfinder/issues/xd-dsl-0011-adapter-api.md
  - docs/wayfinder/issues/xd-dsl-0012-acceptance.md
  - docs/wayfinder/issues/xd-dsl-0013-encoding-evidence.md
  - docs/wayfinder/comments/xd-dsl-0002/0001-research-context.md
  - docs/wayfinder/comments/xd-dsl-0002/0002-resolution.md
  - docs/wayfinder/comments/xd-dsl-0003/0001-research-context.md
  - docs/wayfinder/comments/xd-dsl-0003/0002-resolution.md
  - docs/wayfinder/research/xd-midi-capabilities.md
  - docs/wayfinder/research/structured-score-formats.md
  - CONTEXT.md
related:
  - glossary.md
  - design/architecture.md
  - design/realtime-controller.md
  - design/sysex-transactions.md
  - design/examples.md
  - protocol/errata.md
  - protocol/nrpn-map.md
  - protocol/midi-messages.md
  - protocol/sequencer-blob.md
  - zig/port-overview.md
  - sources/design-docs.md
  - sources/korg-midi-implementation.md
updated: 2026-10-11
---

# Wayfinder — XD control DSL and real-time adapter

> `docs/wayfinder/` is a local Markdown issue tracker. It charts the decisions still needed
> before a **structured score format (DSL)** and a **host-clocked real-time adapter** can be
> specified for the XD. Nothing is implemented.
>
> Of 13 tickets, two research tickets are closed:
>
> - 0002 produced an XD capability inventory with spec-versus-code discrepancies.
> - 0003 compared JSON, YAML, MusicXML, and MNX representations of exact musical time.
>
> Ten decision tickets, which need a human, remain open. The open tickets define what the core
> library must eventually provide: timestamped output, atomic multi-message parameter groups,
> clock and transport output, bandwidth accounting, and an evidence-graded parameter map.
> `CONTEXT.md` defines the domain vocabulary (Score, Patch, Live parameter change, …).

## Facts

### Status of these files

At the wiki baseline (`361d018`), `docs/wayfinder/` and `CONTEXT.md` are **untracked** in git.
They exist in the working tree only. Both research reports name `361d018` as their source
baseline (`research/xd-midi-capabilities.md:3`; `comments/xd-dsl-0003/0001-research-context.md`).

### Tracker mechanics (`docs/wayfinder/README.md:1-38`)

**Issue files.** Issues are `issues/xd-dsl-NNNN-<slug>.md`, with front matter that is JSON
inside `---` fences, so it is also valid YAML (`:11`).

| Field | Meaning |
|-------|---------|
| `id` | Immutable, e.g. `xd-dsl-0007`. Links always use the *title* |
| `title` | Human-facing name |
| `status` | `open` or `closed` |
| `labels` | One of `wayfinder:map`, `:research`, `:prototype`, `:grilling`, `:task` (`:12`) |
| `parent` | The map id (`xd-dsl-0001`); the map has `null` |
| `assignee` | Claimed by `ikolomiets`; `null` = available |
| `mode` | `map`, `AFK` (an agent can resolve it unattended), or `HITL` (needs the user live, `:20`) |
| `blocked_by` | Authoritative dependency list; mirrored as links under "Blocked by" (`:14`) |
| `worker` | Agent/session that did the work (closed tickets only) |
| `resolution` | Path of the resolution comment (closed tickets only) |
| `research_branch` | Branch holding the research evidence (closed research tickets) |

**Rules:**

- **Claim:** set `assignee`, then `worker`. Re-read the issue before editing it (`:15`).
- **Resolve:** append `comments/<id>/NNNN-resolution.md` with evidence links, close the issue,
  and add a one-line gist to the map's "Decisions so far" (`:17`).
- **Research:** isolated git worktrees on `research/<name>` branches. A copy of the report is
  kept in `research/` (`:18`).
- **Frontier:** open children of `xd-dsl-0001` that have no assignee and whose `blocked_by`
  entries are all closed. A Python standard-library query lists them (`:22-38`).
- **Dependency graph:** `dependencies.md:5-51` draws it in Mermaid, with 0002 and 0003 shaded
  as resolved.

### The map (`issues/xd-dsl-0001-xd-control-dsl.md`)

**Destination** (`:20`). Reach an implementation-ready DSL specification and real-time MIDI
adapter contract, including worked musical examples and acceptance criteria.

**Standing scope** (`:24-34`). The user confirmed these during charting.

| Aspect | Decision |
|--------|----------|
| Authors | Primarily AI systems and applications generating structured documents; syntax undecided |
| Playback | Prepared scores play while callers make **live sound-parameter changes**; notes and structure are not live-editable |
| Musical scope | Notes, chords, rests, rational beat durations, meter, tempo changes, ties, tuplets, velocity, timed synth automation |
| Control scope | All MIDI-addressable sound parameters (analog, multi-engine, effects), plus patches and onboard sequencing |
| Integration | A Rust library API plus a CLI for loading and testing documents |
| Timing | **The computer owns timing.** Sending MIDI clock to the XD is optional; following an external clock is excluded |
| Constraint | Planning only: no parser, scheduler, or adapter implementation during the map |

The map also observes that the existing `RealtimeController` "emits immediately". Onboard
`SequenceBuilder` and host playback are separate paths (`:33`).

**Out of scope** (`:48-54`):

- implementing the parser, scheduler, CLI, or adapter;
- live restructuring, generative programming, and mandatory microtonal notation;
- a network service, a GUI, engraving, and audio synthesis;
- following an external clock;
- global settings administration, tuning-table management, and user-module installation.
  Sound parameters of installed modules stay in scope.

### Issues and status

The status column comes from a front-matter parse, so it is exact.

| ID | Title | Type / mode | Status | Blocked by | Core question |
|----|-------|-------------|--------|------------|---------------|
| 0001 | Find the way to an XD control DSL and real-time adapter contract | map | open | — | The index of decisions |
| 0002 | Establish XD MIDI capabilities and playback constraints | research / AFK | **closed** | — | What can be controlled, and with which messages and limits? |
| 0003 | Establish structured-document and musical-time representation options | research / AFK | **closed** | — | Which representations can carry exact musical time? |
| 0004 | Choose the structured score document shape | prototype / HITL | open | 0003, 0005 | Concrete format, organization, validation, versioning. Compare throwaway examples, including one invalid example with its diagnostic |
| 0005 | Define musical time and note semantics | grilling / HITL | open (frontier) | — | Beat unit, absolute vs relative positions, pitch spelling, tempo change inside a note, tuplets across barlines, tie vs retrigger, overlapping identical pitches |
| 0006 | Bound the supported performance environment | grilling / HITL | open (frontier) | — | OS, USB vs DIN, device count, exclusive ports, DAW role, which hardware can verify timing |
| 0007 | Define sound-parameter and patch control semantics | grilling / HITL | open | 0002, 0013 | Units vs normalized values; validation vs clamping; live change vs patch preparation vs edit-buffer load vs stored-program select vs persistent write; SysEx-only operations; **no raw-MIDI escape hatch** |
| 0008 | Define the boundary between scores and onboard sequences | grilling / HITL | open | 0002, 0005, 0007 | Onboard sequence as resource and/or compile target; transport and host clock; ownership; motion lanes; over-limit requests must not be silently reduced |
| 0009 | Define arbitration between score automation and live changes | grilling / HITL | open | 0005, 0007 | Command model, target identity, ack meaning, effective time, override duration, resumption, interpolation, concurrent callers |
| 0010 | Define scheduling and MIDI-output guarantees | grilling / HITL | open | 0002, 0005, 0006, 0008, 0009 | Monotonic clock, tempo transitions, clock output, ordering at identical deadlines, **indivisible CC/NRPN groups**, SysEx interaction, coalescing, late events, bandwidth admission, stop/pause/resume, note cleanup, device loss, latency/jitter targets |
| 0011 | Define the Rust adapter and CLI boundary | grilling / HITL | open | 0004, 0007, 0008, 0009, 0010 | Parse → validate → prepare → play; port ownership; execution model and cancellation; dry run; crate placement |
| 0012 | Define the specification handoff and acceptance evidence | grilling / HITL | open | 0004–0011 | Traceability; worked valid/invalid documents; expected event timelines and MIDI transactions; **mock-output checks vs required hardware experiments** |
| 0013 | Choose the evidence standard for disputed MIDI encodings | grilling / HITL | open | 0002, 0006 | Firmware baseline, evidence precedence, reproducible fixtures and captures, exposing unresolved mappings, preventing unsupported "complete coverage" claims |

**Frontier now:** 0005 and 0006. They are open, unblocked, and unassigned.

### Resolutions (`comments/`)

| Ticket | Worker | Evidence | Conclusion |
|--------|--------|----------|------------|
| 0002 | `xd_midi_research` | branch `research/xd-midi-capabilities`, commit `3a78b2e` | Control spans channel messages and program SysEx. Edit-buffer loads, stored writes, and onboard sequences have distinct effects. Doc/code discrepancies are not proven hardware bugs. Wire occupancy constrains scheduling but proves no latency. Created 0013 for the evidence policy |
| 0003 | `score_format_research` | branch `research/structured-score-formats`, commit `8e8114d` | JSON + schema can express the scope, but cross-event checks need a semantic validator. YAML needs a loader profile. MusicXML separates notation from sound; MNX is a draft. No syntax chosen |

Both research worktrees lived under `/private/tmp/…` and are not in the repository.

### Research: XD MIDI capabilities (`research/xd-midi-capabilities.md`)

**Inventory** (`:17-30`). Eight groups:

- played music;
- analog sound (CC);
- digital and effects (CC);
- extended program sound (NRPN);
- patch selection (bank select + program change);
- patch transfer and persistence (SysEx `0x10`/`0x40`, `0x1C`/`0x4C`);
- onboard sequence (program data only; no per-step message);
- onboard transport (clock, start, continue, stop, SPP).

See [MIDI messages](../protocol/midi-messages.md) and
[sequencer blob](../protocol/sequencer-blob.md).

**Ordering** (`:32-36`). Hi-res CC means CC63 (low 3 bits), then the parameter CC (high 7 bits).
NRPN means CC99/98, an optional CC63, then CC6. **Competing writers must not interleave** these
groups on one channel, and the controller has no timestamps or queue.

**Hard constraints** (`:38-46`):

- 4 voices, 16 steps, and 4 motion lanes. Eight note slots per step do **not** mean 8-voice
  polyphony.
- Onboard tempo is 10.0–300.0 BPM.
- "Auto" clock falls back to internal timing.
- DIN wire occupancy at 3,125 bytes/s (0.32 ms/byte):

  | Message | Bytes | Time |
  |---------|------:|-----:|
  | Note | 3 | 0.96 ms |
  | Hi-res CC pair | 6 | 1.92 ms |
  | NRPN | 12 | 3.84 ms |
  | Program dump (1,024 → 1,171 packed + framing) | ≈1,181 | ≈377 ms |

- An ACK proves neither audible onset nor persistence.

**Discrepancies** (`:48-58`), re-checked for this page against Korg MIDI Impl rev 1.01 and the
code:

| Claim in report | Re-check result |
|-----------------|-----------------|
| NRPN names `0x01`–`0x0C`, voice mode `0x10`, noise `0x11` [spec] vs `0x00`–`0x0B`, `0x0C`, `0x0D` [rust]; "do not apply a universal offset" | **Confirmed.** The offset is not uniform: +1 for names (`0x00`–`0x0B`→`0x01`–`0x0C`) and LFO sync/target (`0x33`–`0x35`→`0x34`–`0x36`); +4 for voice mode and multi select/shape/shift (`0x0C`–`0x15`→`0x10`–`0x19`); +10 for bend/joystick (`0x16`–`0x1B`→`0x20`–`0x25`); +12 for CV (`0x1C`–`0x20`→`0x28`–`0x2C`); +2 for velocity…portamento BPM sync (`0x36`–`0x3C`→`0x38`–`0x3E`); 0 for `0x30`–`0x32`, `0x3F`, `0x40`–`0x45`, `0x48`–`0x4D`, `0x50`, `0x7F`. The code's `0x51` (after-touch assign) is not in the spec table. See [NRPN map](../protocol/nrpn-map.md) |
| Bank select: spec CC0 = 0 and CC32 = bank 0–4; controller sends CC0 = bank and CC32 = slot | **Confirmed** (§1-1/§2-1; `controller.rs:460-469`). No [hw] capture |
| CC93/94 (TXT) vs 94/95 (E8 manual); CC7/11 absent from the TXT receive table | **Confirmed** for the TXT; the code uses 93/94 |
| CC59 listed "without the high-resolution footnote" | **Half right.** §1-1 (transmit) has no *1-4, but §2-1 (receive) marks CC59 with *5-4 (10-bit). The code's 10-bit handling matches receive |
| Dump diagrams say 336 bytes vs TABLE 2's 1024; program-number bit annotations disagree | **Confirmed.** 1024 is [hw]-verified (`1c16566`); see [design docs](design-docs.md) |

**New question** (`:60-64`). Which firmware-specific evidence gates a "full MIDI control"
claim? This became ticket 0013.

### Research: structured score formats (`research/structured-score-formats.md`)

**JSON** (`:9-17`):

- Object member order carries no meaning.
- Duplicate keys behave inconsistently across parsers.
- Integers are exact only within ±(2^53 − 1), and there is no native fraction type.
- JSON Schema 2020-12 cannot check cross-event arithmetic, so a **second, semantic validation
  stage** is required. It covers denominators and overflow, measure totals, ID uniqueness, tie
  targets, and automation conflicts.

**YAML 1.2.2** (`:19-23`). Usable only with a loader profile restricted to JSON-compatible
trees.

**Exact durations** (`:25-38`). Four candidates, with no choice made:

- an integer pair;
- a `"n/d"` string;
- integer divisions (MusicXML-style);
- a written note value plus tuplet context.

Keep rationals through compilation, and round once when mapping to host deadlines.

**Notation** (`:40-48`):

- MusicXML separates `tie` (sound) from `tied` (notation), and spells pitch as
  step/alter/octave. A MIDI number alone loses the enharmonic spelling.
- MNX ties reference note IDs.

**Maturity** (`:50-56`). MusicXML 4.0 is a Community Group Report; MNX is a 1.0 draft. Serde
ignores unknown fields by default, and `deny_unknown_fields` conflicts with `flatten`. The
crate has no serde, schema, YAML, or rational-number dependency.

### Domain vocabulary (`CONTEXT.md:7-33`)

| Term | Definition (summarized) | Avoid |
|------|-------------------------|-------|
| **Score** | A prepared description of music to perform, with musical timing and planned sound changes | "patch", "program" |
| **Patch** | A configuration of the synth's sound parameters, distinct from the music played with it | "score" |
| **Live parameter change** | A sound-parameter adjustment supplied during score playback; it does not rewrite the score | "live score editing" |
| **Sound parameter** | Any setting that shapes the sound: oscillators, multi-engine, filter, modulation, effects | "analog parameter" for the whole surface |
| **Onboard sequence** | A sequence held and played by the XD's built-in sequencer | "score" for it |
| **Score automation** | Sound-parameter changes planned at musical times inside a score | "live parameter change" for those |
| **Host clock** | The computer's timing authority for playback; it may drive the XD through outgoing MIDI clock | "external clock" |

## Invariants

These are the constraints the planned adapter must honour. They come from the map scope and
the research.

- The computer is the only timing authority. The XD may *follow* host MIDI clock. The adapter
  never follows the XD's clock in v1.
- A multi-message parameter write (hi-res CC pair, 4-CC NRPN) is atomic on the wire for its
  channel. No other message for that channel is sent between its parts.
- Edit-buffer load (`0x40`), stored write (`0x4C`), and program select (bank + PC) are
  different operations with different persistence. They must not be conflated.
- Requests that exceed onboard limits (16 steps, 4 motion lanes, 4 voices) produce an explicit
  error, never a silent reduction (0008).
- Every parameter mapping carries an evidence status. Disputed mappings are exposed, not
  guessed (0013).

## Errata & gotchas

- The capability report's CC59 statement is half right; see the re-check table.
- `quintal`'s ordering of events at equal times (label, then cutoff, then note-off, then
  note-on) is only a precedent, not a decided rule. It is input to 0010's "ordering at identical
  deadlines". See [examples](../design/examples.md).
- Comments reference worktrees under `/private/tmp/…`. Those paths are ephemeral and will not
  resolve.
- The tracker and `CONTEXT.md` are untracked, so a fresh clone does not contain them.

## Rust design

The adapter would sit *above* the current crate (0011 decides crate placement). Today the
crate offers immediate-send `RealtimeController`, blocking `SysexTransaction`, and offline
`MidiFileBuilder`. None of them provides scheduling, timestamps, or arbitration. See
[realtime controller](../design/realtime-controller.md) and
[SysEx transactions](../design/sysex-transactions.md).

### What the future adapter needs from the core library

| Need | Driving ticket | Core-library implication |
|------|----------------|--------------------------|
| Timestamped, non-blocking output | 0010, 0011 | Encode messages into caller-supplied buffers. Let the caller (scheduler) decide *when* to write. Separate encoding from the send path |
| Atomic message groups | 0010, research `:32-36` | Encoders return a whole group (CC63 + CC, or CC99/98/63/6) as one indivisible unit |
| Host clock and transport | 0008, 0010 | Encoders for `F8`, `FA`, `FB`, `FC`, and SPP. A monotonic-clock abstraction; a tempo map from rational beats to nanoseconds |
| Bandwidth admission | 0010, research `:44` | Per-message wire-size functions and a DIN byte-time model (0.32 ms/byte) |
| SysEx without stalling realtime | 0010 | A transaction state machine driven by `poll(now)` and fed bytes, not a blocking `receive(timeout)`. Clock may need to interleave during dumps |
| Note cleanup and device loss | 0010 | Active-note tracking (fixed 128-bit set per channel); all-notes-off; distinct errors for a disconnected port |
| Distinct patch operations | 0007 | Separate APIs: edit-buffer load, stored write (with slot), program select, live CC/NRPN |
| Parameter identity and arbitration | 0009 | A stable parameter ID per sound parameter, so score automation and live changes can be compared and coalesced |
| Evidence-graded mapping table | 0013 | Each CC/NRPN/blob mapping carries `spec`, `hw`, or `disputed`; unresolved ones can be refused |
| Onboard sequence as compile target | 0008 | A sequence builder with motion lanes and explicit over-limit errors |
| Dry run and simulation | 0011, 0012 | Mock output that records timestamped bytes; a simulated device for acceptance checks |

## Zig / TigerStyle notes

The Zig port should build the core so that each need above is cheap to add later:

- **Encoders write into `[]u8` buffers and return lengths.** Nothing sends. A group encoder
  (`encode_cc_10bit`, `encode_nrpn`) emits its whole group in one call.
- **Every timed operation takes an explicit `now_ns: u64`.** That covers transactions,
  receivers, and future schedulers. Nothing reads a clock internally, so all of them can be
  simulated deterministically ([testing strategy](../design/testing-strategy.md)).
- **Bound every queue.** For example: `events_max`, `active_notes` as a `[128]bool` or
  `u128` bit set per channel, and `pending_groups_max`. Overload is an explicit error
  (bandwidth admission), not unbounded growth.
- **Keep mappings in comptime tables with an `evidence` field.** The tests assert that no
  `disputed` entry is reachable from the "complete coverage" API without an explicit opt-in.
- Rational musical time (`numerator`/`denominator` as `u32`) belongs in the future adapter, not
  the core.

See [port overview](../zig/port-overview.md).

## Open questions

All of tickets 0004–0013 are open. Those that most affect the core library are:

- 0010: ordering at identical deadlines, and whether SysEx dumps may pause clock output.
- 0013: the evidence precedence that will settle the NRPN, bank-select, and voice-mode
  disputes.
- 0006: whether DIN must be supported. That decides whether the 0.32 ms/byte budget is binding.

## Sources

- `docs/wayfinder/README.md:1-38` — tracker rules and frontier query.
- `docs/wayfinder/dependencies.md:1-56` — dependency graph.
- `docs/wayfinder/issues/xd-dsl-0001-xd-control-dsl.md:16-54` — map destination, notes, decisions, out of scope.
- `docs/wayfinder/issues/xd-dsl-0004…0013` — the "Question" section of each.
- `docs/wayfinder/comments/xd-dsl-0002/*.md`, `docs/wayfinder/comments/xd-dsl-0003/*.md` — context and resolutions.
- `docs/wayfinder/research/xd-midi-capabilities.md:1-64`.
- `docs/wayfinder/research/structured-score-formats.md:1-69`.
- `CONTEXT.md:7-33` — domain terms.
- Korg MIDI Impl rev 1.01 §1-1, §2-1, *3 NRPN table, MIDI EXCLUSIVE FORMAT (2), (4), (5).
- `crates/minilogue-xd/src/controller.rs:460-469`, `crates/minilogue-xd/src/param/nrpn.rs:136-189`.
