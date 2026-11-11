---
title: Overview — minilogue-xd design wiki
category: meta
sources:
  - README.md
  - crates/minilogue-xd/src/lib.rs
  - docs/design/06-final/0001-minilogue-xd-rust-library-project-plan.md
  - CONTEXT.md
related:
  - index.md
  - design/architecture.md
  - zig/port-overview.md
  - protocol/errata.md
updated: 2026-11-11
---

# Overview — minilogue-xd design wiki

> `minilogue-xd` is a Rust library that drives a **Korg minilogue xd** synthesizer (keyboard or
> desktop module) over USB/DIN MIDI. It covers every message in the Korg MIDI Implementation
> rev 1.01: notes, CC, NRPN, SysEx program/global dumps, the 16-step sequencer and motion data,
> tuning tables, and logue SDK user-module management. This wiki explains *how and why* it is
> designed that way, so it can be **re-implemented in Zig under TigerStyle with zero
> dependencies**.

## What the tool does

| Capability | Wire mechanism | Wiki page |
|------------|----------------|-----------|
| Play notes, bend, aftertouch, transport | Channel and realtime messages | [MIDI messages](protocol/midi-messages.md) |
| Turn every knob in real time | CC (7-bit, 10-bit via CC63, stepped) and NRPN | [parameter encodings](protocol/parameter-encodings.md), [CC map](protocol/cc-map.md), [NRPN map](protocol/nrpn-map.md), [realtime controller](design/realtime-controller.md) |
| Read/write patches (programs) | SysEx dump of a 1024-byte blob, 7-bit packed | [program blob](protocol/program-blob.md), [sequencer blob](protocol/sequencer-blob.md), [SysEx framing](protocol/sysex-framing.md) |
| Read/write global settings | SysEx dump of a 63-byte blob | [global blob](protocol/global-blob.md) |
| Microtuning | User scale/octave dumps, MIDI Tuning Standard | [tuning](protocol/tuning.md) |
| Manage oscillator/effect plug-ins | logue SDK user slot SysEx with CRC-32 | [user modules](protocol/user-modules.md) |
| Identify the device, poly chain | Universal SysEx, Korg search | [device identity](protocol/device-identity.md), [poly chain](protocol/poly-chain.md) |
| Librarian files | Zip archives of program blobs | [file formats](protocol/file-formats.md) |
| Export performances | Standard MIDI File | [SMF](protocol/standard-midi-file.md), [MIDI file export](design/midi-file-export.md) |
| Compose patches/sequences in code | Builders over the blobs | [builders](design/builders.md) |

## How it is built (Rust)

The crate has four layers: foundation, parameters, SysEx, and high-level API. Encoding and
decoding are pure. Only the transport, the real-time controller, and the SysEx transaction
touch I/O. See [architecture](design/architecture.md) and the
[source summary](sources/rust-crate.md). It is about 11k lines of implementation plus about 10k
lines of tests (973 unit tests plus property tests). See [testing strategy](design/testing-strategy.md).

## Key numbers

| Thing | Value | Page |
|-------|-------|------|
| Korg SysEx header | `F0 42 3g 00 01 51 <func> … F7` (g = global channel 0–15) | [SysEx framing](protocol/sysex-framing.md) |
| Program blob | 1024 bytes (synth 0–155, sequencer 156–1023); 1171 bytes after 7-bit packing | [program blob](protocol/program-blob.md) |
| Preset program blob (fw ≥ 2.10 files) | 448 bytes | [file formats](protocol/file-formats.md), [errata](protocol/errata.md) |
| Global blob | 63 bytes; 72 packed | [global blob](protocol/global-blob.md) |
| User scale / octave | 384 / 36 bytes | [tuning](protocol/tuning.md) |
| Programs | 500 (5 banks × 100) | [program blob](protocol/program-blob.md) |
| 10-bit CC | CC63 = low 3 bits first, then CC*n* = high 7 bits | [parameter encodings](protocol/parameter-encodings.md) |
| Sequencer | 16 steps, 8 notes/step, 4 motion slots | [sequencer blob](protocol/sequencer-blob.md) |
| USB ports | OUT: SOUND, MIDI OUT; IN: KBD/KNOB, MIDI IN | [USB ports](protocol/usb-ports.md) |

## Where the truth lives

Hardware behavior beats the spec, and the spec beats the Rust code. The Korg document has known
errors: the bank-select encoding, NRPN addresses, the 336- vs 1024-byte dump diagram, and
preset blob sizes. All known discrepancies are collected on [errata](protocol/errata.md), with
provenance tags `[hw]`, `[spec]`, `[rust]`, and `[community]`. See [schema](AGENTS.md#conventions).

## The Zig port in one paragraph

The Zig port keeps the layering and the pure codecs, but tightens them into a **sans-I/O core**.
Every builder writes into a fixed buffer sized by a comptime maximum. Every parser reads a
slice. The SysEx transaction becomes a state machine fed bytes and `now_ns`. MIDI I/O moves to
three thin OS backends that push bytes into a bounded ring buffer, which the program polls at
its own pace. `midir`, `zip`, `crc32fast`, `thiserror`, and `proptest` are replaced by the Zig
std plus about 1k lines of local code. Start at [port overview](zig/port-overview.md).

## How to use this wiki

- **Humans:** start with [index](index.md), then [architecture](design/architecture.md), then
  the protocol pages for whatever you are porting.
- **LLM agents:** read [AGENTS.md](AGENTS.md) (schema and workflows) and [index](index.md)
  first. Append to [log](log.md) after any ingest or lint.

## Sources

- `README.md` — feature list, examples, community references.
- `crates/minilogue-xd/src/lib.rs:1-28` — crate summary and layer diagram.
- `docs/design/06-final/0001-minilogue-xd-rust-library-project-plan.md` — original plan.
- `CONTEXT.md` — domain language for the planned score/DSL work.
