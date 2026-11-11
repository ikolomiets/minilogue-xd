---
title: Architecture of the Rust crate
category: design
sources:
  - crates/minilogue-xd/src/lib.rs
  - crates/minilogue-xd/src/transport.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - docs/design/06-final/0001-minilogue-xd-rust-library-project-plan.md
related:
  - overview.md
  - sources/rust-crate.md
  - zig/module-layout.md
  - zig/tigerstyle-mapping.md
updated: 2026-10-11
---

# Architecture of the Rust crate

> `minilogue-xd` has four layers: **foundation** (bytes, messages, codec, I/O),
> **parameters** (CC/NRPN maps and encodings), **SysEx** (frames, blobs, transactions), and the
> **high-level API** (controller, builders, MIDI-file export). Most of the protocol logic is
> *pure*: `build_*` and `parse_*` functions turn typed values into byte vectors and back. Only
> three components touch I/O: `transport`, `RealtimeController`, and `SysexTransaction`. This
> split is the most important thing to keep in the Zig port. Make it stricter, so the core does
> no I/O at all.

## Layers

```text
 Layer 4  High-level API   controller.rs (RealtimeController)   builder/ (PatchBuilder, SequenceBuilder)
                           midi_file.rs (MidiFileBuilder → SMF bytes)
 ───────────────────────────────────────────────────────────────────────────────────────────
 Layer 3  SysEx            sysex/frame.rs        Korg header, function IDs, ACK/NAK status
                           sysex/program/*       1024-byte program blob (synth + sequencer), files
                           sysex/global.rs       global blob        sysex/tuning.rs  scales, MTS
                           sysex/user_module.rs  logue SDK slots    sysex/identity.rs, poly_chain.rs
                           sysex/transaction.rs  request → response over transport (blocking)
 ───────────────────────────────────────────────────────────────────────────────────────────
 Layer 2  Parameters       param/enums.rs   stepped enums (SteppedParam)
                           param/encoding.rs 10-bit / 8-bit / 14-bit splits
                           param/cc.rs      CcParam + CcParamReceiver (CC63 pairing)
                           param/nrpn.rs    NrpnParam + NrpnReceiver (4-state FSM)
 ───────────────────────────────────────────────────────────────────────────────────────────
 Layer 1  Foundation       message/*  channel / realtime / common messages, U4/U7/U14/I14
                           codec.rs   Korg 7-bit ↔ 8-bit packing (NOTE 1)
                           transport.rs  MidiOutput / MidiInput traits, mocks, midir backend
                           device.rs  USB port names, discovery, cc_name
 Cross-cutting             error.rs   Error / SysexError (thiserror)
```

`crates/minilogue-xd/src/lib.rs:8-28` documents the same four layers. The real import graph is
on [rust crate source](../sources/rust-crate.md#internal-dependency-graph-non-test-code). It has
two back-edges (`error → sysex::frame`, `transport → device`) that the Zig port should remove.

## Design principles visible in the code

| Principle | Where | Keep in Zig? |
|-----------|-------|--------------|
| **Range-checked values at the boundary.** `U4`, `U7`, `U14`, `I14`, `TenBitParam`, `ProgramNumber`, `ProgramName` cannot hold out-of-spec values. | [typed values](typed-values.md) | Yes. Use native `u4`/`u7`/`u10`/`u14` plus asserts; check only where untrusted bytes enter. |
| **Typed enums for every stepped control.** Wire values come from per-enum tables with separate *transmit* values and *receive* ranges. | [stepped params](stepped-params.md) | Yes, as comptime tables. |
| **Pure encoders/decoders.** `build_*` → `Vec<u8>`, `parse_*(&[u8])` → typed value. No I/O, no global state. | all `sysex/*`, `message/*`, `codec.rs` | Yes. Write into caller-provided `[]u8` instead of returning `Vec`. |
| **Push parsers for multi-message protocols.** `CcParamReceiver::feed`, `NrpnReceiver::feed`, and `TenBitReceiver` take one CC and return `Option<Param>`. | [CC map](../protocol/cc-map.md), [NRPN map](../protocol/nrpn-map.md) | Yes. These are already sans-I/O state machines. |
| **Byte fidelity.** `ProgramData` keeps the raw 1024-byte blob and writes modelled fields over it, so unmodelled bits survive a round trip. | [byte fidelity](byte-fidelity.md) | Yes, and simpler: the blob *is* the storage, and accessors read/write fields in place. |
| **Transport as a trait.** `MidiOutput::send(&[u8])` and `MidiInput::receive(timeout) → Option<Vec<u8>>`. Mocks make every flow testable without hardware. | [transport](transport.md) | Yes, but invert control: the core never blocks on input. |
| **Blocking request/response.** `SysexTransaction` sends, then loops on `receive` until a full `F0..F7` frame arrives or a 5 s deadline passes. | [SysEx transactions](sysex-transactions.md) | Redesign as a state machine fed bytes and time. |
| **Fluent, by-value builders.** `PatchBuilder::new().vco1(..).filter(..).build()`. | [builders](builders.md) | No. Use struct literals with defaults and in-place setters. |
| **Feature-gated I/O.** `midi-io` and `file-formats` keep `midir`/`zip` optional. | [rust crate source](../sources/rust-crate.md#feature-flags) | Replace with comptime OS selection; there are no third-party deps to gate. |

## Core data types and sizes

| Type | Size | Notes | Page |
|------|------|-------|------|
| `ProgramData` | 1024-byte blob (`SynthParams` 0–155 and `SequencerParams` 156–1023) plus the retained raw blob | 7-bit encoded: 1171 bytes | [program blob](../protocol/program-blob.md), [sequencer blob](../protocol/sequencer-blob.md) |
| `GlobalParams` | 63-byte `GLOB`-tagged blob (TABLE 1) | 7-bit encoded: 72 bytes | [global blob](../protocol/global-blob.md) |
| `UserScale` / `UserOctave` | 128×3 = 384 / 12×3 = 36 bytes | | [tuning](../protocol/tuning.md) |
| `UserModuleInfo` / `UserSlotStatus` / `UserSlotData` | 9 / 32 / variable | CRC-32 on slot data | [user modules](../protocol/user-modules.md) |
| `SysexFrame` | channel, function ID, decoded payload | | [SysEx framing](../protocol/sysex-framing.md) |
| `MidiMessage` | enum over all channel, realtime, and common messages | | [MIDI messages](../protocol/midi-messages.md) |
| `CcParam` / `NrpnParam` | enums: one variant per parameter, carrying a typed value | | [CC map](../protocol/cc-map.md), [NRPN map](../protocol/nrpn-map.md) |

## Data flows

### 1. Real-time parameter change (host → XD, SOUND port)

```text
set_cutoff(0.6)                                 controller.rs
  → f32_to_10bit: clamp/validate, scale to 0..1023   → TenBitParam
  → send_cc(63, lsb)   then   send_cc(43, msb)       (CC63 carries the low bits first)
  → ControlChange::to_midi_bytes → [B0|ch, cc, v]
  → MidiOutput::send(&bytes)                     one send() per 3-byte message
```

Stepped parameters send one CC with the enum's transmit value. NRPN parameters send the CC
99/98/(63)/6 sequence. See [realtime controller](realtime-controller.md) and
[parameter encodings](../protocol/parameter-encodings.md).

### 2. Read a program (XD → host, SysEx over SOUND/KBD-KNOB)

```text
SysexTransaction::request_program(n)
  → build_program_request(ch, n)  = F0 42 3g 00 01 51 1C <n lo> <n hi> F7
  → MidiOutput::send               (SOUND port)
  → receive_sysex: loop MidiInput::receive(remaining) until F0…F7 complete
        drop bytes > 0xF7 (realtime), join fragments, skip non-SysEx, deadline 5 s
  → parse_program_dump: header check, program number, decode_7bit(payload)
  → ProgramData::from_bytes(1024 bytes): parse fields, retain raw blob
```

### 3. Write a program (host → XD)

```text
ProgramData::to_bytes: raw blob ⊕ modelled fields (to_bytes_with_base)
  → encode_7bit (1024 → 1171) → frame with function 0x4C (+ program number) or 0x40 (edit buffer)
  → send → receive_sysex → expect status frame 0x23 (DATA LOAD COMPLETED); 0x24..0x2F → NAK error
```

### 4. Files

`.mnlgxdprog` / `.mnlgxdlib` are zip archives of `Prog_NNN.prog_bin` entries (each 1024 bytes),
read and written by `sysex/program/file.rs` through the `zip` crate. See
[file formats](../protocol/file-formats.md).

### 5. Incoming control (XD → host)

The XD transmits knob moves as CCs (10-bit ones preceded by CC63) and NRPN sequences.
`parse_midi_bytes` turns bytes into `MidiMessage`. `CcParamReceiver` and `NrpnReceiver`
rebuild typed parameters from the stream. Nothing in the crate runs this loop for you; the
application owns it.

### 6. Standard MIDI File export

`MidiFileBuilder` collects timed events (notes, CCs, 10-bit CC pairs, a patch snapshot as CCs),
sorts them, and writes SMF Format 0. See [MIDI file export](midi-file-export.md).

## Concurrency and time

- The core is single-threaded and synchronous. `MidiOutput`/`MidiInput` require `Send` so an
  application *may* move them across threads.
- `MidirInput` receives bytes on a midir callback thread and forwards them over a
  `std::sync::mpsc` channel. `receive(timeout)` blocks on that channel. See
  [transport](transport.md).
- Time appears only in `SysexTransaction` (`Instant` deadline, `Duration` timeout) and in
  `midi_file.rs` (ticks and tempo). The controller sends immediately; it has no scheduler or
  timestamps. The [wayfinder DSL work](../sources/wayfinder-dsl.md) identifies scheduling as the
  next design problem.

## What is essential vs. incidental

**Essential**, so carry it over: the layering, pure encode/decode, typed ranges, comptime
parameter tables, the push-parser receivers, the CC63-first ordering, raw-blob byte fidelity,
fragment-tolerant SysEx assembly, ACK/NAK handling, and mock-driven tests.

**Incidental Rust idioms**, so drop them: `Vec<u8>` returns (heap per message), `thiserror`,
`TryFrom`/`Display` impls, by-value builders, generic `O: MidiOutput` parameters (use a comptime
type or a small vtable), `Box`/`String`, feature flags, and the umbrella crate.

## Zig / TigerStyle notes

- Make the core **sans-I/O**. Every protocol module takes `[]const u8` in and writes into
  `[]u8` out, with sizes known at comptime. Only `io/` touches the OS. Transactions become a
  state machine: `begin(request)` → `on_bytes(chunk)` → `tick(now_ns)` → `.pending | .done |
  .failed`. Tests then simulate fragmentation, interleaved clock, and timeouts deterministically,
  in the spirit of TigerBeetle's VOPR.
- Allocate statically. Every message has a known maximum wire length. The largest is a user
  slot data dump, then the program dump at about 1.2 KB. See [memory budget](../zig/memory-budget.md).
- Assert across layers in pairs: validate a program blob before `encode_7bit` and again after
  `decode_7bit`.
- The target module tree is on [module layout](../zig/module-layout.md). The idiom-by-idiom
  mapping is on [TigerStyle mapping](../zig/tigerstyle-mapping.md).

## Sources

- `crates/minilogue-xd/src/lib.rs:8-28` — four-layer diagram.
- `crates/minilogue-xd/src/transport.rs:13-35` — `MidiInput`/`MidiOutput` traits.
- `crates/minilogue-xd/src/controller.rs:84-149` — controller state and `send_cc`,
  `send_10bit_cc`, `send_stepped_cc`, `send_on_off_cc`, `send_nrpn` helpers.
- `crates/minilogue-xd/src/sysex/transaction.rs:55-216` — transaction struct, `receive_sysex`,
  `request_response`, `raw_request_response`, `send_and_wait_ack`.
- `docs/design/06-final/0001-minilogue-xd-rust-library-project-plan.md` — phase plan that
  produced the layering.
