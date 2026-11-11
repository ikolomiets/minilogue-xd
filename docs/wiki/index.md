---
title: Index
category: meta
sources: []
related:
  - AGENTS.md
  - overview.md
  - glossary.md
  - log.md
updated: 2026-11-11
---

# Index

> This is the content catalog. Every wiki page is listed once, by category, with a one-line
> summary. Read this first when answering a question, then drill into pages. The schema and
> workflows are in [AGENTS.md](AGENTS.md); the change history is in [log](log.md).

## Start here

| Page | Summary |
|------|---------|
| [Overview](overview.md) | What the tool is, its four layers, the main data flows, and how to read this wiki for a Zig port. |
| [Glossary](glossary.md) | Canonical terms, one definition each, with a link to the defining page. |
| [Zig port — overview and roadmap](zig/port-overview.md) | Target shape (sans-I/O core, static buffers, OS MIDI backends) and milestones M0–M10. |

## Sources

Summaries of each raw source.

| Page | Summary |
|------|---------|
| [Rust crate](sources/rust-crate.md) | Workspace, dependencies, feature flags, module map, internal dependency graph, README drift, key commits. |
| [Korg MIDI Implementation rev 1.01](sources/korg-midi-implementation.md) | Where to get the official spec, how its sections map to wiki pages, and how reliable it is. |
| [Community references](sources/community-references.md) | The four community codebases used to cross-check the spec, and what each settles. |
| [Korg owner's manual](sources/korg-user-guide.md) | The in-repo Markdown manual: what it is good for and its caveats. |
| [Design and development documents](sources/design-docs.md) | `docs/design/` and `docs/dev/`: the project plan, the MIDI-export plan, ODM workflow, and planned-vs-implemented gaps. |
| [Wayfinder DSL](sources/wayfinder-dsl.md) | The score DSL and real-time adapter ticket map, with its domain vocabulary and open decisions. |

## Protocol

Device and wire facts that hold for any implementation language.

| Page | Summary |
|------|---------|
| [MIDI messages](protocol/midi-messages.md) | Channel, system common and real-time messages, bank select and program numbers. |
| [USB ports](protocol/usb-ports.md) | The XD's four USB MIDI ports, recommended pairs, platform naming, MIDI Route, and port discovery. |
| [Korg 7-bit codec](protocol/korg-7bit-codec.md) | NOTE 1 packing of 8-bit data into 7-bit SysEx bytes: algorithm, lengths, vectors. |
| [Parameter encodings](protocol/parameter-encodings.md) | 10-bit (CC63 + CC), 8-bit hi-res and 14-bit splits, and float ↔ integer mappings. |
| [CC map](protocol/cc-map.md) | Every Control Change: kind, TX values, RX bands, context-dependent CC96/CC103, receiver rules. |
| [NRPN map](protocol/nrpn-map.md) | NRPN address table, wire sequence and receiver state machine. |
| [SysEx framing](protocol/sysex-framing.md) | Korg exclusive header, every function ID, status codes and frame sizes. |
| [Device identity](protocol/device-identity.md) | Universal Identity Request and Korg Search Device, with both 15-byte replies. |
| [Poly chain](protocol/poly-chain.md) | Note on/off SysEx (`0x60`/`0x61`) between two chained units, including the 21-bit pitch. |
| [Tuning](protocol/tuning.md) | User scale, user octave, and MIDI Tuning Standard bulk and single-note messages. |
| [Program blob](protocol/program-blob.md) | Synth half of the 1024-byte program (offsets 0–155) and the four program SysEx messages. |
| [Sequencer blob](protocol/sequencer-blob.md) | Sequencer half (156–1023): header, motion slots, 16 step events, ARP fields. |
| [File formats](protocol/file-formats.md) | `.mnlgxdprog`/`.mnlgxdlib`/`.mnlgxdpreset` zip layout, manifests and a stored-zip writer. |
| [Global blob](protocol/global-blob.md) | The 63-byte TABLE 1 `GLOB` blob: every field, its encoding, and round-trip gaps. |
| [User modules](protocol/user-modules.md) | logue SDK module types, slots, TABLES 5–7, CRC and buffer sizing. |
| [Standard MIDI file](protocol/standard-midi-file.md) | The exact SMF Format 0 byte layout the crate writes. |
| [Errata](protocol/errata.md) | The single register of spec, hardware, community and Rust disagreements, with resolutions. |

## Design

How the Rust crate is built, and what to keep or drop.

| Page | Summary |
|------|---------|
| [Architecture](design/architecture.md) | Four layers, module dependencies, and the end-to-end data flows. |
| [Typed values](design/typed-values.md) | `U2`/`U4`/`U7`/`U14`/`I14` range-checked newtypes and their Zig equivalents. |
| [Errors](design/errors.md) | The `Error`/`SysexError` taxonomy and the proposed Zig error sets. |
| [Transport](design/transport.md) | `MidiOutput`/`MidiInput` traits, mocks, the midir backend, connections and input parsing. |
| [SysEx transactions](design/sysex-transactions.md) | Request/response matching, the stream assembler, timeouts and the Zig state machine. |
| [Stepped parameters](design/stepped-params.md) | The `SteppedParam` trait and every discrete-parameter enum, with value tables. |
| [Realtime controller](design/realtime-controller.md) | The immediate-send CC/NRPN/note API: every setter and its exact wire bytes. |
| [Builders](design/builders.md) | `PatchBuilder` and `SequenceBuilder`: defaults, clamping and gaps. |
| [Byte fidelity](design/byte-fidelity.md) | Raw-base overlay, what survives a round trip, and the in-place accessor design for Zig. |
| [MIDI file export](design/midi-file-export.md) | `MidiFileBuilder`: events, patch snapshot, CC legend and fixed-capacity Zig design. |
| [Testing strategy](design/testing-strategy.md) | How the crate is tested (unit, property, integration, CI, hardware), the golden-vector index, and the gaps. |
| [Examples](design/examples.md) | The nine examples: what each does, the APIs it uses, and which become acceptance demos. |

## Zig

How to rebuild the library in Zig with TigerStyle and zero dependencies.

| Page | Summary |
|------|---------|
| [Port overview](zig/port-overview.md) | Goals, non-goals and milestone plan for the port. |
| [Module layout](zig/module-layout.md) | Proposed source tree, import rules and the sans-I/O boundary. |
| [TigerStyle mapping](zig/tigerstyle-mapping.md) | Rust idiom → Zig/TigerStyle translation table. |
| [Dependency elimination](zig/dependency-elimination.md) | How each Rust crate is replaced by the Zig std library or an OS API. |
| [Platform MIDI](zig/platform-midi.md) | CoreMIDI, ALSA rawmidi and WinMM bindings, plus the SPSC input ring. |
| [Memory budget](zig/memory-budget.md) | Every comptime size and limit, the two build profiles, static totals and timing. |
| [Test plan](zig/test-plan.md) | Test tiers, milestone gates, vector file format and CI matrix for the port. |

## Meta

| Page | Summary |
|------|---------|
| [AGENTS.md](AGENTS.md) | Wiki schema: layers, page format, conventions and workflows. |
| [Log](log.md) | Append-only record of ingests, queries and lint passes. |
