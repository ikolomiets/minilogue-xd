---
title: Zig port — overview and roadmap
category: zig
sources:
  - crates/minilogue-xd/src/lib.rs
  - crates/minilogue-xd/Cargo.toml
  - crates/minilogue-xd/examples/
  - docs/wayfinder/README.md
  - https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md
related:
  - overview.md
  - design/architecture.md
  - zig/module-layout.md
  - zig/tigerstyle-mapping.md
  - zig/dependency-elimination.md
  - zig/memory-budget.md
  - zig/platform-midi.md
  - zig/test-plan.md
  - protocol/errata.md
updated: 2026-10-11
---

# Zig port — overview and roadmap

> Rebuild `minilogue-xd` in Zig as a **sans-I/O protocol core** with fixed-size buffers, comptime
> tables, and assertion-dense code, plus thin OS MIDI backends. Work bottom-up in milestones that
> follow the Rust layers. Each milestone ends with byte-exact agreement against vectors produced
> by the Rust crate, which serves as the **oracle**. The Rust crate's 973 unit tests and 2 property
> suites are the checklist. Hardware remains the final judge where Rust and spec disagree. See
> [errata](../protocol/errata.md).

## Goals and non-goals

| Goal | Measure |
|------|---------|
| Same wire behavior as the Rust crate | Byte-exact equality on every vector in [test plan](test-plan.md); differential fuzzing against a Rust oracle CLI |
| Zero third-party dependencies | Empty `.dependencies` in `build.zig.zon`; system frameworks/libs only. See [dependency elimination](dependency-elimination.md). |
| TigerStyle | Static allocation, bounded loops, ≥2 asserts/function, ≤70-line functions, explicit int sizes. See [mapping](tigerstyle-mapping.md). |
| Deterministic tests without hardware | Simulation of fragmentation, interleaved realtime bytes, timeouts, and NAKs with a seeded PRNG |
| Fix known Rust gaps | Unbounded input queue, per-message allocation, blocking receive, layering warts. See [architecture](../design/architecture.md#what-is-essential-vs-incidental). |

Non-goals for v1: a fluent builder API (use struct defaults), `music-comp-mt` theory types,
MIDI 2.0/UMP, and an async runtime.

## Milestones

| # | Deliverable | Depends on | Exit criterion |
|---|-------------|------------|----------------|
| M0 | `build.zig`, `constants.zig`, CI (`zig fmt --check`, `zig build test` on macOS/Linux/Windows) | — | Empty test suite green on 3 OSes |
| M1 | `wire/message.zig`, `wire/stream_parser.zig`, `wire/codec_7bit.zig` | M0 | [Codec](../protocol/korg-7bit-codec.md) vectors (lengths 0…1024) and [message](../protocol/midi-messages.md) round trips; PRNG property tests |
| M2 | `param/encoding.zig`, `param/stepped.zig`, `param/cc.zig`, `param/nrpn.zig` | M1 | Every CC/NRPN/stepped vector from [CC map](../protocol/cc-map.md), [NRPN map](../protocol/nrpn-map.md), [stepped params](../design/stepped-params.md) |
| M3 | `sysex/frame.zig`, `sysex/assembler.zig`, `identity`, `poly_chain`, `tuning`, `global` | M1 | Frame/status vectors; assembler simulation (fragments, realtime, garbage, overlong) |
| M4 | `sysex/program*.zig` (in-place accessors) | M2, M3 | Byte-identical blob round trips on real `.prog_bin` fixtures; see [byte fidelity](../design/byte-fidelity.md) |
| M5 | `transaction.zig` state machine | M3, M4 | Simulated request/response for every operation, including NAK `0x24…0x2F`, wrong function, and timeout; see [SysEx transactions](../design/sysex-transactions.md) |
| M6 | `io/mock.zig`, `io/coremidi.zig` (dev host), `examples/list_ports`, `read_edit_buffer` | M5 | Reads the edit buffer from real hardware on macOS |
| M7 | `io/alsa_rawmidi.zig`, `io/winmm.zig` | M6 | `list_ports` + edit buffer read on Linux and Windows; record actual port names (open question) |
| M8 | `controller.zig`, `file/zip_reader.zig`, `file/zip_writer.zig`, `file/program_file.zig` | M4 | Opens every fixture `.mnlgxdprog`/`.mnlgxdlib`; Rust reads Zig-written files; [file formats](../protocol/file-formats.md) |
| M9 | `sysex/user_module.zig` | M5 | CRC vectors; slot info/status/data round trips; see [user modules](../protocol/user-modules.md) |
| M10 | `file/smf.zig`, remaining examples | M2 | Byte-equal SMF output vs Rust for the same event list; see [MIDI file export](../design/midi-file-export.md) |

The order follows risk. M1–M5 are pure and fully testable on any machine. The OS backends (M6–M7)
are isolated and swappable. Files and user modules (M8–M9) depend on the most unverified facts.

## The Rust crate as oracle

1. Add a tiny Rust binary (outside this wiki's scope) that reads a JSON or line-oriented request
   such as `encode_7bit <hex>`, `program_dump <ch> <n> <hex blob>`, or `parse_cc_stream <hex>`,
   and prints the result bytes in hex.
2. Generate golden vectors once and commit them under `test/vectors/` in the Zig repo. Zig tests
   read them with `@embedFile`, which adds no runtime I/O and no dependency.
3. For differential fuzzing, the Zig test runner generates seeded random inputs and both
   implementations must agree. Run this outside CI, or in a nightly job, since it requires `cargo`.

Where the Rust behavior is **known wrong or doubtful**, encode the hardware truth in Zig and
mark the vector `[hw]` with a link to [errata](../protocol/errata.md). Examples include the
NRPN address offsets, the bank-select split, and the CC93/94 vs 94/95 effect switches.

## Design decisions (summary)

| Decision | Choice | Why | Page |
|----------|--------|-----|------|
| I/O model | Sans-I/O core; callers poll `drain()` and call `tick(now_ns)` | TigerStyle: don't react directly to external events; deterministic simulation | [architecture](../design/architecture.md) |
| Program storage | `[1024]u8` with offset accessors | Byte fidelity for free; no 300-line to/from functions | [byte fidelity](../design/byte-fidelity.md) |
| Value types | Native `u4/u7/u10/u14`, asserts inside, validation at the boundary | Explicit sizes, fewer wrapper types | [typed values](../design/typed-values.md) |
| Stepped params | Comptime tables (`tx`, `rx` bands, program value) | Data, not code | [stepped params](../design/stepped-params.md) |
| Errors | Per-layer error sets + `Diagnostics` out-struct | No allocation; operating vs programmer errors | [errors](../design/errors.md) |
| Input queue | SPSC byte ring, bounded, overflow counted | Bounded memory, no locks | [platform MIDI](platform-midi.md#input-handoff) |
| Zip | Own reader/writer + `std.compress.flate` | No `zip` crate | [dependency elimination](dependency-elimination.md) |
| Linux MIDI | rawmidi via ioctls, not libasound | Zero userspace deps | [platform MIDI](platform-midi.md) |

## Looking ahead: the score / DSL layer

`docs/wayfinder/` plans a control DSL and score player on top of this library. It introduces the
terms Score, Patch, Live parameter change, Score automation, and Host clock; see
[glossary](../glossary.md) and [wayfinder DSL](../sources/wayfinder-dsl.md). The Zig core should
not implement it, but must not block it:

- `controller.zig` writes messages into a caller buffer instead of owning a transport, so a
  scheduler can timestamp and order them.
- No hidden sends. Every outbound byte sequence comes from a pure function, so a score
  renderer can produce MIDI files and live output from the same code.
- The core keeps no wall-clock assumptions. The **host clock** belongs to the application and
  is passed in as `now_ns`.

## Open questions

- Should the oracle CLI live in this Rust repo, e.g. as an `examples/oracle.rs`, so vectors
  can be regenerated when Rust changes?
- What is the minimum supported Zig version? Local is 0.16.0; 0.17.0 is current upstream.
  `std.Io` changes between them, so pin one.
- Which hardware facts block release? See the open-questions sections on
  [errata](../protocol/errata.md) and [platform MIDI](platform-midi.md).

## Sources

- `crates/minilogue-xd/src/lib.rs` — public modules (scope of the port).
- `crates/minilogue-xd/Cargo.toml` — dependencies to eliminate.
- `crates/minilogue-xd/examples/` — 9 examples (`list_ports`, `read_patch`,
  `read_edit_buffer`, `sysex_roundtrip_diag`, `test_drive`, `berlin_school`, `exit`, `quintal`,
  `export_midi`) that define the user-facing workflows to replicate.
- `docs/wayfinder/README.md` — future DSL/score work.
- TigerStyle — see [schema digest](../AGENTS.md#tigerstyle-digest-constraints-for-every-zig-note).
