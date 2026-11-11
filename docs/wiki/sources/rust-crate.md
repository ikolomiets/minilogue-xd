---
title: Source — the minilogue-xd Rust crate
category: source
sources:
  - Cargo.toml
  - crates/minilogue-xd/Cargo.toml
  - crates/minilogue-xd/src/lib.rs
  - crates/minilogue/src/lib.rs
  - README.md
related:
  - design/architecture.md
  - zig/module-layout.md
  - zig/dependency-elimination.md
updated: 2026-11-11
---

# Source — the minilogue-xd Rust crate

> The primary raw source for this wiki: a Rust 2021 workspace (`minilogue-xd` 0.2.0 plus the
> `minilogue` umbrella crate) of about 10.9k lines of implementation and 10.4k lines of in-file
> unit tests, with 973 `#[test]` functions. It covers the Korg Minilogue XD MIDI Implementation
> rev 1.01 end to end. Every protocol, design, and Zig page in this wiki was compiled from it.

## Workspace

| Crate | Path | Role |
|-------|------|------|
| `minilogue-xd` | `crates/minilogue-xd/` | The library. All logic lives here. |
| `minilogue` | `crates/minilogue/` | Umbrella crate: `pub use minilogue_xd as xd;` (one line). Placeholder for future Minilogue variants. |

The umbrella crate has no design content. The Zig port needs no equivalent, so drop it. A Zig
package can expose `xd` as a namespace later if other synthesizers are added.

## Third-party dependencies and what they are used for

| Crate | Feature gate | Used in | Purpose | Zig replacement |
|-------|--------------|---------|---------|-----------------|
| `thiserror` 2 | always | `src/error.rs` | Derive `Display`/`From` for error enums | Native error sets; see [errors](../design/errors.md) |
| `bitflags` 2 | always | — | **Declared but unused** (no `bitflags!` anywhere in `src/`) | Nothing |
| `crc32fast` 1 | always | `src/sysex/user_module.rs` | CRC-32 for user slot data | `std.hash.Crc32`, or a local comptime table |
| `music-comp-mt` 0.5 | always | Re-exported as `minilogue_xd::theory`; used **only by examples** | Note/chord helpers | Small local note-name table; see [examples](../design/examples.md) |
| `midir` 0.10 | `midi-io` | `src/transport.rs`, `src/device.rs`, `src/connection.rs` | Cross-platform MIDI I/O | Direct OS APIs; see [platform MIDI](../zig/platform-midi.md) |
| `zip` 8.5 (deflate) | `file-formats` | `src/sysex/program/file.rs` | `.mnlgxdprog` / `.mnlgxdlib` containers | `std.zip` reader + a small writer; see [file formats](../protocol/file-formats.md) |
| `proptest` 1 | dev | 2 `proptest!` blocks | Property tests | Seeded `std.Random` loops / `std.testing.fuzz`; see [test plan](../zig/test-plan.md) |

The full replacement plan is on [dependency elimination](../zig/dependency-elimination.md).

## Feature flags

| Flag | Default | Actual effect in code |
|------|---------|-----------------------|
| `midi-io` | on | Gates `MidirOutput`, `MidirInput`, port discovery in `device.rs`, `MinilogueXd::connect`, and `Error::Midi*` variants. |
| `file-formats` | on | Gates `sysex::program::file` and the `Error::Zip`/`Error::Io` variants. |
| `std` | on | **Has no effect.** No `#![no_std]` or `cfg(feature = "std")` exists; the crate always needs `std` (`Vec`, `String`, `std::time`). The README `no_std` claim is aspirational. |

Citations: `crates/minilogue-xd/Cargo.toml:12-16`, `crates/minilogue-xd/src/device.rs:22`,
`crates/minilogue-xd/src/transport.rs:142-147`, `crates/minilogue-xd/src/error.rs:116-126`,
`crates/minilogue-xd/src/sysex/program/mod.rs:8`.

Zig has no need for feature flags here. Platform backends are selected at comptime from
`builtin.os.tag`. The protocol core has no I/O at all, so it is freestanding-capable by
construction.

## Module map

Implementation and test line counts come from the first `#[cfg(test)]` line of each file. Layer
numbers follow [architecture](../design/architecture.md).

| File (`crates/minilogue-xd/src/`) | Impl | Tests | Layer | Wiki pages |
|-----------------------------------|-----:|------:|:-----:|------------|
| `message/types.rs` | 221 | 169 | 1 | [typed values](../design/typed-values.md) |
| `message/channel.rs`, `realtime.rs`, `common.rs`, `mod.rs` | 882 | 602 | 1 | [MIDI messages](../protocol/midi-messages.md) |
| `codec.rs` | 104 | 152 | 1 | [7-bit codec](../protocol/korg-7bit-codec.md) |
| `transport.rs` | 331 | 83 | 1 | [transport](../design/transport.md) |
| `connection.rs` | 68 | 53 | 1 | [transport](../design/transport.md) |
| `device.rs` | 344 | 108 | 1 | [USB ports](../protocol/usb-ports.md), [CC map](../protocol/cc-map.md) (`cc_name`) |
| `error.rs` | 133 | 154 | all | [errors](../design/errors.md) |
| `param/mod.rs`, `param/enums.rs` | 649 | 726 | 2 | [stepped params](../design/stepped-params.md) |
| `param/encoding.rs` | 417 | 387 | 2 | [parameter encodings](../protocol/parameter-encodings.md) |
| `param/cc.rs` | 492 | 845 | 2 | [CC map](../protocol/cc-map.md) |
| `param/nrpn.rs` | 650 | 876 | 2 | [NRPN map](../protocol/nrpn-map.md) |
| `sysex/mod.rs`, `sysex/frame.rs` | 364 | 362 | 3 | [SysEx framing](../protocol/sysex-framing.md) |
| `sysex/identity.rs` | 209 | 243 | 3 | [device identity](../protocol/device-identity.md) |
| `sysex/poly_chain.rs` | 326 | 350 | 3 | [poly chain](../protocol/poly-chain.md) |
| `sysex/tuning.rs` | 469 | 461 | 3 | [tuning](../protocol/tuning.md) |
| `sysex/global.rs` | 356 | 476 | 3 | [global blob](../protocol/global-blob.md) |
| `sysex/enums.rs` | 274 | 233 | 3 | [global blob](../protocol/global-blob.md), [program blob](../protocol/program-blob.md) |
| `sysex/helpers.rs` | 76 | 183 | 3 | [byte fidelity](../design/byte-fidelity.md) |
| `sysex/program/mod.rs`, `synth.rs` | 1135 | 844 | 3 | [program blob](../protocol/program-blob.md) |
| `sysex/program/sequencer.rs` | 337 | 282 | 3 | [sequencer blob](../protocol/sequencer-blob.md) |
| `sysex/program/file.rs` | 149 | 115 | 3 | [file formats](../protocol/file-formats.md) |
| `sysex/user_module.rs` | 560 | 449 | 3 | [user modules](../protocol/user-modules.md) |
| `sysex/transaction.rs` | 367 | 674 | 3 | [SysEx transactions](../design/sysex-transactions.md) |
| `controller.rs` | 544 | 736 | 4 | [realtime controller](../design/realtime-controller.md) |
| `builder/*.rs` | 400 | 443 | 4 | [builders](../design/builders.md) |
| `midi_file.rs` | 923 | 378 | 4 | [MIDI file export](../design/midi-file-export.md), [SMF](../protocol/standard-midi-file.md) |
| `lib.rs` | 94 | 0 | — | [architecture](../design/architecture.md) |
| **Total** | **10874** | **10384** | | |

Integration tests (`crates/minilogue-xd/tests/`) and examples (`crates/minilogue-xd/examples/`)
are covered by [testing strategy](../design/testing-strategy.md) and
[examples](../design/examples.md).

## Internal dependency graph (non-test code)

Derived from `use crate::…` imports above each file's test module:

```text
error ──► sysex::frame (SysexStatus)          (a layering wart: layer 0 imports layer 3)
codec, device, message::types, param::encoding, sysex::helpers ──► error only
message::{channel,common,realtime} ──► message::types
param::{cc,nrpn} ──► message::{channel,types}, param::{encoding,enums}
sysex::frame ──► codec, message::types
sysex::{global,identity,poly_chain,tuning,user_module} ──► sysex::frame, message::types
sysex::program ──► codec, sysex::frame, sysex::helpers, param::enums, sysex::enums
sysex::program::file ──► device (product name), zip
sysex::transaction ──► transport, sysex::{frame,global,identity,program,tuning}
transport ──► device (port-name constants)
controller ──► transport, param::*, message::*, sysex::program (ProgramNumber)
builder ──► sysex::program, param::enums
midi_file ──► device::cc_name, param::enums, sysex::program (SynthParams)
```

Two edges point the wrong way for a strict layering: `error` imports `sysex::frame::SysexStatus`,
and `transport` imports `device` for port names. In Zig, place `SysexStatus` in the framing module
and let errors carry only the raw status byte. Keep port-name constants in a platform-neutral
module that both transport and discovery import.

## README and code drift

Claims in `README.md` that the code does not back up. Treat the code as authoritative.

| README claim | Reality |
|--------------|---------|
| `minilogue_xd::load_lib_file("factory.mnlgxdlib")` | No such function. File I/O is `sysex::program::file::read_lib_file(reader)`. |
| `PatchBuilder::build()?` returns `Result` | `build()` returns `ProgramData` directly (`crates/minilogue-xd/src/builder/patch.rs:221`). |
| `SequenceBuilder` takes `.step(0, &[60], &[100], GateTime::Percent(50))`, `.motion_slot(...)` | Actual API: `step(index, note, velocity)`, `step_poly(index, &[(note, vel)])` (`crates/minilogue-xd/src/builder/sequence.rs:93-121`). |
| `no_std` support via `std` flag | Flag is a no-op (see above). |
| `MIDI_COVERAGE.md` maps spec items to code | File does not exist in the repo. |
| `.mnlgxdpreset` and `.mnlgxdunit` support | See [file formats](../protocol/file-formats.md) and [user modules](../protocol/user-modules.md) for what is actually implemented. |
| `minilogue-xd = "0.1"` (`README.md:26`, `:33`) | The crate is `0.2.0` (`crates/minilogue-xd/Cargo.toml:3`). |
| Feature table "User modules: Query, upload, download, clear, swap" and "File formats: … `.mnlgxdpreset`, `.mnlgxdunit`" (`README.md:155-156`) | There is no upload builder, no `0x47` parser, no slot transaction methods, no `.mnlgxdpreset` (448-byte) reader and no `.mnlgxdunit` support. See [user modules](../protocol/user-modules.md#errata--gotchas). |
| `.motion_slot(..)` / `.motion_step(..)` on `SequenceBuilder` (`README.md:119-120`) | No motion API exists. See [builders](../design/builders.md#errata--gotchas). |
| `xd.play_note(60, 100)?` with plain integers (`README.md:67`) | The controller's `play_note` takes `U7` values ([realtime controller](../design/realtime-controller.md)). |
| `scale.set_note(64, CentOffset(-14.0))` (`README.md:135`) | `CentOffset` is a struct with a `semitone` field and a fraction (`sysex/tuning.rs:33`), not a float newtype. See [tuning](../protocol/tuning.md). |
| Dual licence `LICENSE-APACHE` / MIT (`README.md:280`) | The repo has a single `LICENSE` file. |
| Examples list (`README.md:238-247`) | Omits `quintal`, `read_edit_buffer` and `sysex_roundtrip_diag`. See [examples](../design/examples.md). |
| `CLAUDE.md`: "29 NRPN parameters" (`CLAUDE.md:52`) and the "planned" module list | The spec has 61 NRPN addresses; Rust has 51 variants over 62 addresses. The module list omits `device`, `connection` and `midi_file`. |

## Git history worth knowing

| Commit | Lesson | Page |
|--------|--------|------|
| `1c16566` | Five byte-fidelity bugs in program dumps (10-bit endianness, flag bits, `PREDSQ` magic, byte 148 flags, null-padded names); raw-blob preservation introduced | [byte fidelity](../design/byte-fidelity.md) |
| `685ea64`, `d465474`, `5e4b4f0` | SysEx receive hardening: complete `F0..F7` frames, fragmentation, embedded realtime bytes | [SysEx transactions](../design/sysex-transactions.md) |
| `213020c` | Typed `OutputPort`/`InputPort` enums replace port-name strings | [USB ports](../protocol/usb-ports.md) |

## Sources

- `Cargo.toml:1-6` — workspace members.
- `crates/minilogue-xd/Cargo.toml:12-27` — features and dependencies.
- `crates/minilogue-xd/src/lib.rs:8-94` — layer diagram, module list, `theory` re-export.
- `crates/minilogue/src/lib.rs:20` — umbrella re-export.
- `README.md:74-121` — drifted patch and sequencer examples (`build()?`, `load_lib_file`,
  `GateTime`, `motion_slot`).
- `README.md:269-271` — the `MIDI_COVERAGE.md` reference.
