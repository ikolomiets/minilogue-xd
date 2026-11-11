---
title: Memory budget — every buffer, size and limit for a statically allocated port
category: zig
sources:
  - crates/minilogue-xd/src/codec.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/sysex/program/synth.rs
  - crates/minilogue-xd/src/sysex/program/sequencer.rs
  - crates/minilogue-xd/src/sysex/global.rs
  - crates/minilogue-xd/src/sysex/tuning.rs
  - crates/minilogue-xd/src/sysex/user_module.rs
  - crates/minilogue-xd/src/sysex/identity.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/src/transport.rs
  - crates/minilogue-xd/src/midi_file.rs
related:
  - zig/module-layout.md
  - zig/tigerstyle-mapping.md
  - zig/platform-midi.md
  - zig/dependency-elimination.md
  - zig/test-plan.md
  - protocol/korg-7bit-codec.md
  - protocol/sysex-framing.md
  - protocol/program-blob.md
  - protocol/global-blob.md
  - protocol/tuning.md
  - protocol/user-modules.md
  - protocol/file-formats.md
  - protocol/standard-midi-file.md
  - protocol/usb-ports.md
  - design/transport.md
  - design/sysex-transactions.md
  - design/realtime-controller.md
updated: 2026-11-11
---

# Memory budget — every buffer, size and limit for a statically allocated port

> TigerStyle requires that every buffer has a compile-time maximum and nothing is allocated after
> init. This page is the single list of those maxima for the Zig port: wire sizes, blob sizes,
> buffer capacities, file limits and timing budgets, each with its derivation and the page that
> owns the underlying fact. The key result: **without user-module slot data, the whole device
> session fits in about 23 KiB of static buffers, and the largest frame is the 1181-byte stored
> program dump.** With user-module slot data (`0x4A`), one frame can reach about 38.8 KB, so the
> port needs a second, larger build profile.

## Facts

### Sizing rule for the Korg 7-bit codec

`encoded_len(n) = n + ceil(n / 7)`, which is the same as Rust's
`(n / 7) * 8 + (if n % 7 > 0 { n % 7 + 1 } else { 0 })` (`crates/minilogue-xd/src/codec.rs:29-33`).
Every encoded size below comes from this one function. Assert each at comptime. See
[7-bit codec](../protocol/korg-7bit-codec.md).

A Korg exclusive frame is `F0 42 3g 00 01 51 fid` (7 bytes), then the body, then `F7` (1 byte).
See [SysEx framing](../protocol/sysex-framing.md).

### Comptime constants (`constants.zig`)

Names follow TigerStyle (qualifier last). The canonical fact lives on the linked page; this
table only collects the numbers.

**Messages and requests**

| Constant | Value | Derivation | Owner page |
|----------|------:|------------|------------|
| `channel_message_size_max` | 3 | Status + 2 data bytes | [MIDI messages](../protocol/midi-messages.md) |
| `sysex_header_size` | 7 | `F0 42 3g 00 01 51 fid` | [SysEx framing](../protocol/sysex-framing.md) |
| `status_frame_size` | 8 | Header 7 (fid `0x23`–`0x2F`) + `F7` | [SysEx framing](../protocol/sysex-framing.md) |
| `request_frame_size_min` | 8 | Global request `0x0E`: header + `F7` | [global blob](../protocol/global-blob.md) |
| `request_frame_size_max` | 11 | Swap user data `0x1E`: header + 3 + `F7` | [SysEx framing](../protocol/sysex-framing.md) |
| `identity_request_size` | 6 | `F0 7E nn 06 01 F7` | [device identity](../protocol/device-identity.md) |
| `identity_reply_size` | 15 | `identity.rs:43` | [device identity](../protocol/device-identity.md) |
| `poly_chain_note_on_size` | 14 | | [poly chain](../protocol/poly-chain.md) |
| `poly_chain_note_off_size` | 10 | | [poly chain](../protocol/poly-chain.md) |
| `controller_group_size_max` | 12 | NRPN address + 10-bit value with CC63: 4 CCs × 3 | [parameter encodings](../protocol/parameter-encodings.md), [realtime controller](../design/realtime-controller.md) |

**Program data**

| Constant | Value | Derivation | Owner page |
|----------|------:|------------|------------|
| `program_size` | 1024 | `program/mod.rs:107` | [program blob](../protocol/program-blob.md) |
| `program_synth_size` | 156 | `synth.rs:361` | [program blob](../protocol/program-blob.md) |
| `program_sequencer_size` | 868 | `sequencer.rs:150-153`; offset 156 | [sequencer blob](../protocol/sequencer-blob.md) |
| `sequencer_step_size` | 52 | `sequencer.rs:45` | [sequencer blob](../protocol/sequencer-blob.md) |
| `program_encoded_size` | 1171 | `encoded_len(1024)` = 1024 + 147 | [7-bit codec](../protocol/korg-7bit-codec.md) |
| `program_current_frame_size` | 1179 | `0x40`: 7 + 1171 + 1 | [SysEx framing](../protocol/sysex-framing.md) |
| `program_stored_frame_size` | 1181 | `0x4C`: 7 + 2 (program number) + 1171 + 1 | [SysEx framing](../protocol/sysex-framing.md) |
| `program_number_max` | 499 | `program/mod.rs:35` | [MIDI messages](../protocol/midi-messages.md#bank-select-and-program-numbers) |
| `preset_program_size` | 448 | Firmware 2.10+ `.mnlgxdpreset` entries; `encoded_len(448)` = 512 | [file formats](../protocol/file-formats.md) |

**Global data and tuning**

| Constant | Value | Derivation | Owner page |
|----------|------:|------------|------------|
| `global_blob_size` | 63 | `global.rs:138` | [global blob](../protocol/global-blob.md) |
| `global_dump_encoded_size` | 72 | `encoded_len(63)` | [global blob](../protocol/global-blob.md) |
| `global_dump_frame_size` | 80 | 7 + 72 + 1 | [global blob](../protocol/global-blob.md) |
| `user_scale_size` | 384 | 128 notes × 3 bytes (`tuning.rs:90`) | [tuning](../protocol/tuning.md) |
| `user_octave_size` | 36 | 12 notes × 3 bytes (`tuning.rs:155`) | [tuning](../protocol/tuning.md) |
| `user_scale_frame_size_max` | 447 | Rust form: 7 + `encoded_len(384)` = 439 + 1. The spec form (raw, with `ss`) is 393 | [tuning](../protocol/tuning.md) |
| `user_octave_frame_size_max` | 50 | Rust form: 7 + 42 + 1. The spec form is 45 | [tuning](../protocol/tuning.md) |
| `mts_bulk_frame_size` | 408 | Fixed [spec] | [tuning](../protocol/tuning.md) |
| `mts_note_change_frame_size_max` | 516 | `8 + 4·n` with `n ≤ 127` | [tuning](../protocol/tuning.md) |

**User modules**

| Constant | Value | Derivation | Owner page |
|----------|------:|------------|------------|
| `user_module_info_size` | 9 | `user_module.rs:112` | [user modules](../protocol/user-modules.md) |
| `user_slot_status_size` | 32 | `user_module.rs:175` | [user modules](../protocol/user-modules.md) |
| `user_slot_data_header_size` | 8 | `user_module.rs:255` | [user modules](../protocol/user-modules.md) |
| `user_slot_payload_size_max` | 33,932 | 8 + 1024 + 32,768 + 132 (assumed largest xd `load_size`) | [user modules](../protocol/user-modules.md) |
| `user_slot_encoded_size_max` | 38,780 | `encoded_len(33,932)` | [user modules](../protocol/user-modules.md) |
| `user_slot_frame_size_max` | 38,790 | 7 + 2 + 38,780 + 1 | [user modules](../protocol/user-modules.md) |

**Host-side limits (Zig decisions, not device facts)**

| Constant | Value | Why | Owner page |
|----------|------:|-----|------------|
| `port_count_max` | 64 | Enumerate into a fixed array; truncation is counted | [USB ports](../protocol/usb-ports.md) |
| `port_name_bytes_max` | 128 | Longer names are truncated and counted | [USB ports](../protocol/usb-ports.md) |
| `messages_max` / `message_bytes_max` | 64 / 1181 | `MockOutput` capacity: one stored program dump per slot | [transport](../design/transport.md) |
| `program_count_max` | 500 | Programs per `.mnlgxdlib`; equals `program_number_max + 1` | [file formats](../protocol/file-formats.md) |
| `events_max` | 4096 | SMF events; the bundled example needs 1167 | [MIDI file export](../design/midi-file-export.md) |
| `meta_pool_size` | 4096 | SMF track-name and SysEx bodies | [standard MIDI file](../protocol/standard-midi-file.md) |
| `timeout_ms` (default) | 5000 | Rust `DEFAULT_TIMEOUT` (`transaction.rs:48`) | [SysEx transactions](../design/sysex-transactions.md) |
| `poll_interval_ms` | 1 | `poll_count_max = timeout_ms / poll_interval_ms + 1` = 5001 | [SysEx transactions](../design/sysex-transactions.md) |
| `frames_unrelated_max` | 16 | Unrelated frames tolerated per transaction | [SysEx transactions](../design/sysex-transactions.md) |

### Build profiles

Slot-data transfer (`0x4A`) is the only feature that needs frames larger than about 1.2 KB. To
keep it from inflating every buffer, select one of two profiles with a `build.zig` option. That
uses only `std.Build` options, so there is still no third-party dependency.

| Constant | Profile **core** (default) | Profile **user_modules** | Rule |
|----------|---------------------------:|-------------------------:|------|
| `frame_size_max` | 2048 | 49,152 (48 KiB) | ≥ largest accepted frame: 1181, or 38,790 |
| `blob_size_max` (decode target) | 1024 | 40,960 (40 KiB) | ≥ largest decoded body: 1024, or 33,932 |
| `ring_capacity` | 16,384 | 262,144 (256 KiB) | Power of two, ≥ 4 × the largest frame (4,724, or 155,160) |
| `read_scratch_size` | 256 | 256 | One `drain()` chunk. Bounded by the ring, so the size only sets the loop count |

`ring_capacity` uses the 4× headroom rule from [transport](../design/transport.md) and
[platform MIDI](platform-midi.md). A 4,096-byte ring (3 × 1181) is the bare minimum for one
program dump plus noise; the 4× rule adds room for a stalled application loop.

### Static allocation totals

**Device session, profile core.** These are long-lived buffers owned by the session struct.

| Buffer | Bytes | Owner |
|--------|------:|-------|
| Input ring (SPSC) | 16,384 | `io/` ([platform MIDI](platform-midi.md)) |
| Read scratch | 256 | Application loop |
| Stream assembler (`frame_size_max`) | 2048 | [SysEx transactions](../design/sysex-transactions.md) |
| Transaction reply copy (`out: *[frame_size_max]u8`) | 2048 | [SysEx transactions](../design/sysex-transactions.md) |
| Transmit frame (largest write: `0x4C`) | 1181 | Request builder |
| Decoded blob (`Program = [1024]u8`) | 1024 | Caller |
| **Total** | **22,941 (≈ 22.4 KiB)** | |

Optional: port enumeration needs 64 × 128 = 8,192 bytes during `list_ports` only (stack or a
temporary). The [realtime controller](../design/realtime-controller.md) needs only a 12-byte
group on the stack per call.

**Device session, profile user_modules.**

| Buffer | Bytes |
|--------|------:|
| Input ring | 262,144 |
| Read scratch | 256 |
| Stream assembler | 49,152 |
| Transaction reply copy | 49,152 |
| Transmit frame (upload `0x4A`) | 49,152 |
| Decoded blob | 40,960 |
| **Total** | **450,816 (≈ 440 KiB)** |

To save memory, a slot upload can reuse the reply-copy buffer as its transmit frame, because the
reply to an upload is an 8-byte status frame. The cost is aliasing that has to be asserted. Do
this only if 48 KiB matters.

**File I/O (allocated by the caller, used only during the call).**

| Operation | Buffer | Bytes | Why |
|-----------|--------|------:|-----|
| Read `.mnlgxdlib` | `[]Program` output | ≤ 512,000 | 500 × 1024 |
| | Entry buffer | 1024 | One `prog_bin` at a time |
| | Central-directory and tail scratch | 131,072 | ≤ 1002 entries × (46 + `zip_entry_name_bytes_max` = 64) = 110,220; the end-record search needs 22 + 65,535 = 65,557 and reuses this buffer |
| | Inflate window | 65,536 | `std.compress.flate.max_window_len` (Zig 0.16 std, `flate.zig:5-7`) |
| Write `.mnlgxdlib` (Stored, streaming) | Per-entry record | 500 × 12 | Offset `u32`, CRC `u32`, index `u16`, padding. Lets the central directory be written after the data |
| Write `.mnlgxdlib` (into memory) | Output | ≈ 570 KB stored, `prog_bin` only; budget 800 KB with `prog_info` and the full manifest | Formula on [file formats](../protocol/file-formats.md) |
| Write SMF | `events: [4096]Event` | 4096 × `@sizeOf(Event)` (≤ 12 → 49,152) | [MIDI file export](../design/midi-file-export.md) |
| | Meta pool | 4096 | |
| | Output | ≤ 32,809 | `41 + 4096 × 7 + 4096`; header chunk 22, empty file 41 |

`zip_entry_name_bytes_max = 64` is a Zig decision. Korg names are 17–19 bytes
(`Prog_NNN.prog_bin`, `FileInformation.xml`); 64 leaves room for the nested folders the lenient
reader accepts. Reject longer names with `error.ZipInvalid`.

### Timing budget

| Item | Value | Derivation |
|------|------:|------------|
| DIN MIDI byte rate | 3125 B/s | 31,250 baud ÷ 10 bits per byte |
| Global dump `0x51` over DIN | ≈ 26 ms | 80 B |
| Program dump `0x4C` over DIN | ≈ 378 ms | 1181 B |
| User slot dump `0x4A` (largest) over DIN | ≈ 12.4 s | 38,790 B. **Exceeds the 5 s default timeout** |
| Controller group over DIN | ≈ 3.8 ms | 12 B |
| 16 KiB ring at DIN rate | ≈ 5.2 s of input | How long the loop may stall before data is dropped |

USB-MIDI is much faster than DIN, but the XD's USB pacing is unmeasured. See the
[transport](../design/transport.md) open questions.

## Invariants

Each bullet is meant to be a `comptime assert` in `constants.zig`, or a runtime `assert` where
noted.

- `encoded_len(program_size) == program_encoded_size` (1171). The same holds for `global_blob_size`
  (72), 448 (512), `user_scale_size` (439), `user_octave_size` (42) and
  `user_slot_payload_size_max` (38,780).
- `program_synth_size + program_sequencer_size == program_size`.
- `sysex_header_size + 2 + program_encoded_size + 1 == program_stored_frame_size`.
- `frame_size_max >=` every fixed-size frame: 1181, 1179, 516, 447, 408, 80, 50, 15, 14, 11, 10, 8.
  In profile user_modules, also `>= user_slot_frame_size_max`.
- `blob_size_max >= program_size`. In profile user_modules, also `>= user_slot_payload_size_max`.
- `std.math.isPowerOfTwo(ring_capacity)` and `ring_capacity >= 4 * frame_size_max_fixed`. In
  profile user_modules, also `ring_capacity >= 4 * user_slot_frame_size_max`.
- `frame_size_max <= std.math.maxInt(u16)` in both profiles. Frame lengths are `u16`, as on
  [SysEx framing](../protocol/sysex-framing.md).
- `program_count_max == program_number_max + 1`.
- `41 + events_max * 7 + meta_pool_size <= smf_file_size_max`, and `@sizeOf(Event) <= 12`.
- File offsets are `u32`, so assert every archive budget is `< 1 << 32`.
- Runtime: before a slot upload or download, assert `payload_size <= blob_size_max`, and reject
  (error, not assert) `payload_size > max_program_size` reported by the device. See
  [user modules](../protocol/user-modules.md).

## Errata & gotchas

- **Rust has no bounds anywhere.** It uses `Vec<u8>` for every frame, `Vec<Vec<u8>>` for
  `MockOutput` (`transport.rs:49-50`), and an unbounded event list for SMF. It also accepts frames
  longer than expected (identity, poly chain, MTS bulk). Every limit on this page is a new Zig
  decision, except the device-defined sizes.
- **Tuning dump size depends on who is right.** Rust 7-bit-encodes `0x44`/`0x45` and omits `ss`
  (447 / 50 bytes). The spec sends raw bytes with `ss` (393 / 45). Budget the larger, and accept
  both on receive until [hw] settles it. See [tuning](../protocol/tuning.md) and
  [errata](../protocol/errata.md).
- **The 5 s default timeout cannot cover a large slot dump over DIN** (≈ 12.4 s). Scale the
  deadline with frame size, for example
  `timeout_ms = timeout_base_ms + frame_size * 2 * 1000 / din_bytes_per_second`, or require USB for
  slot transfers.
- **Ring sizing is easy to under-state.** "Fits one program dump" (4096) is not the same as
  "survives a stalled loop" (4×). This page is canonical: 16 KiB (core) and 256 KiB
  (user_modules).
- **The largest user-module size is an assumption.** It is `load_size ≤ 32768` for an xd
  oscillator. The NTS-1 reports 36,848 as its maximum slot size. If the XD reports more, raise
  `user_slot_payload_size_max` and re-derive the whole profile.

## Rust design

Rust sizes buffers at run time: `encode_7bit` returns a fresh `Vec`, `decode_7bit` accepts any
length, and transactions collect frames into a growing `Vec`. That is convenient, but it makes
worst-case memory unknowable. The Zig port turns each `Vec` into a `[N]u8` declared from
`constants.zig`, or into a caller-provided slice whose length is asserted against the same
constants. See [dependency elimination](dependency-elimination.md) and
[TigerStyle mapping](tigerstyle-mapping.md).

## Zig / TigerStyle notes

```zig
// sketch — constants.zig
const std = @import("std");
const assert = std.debug.assert;
const build_options = @import("build_options");

pub fn encoded_len(decoded_len: u32) u32 {
    return decoded_len + (decoded_len + 6) / 7;
}

pub const program_size = 1024;
pub const program_encoded_size = encoded_len(program_size);
pub const program_stored_frame_size = 7 + 2 + program_encoded_size + 1;
pub const frame_size_max_fixed = program_stored_frame_size;

pub const frame_size_max: u32 = if (build_options.user_modules) 48 * 1024 else 2048;
pub const blob_size_max: u32 = if (build_options.user_modules) 40 * 1024 else program_size;
pub const ring_capacity: u32 = if (build_options.user_modules) 256 * 1024 else 16 * 1024;

comptime {
    assert(program_encoded_size == 1171);
    assert(program_stored_frame_size == 1181);
    assert(frame_size_max >= frame_size_max_fixed);
    assert(frame_size_max <= std.math.maxInt(u16));
    assert(std.math.isPowerOfTwo(ring_capacity));
    assert(ring_capacity >= 4 * frame_size_max_fixed);
}
```

- Keep one `constants.zig` ([module layout](module-layout.md)). Every buffer type is spelled
  `[constants.x]u8`. A grep for a literal `1024` or `1181` outside `constants.zig` is a lint
  failure.
- Put session buffers in one struct, initialized once (`var session: Session = .{}` at file
  scope or in `main`). Nothing is allocated after init, and no allocator is threaded through the
  core.
- Tests use the same constants. The simulation tests in the [test plan](test-plan.md) also fill
  every buffer to exactly its maximum and to maximum + 1. The second case must fail with an
  error, not corrupt memory.

## Open questions

- What `max_program_size` does the XD report in TABLE 5 for each module type? That decides the
  user_modules profile ([user modules](../protocol/user-modules.md)).
- How fast does the XD push SysEx on its USB IN endpoint? That decides whether 16 KiB of ring is
  ever too little ([transport](../design/transport.md)).
- Does the XD send a slot dump as one SysEx frame, or as several? This page assumes one frame.
- Are real `.mnlgxdlib` files from Korg's librarian ever above 500 entries, or have nested
  folders or archive comments? The zip limits above assume not.

## Sources

- `crates/minilogue-xd/src/codec.rs:29-33` — encoded length formula.
- `crates/minilogue-xd/src/sysex/program/mod.rs:35`, `:107` — program number max and blob size.
- `crates/minilogue-xd/src/sysex/program/synth.rs:361`, `sequencer.rs:45`, `:150-153` — section sizes.
- `crates/minilogue-xd/src/sysex/global.rs:138` — global blob size.
- `crates/minilogue-xd/src/sysex/tuning.rs:90`, `:155` — scale and octave blob sizes.
- `crates/minilogue-xd/src/sysex/user_module.rs:112`, `:175`, `:255` — user-module blob sizes.
- `crates/minilogue-xd/src/sysex/identity.rs:43` — identity reply length.
- `crates/minilogue-xd/src/sysex/transaction.rs:48` — default timeout.
- `crates/minilogue-xd/src/transport.rs:49-50` — unbounded `MockOutput`.
- `crates/minilogue-xd/src/midi_file.rs:895-903` — SMF header and track chunk.
- Zig 0.16 std `lib/std/compress/flate.zig:5-7` — `history_len = 32768`, `max_window_len = 65536`.
