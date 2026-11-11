---
title: Standard MIDI File (SMF Format 0) as emitted by midi_file.rs
category: protocol
sources:
  - crates/minilogue-xd/src/midi_file.rs
  - crates/minilogue-xd/examples/export_midi.rs
  - docs/design/05-active/0002-midi-file-export-implementation-plan.md
related:
  - design/midi-file-export.md
  - protocol/midi-messages.md
  - protocol/parameter-encodings.md
  - protocol/cc-map.md
  - protocol/sysex-framing.md
  - protocol/errata.md
  - zig/memory-budget.md
  - zig/test-plan.md
updated: 2026-11-11
---

# Standard MIDI File (SMF Format 0) as emitted by midi_file.rs

> The crate writes **one-track SMF Format 0** files with no external dependency. The file is
> an `MThd` chunk followed by one `MTrk` chunk. The track holds, in order: a tempo meta event,
> a fixed 4/4 time-signature meta event, the user's events sorted **stably by absolute tick**
> (equal ticks keep insertion order), and End of Track. Every event gets an explicit status
> byte (no running status). Delta times are MIDI variable-length quantities (VLQ). The writer
> has a few sharp edges, listed under Errata & gotchas: the tempo is truncated to 24 bits, VLQs
> are not limited to 4 bytes, SysEx data is written as given (F7 is not added), and nothing
> special happens when a note-off and a note-on share a tick. This page defines the bytes. The
> builder API is on [MIDI file export](../design/midi-file-export.md).

The layout follows the SMF 1.0 specification (MIDI Association). That external standard is cited
as "SMF 1.0" below. Choices this crate makes inside it are tagged **[rust]**.

## File layout

All multi-byte integers are big-endian.

| File off | Bytes | Value | Meaning |
|---:|---:|---|---|
| 0 | 4 | `4D 54 68 64` | `"MThd"` |
| 4 | 4 | `00 00 00 06` | header length = 6 |
| 8 | 2 | `00 00` | format 0 |
| 10 | 2 | `00 01` | ntrks = 1 |
| 12 | 2 | `tpq` u16 BE | division: ticks per quarter note (default 480 = `01 E0`) **[rust]** |
| 14 | 4 | `4D 54 72 6B` | `"MTrk"` |
| 18 | 4 | `len` u32 BE | number of track-data bytes that follow |
| 22 | len | track data | see next table |

Total file size = `22 + len`. The track data always starts at offset 22.

## Track data, in emission order **[rust]**

| # | Delta | Bytes | Event |
|---:|---|---|---|
| 1 | `00` | `FF 51 03 tt tt tt` | Set Tempo: µs per quarter, 24-bit BE (low 24 bits of the computed value) |
| 2 | `00` | `FF 58 04 04 02 18 08` | Time Signature 4/4, 24 MIDI clocks per click, 8 × 32nd notes per quarter (always 4/4) |
| 3…n | VLQ | event bytes | user events, sorted by tick (stable) |
| last | `00` | `FF 2F 00` | End of Track |

The time signature fields are numerator 4, denominator as a power of two (2 → 4), 24 MIDI
clocks per metronome click, and 8 thirty-second notes per MIDI quarter.

Delta for user event *k*: `tick[k] − tick[k−1]`, with `tick[−1] = 0`. The two leading meta
events sit at tick 0 and do not shift the time base. The code subtracts with saturation, but
after the sort deltas are never negative.

### User event encodings

`n` = channel 0–15. The channel is captured **when the event is added**, so `.channel()` affects
only events added after it. Data bytes are masked with `& 0x7F` where noted.

| Builder call | Event bytes | Masking / clamping **[rust]** |
|---|---|---|
| `track_name(s)` | `FF 03 VLQ(len) <UTF-8 bytes>` at tick 0 | none; length is the UTF-8 byte count |
| `note(t, k, v, d)` | `9n k v` at `t`, then `8n k 40` at `t + d` | `k`, `v` masked; note-off velocity always 64; `v = 0` written as is (a note-on with velocity 0 means note-off) |
| `cc(t, c, v)` | `Bn c v` | both masked |
| `program_change(t, p)` | `Cn p` | masked |
| `pitch_bend(t, b)` | `En lsb msb` | `b` clamped to −8192…8191, `u = b + 0x2000`, `lsb = u & 0x7F`, `msb = (u >> 7) & 0x7F` |
| `sysex(t, data)` | `F0 VLQ(len(data)) data` | none. `data` must exclude the leading F0 and should end with `F7` (see gotchas) |
| `ten_bit_cc(t, c, v)` | `Bn 3F (v & 7)` then `Bn c ((v >> 3) & 0x7F)`, same tick | `v` clamped to 1023; **`c` not masked** |
| `stepped_cc` / `on_off_cc` | `Bn c tx` / `Bn c 7F|00` | **`c` not masked** |

The 10-bit CC pair (CC63 carries the low 3 bits, then the parameter CC carries the high 7) is the
XD's high-resolution protocol. See [parameter encodings](parameter-encodings.md).

### Variable-length quantity (VLQ)

```text
vlq(value):                       # value: u64
  if value == 0: return [0x00]
  groups = []                     # 7-bit groups, least significant first
  while value > 0: groups.append(value & 0x7F); value >>= 7
  reverse(groups)                 # most significant first
  set bit 7 on every byte except the last
```

SMF 1.0 caps a VLQ at 4 bytes (max `0x0FFFFFFF`). The Rust encoder accepts any u64 and can emit
up to 10 bytes **[rust]**.

### Tempo

```text
tempo_us_per_quarter = round_half_away_from_zero(60_000_000.0 / bpm)   # f64, then `as u32`
written bytes        = (tempo >> 16) & 0xFF, (tempo >> 8) & 0xFF, tempo & 0xFF
```

| BPM | µs/quarter | Written `tt tt tt` |
|---:|---:|---|
| 120 | 500000 | `07 A1 20` |
| 60 | 1000000 | `0F 42 40` |
| 140 | 428571 | `06 8A 1B` |
| 90 | 666667 | `0A 2C 2B` |
| 3.0 | 20000000 (`0x1312D00`) | `31 2D 00` (**truncated**) |
| 0 | `inf` → saturates to `u32::MAX` | `FF FF FF` |
| NaN, negative, or > 120000000 | 0 | `00 00 00` (invalid) |

The valid range is 1…16777215 µs/quarter, which is about **3.5763 … 60,000,000 BPM**. Rust does
not check it.

### Event ordering at equal ticks

`build()` uses a **stable** sort on the absolute tick only. Events with the same tick come out
in the order they were added. There is no rule such as "note-off before note-on". Consequences:

- Building left to right (`note(0, k, v, 120)`, then `note(120, k, v, 120)`) adds the first
  note's off at tick 120 *before* the second note's on at tick 120, so the order is correct.
- Adding the later note first, or re-triggering a held pitch at the instant it ends from a
  separate pass, puts `9n k` before `8n k` at the same tick. The receiver then cuts the new note
  immediately.
- A `track_name` called after events at tick 0 lands after them. It always lands after the tempo
  and time-signature metas.
- `ten_bit_cc` pairs stay adjacent unless another event at the same tick was inserted between
  the two pushes, which the API makes impossible.

## Invariants

- `file[0..4] == "MThd"`, `u32_be(file[4..8]) == 6`, `u16_be(file[8..10]) == 0`, and
  `u16_be(file[10..12]) == 1`.
- `file[14..18] == "MTrk"` and `u32_be(file[18..22]) == file.len - 22`.
- Track data starts with `00 FF 51 03` and ends with `00 FF 2F 00`. Its bytes at indices 7–14
  are `00 FF 58 04 04 02 18 08`.
- Event ticks are non-decreasing after sorting, so every delta is ≥ 0.
- Every channel-event data byte is < 0x80. Rust guarantees this only for `note`, `cc` and
  `program_change`. Assert it for all events in a port.
- Every delta is ≤ `0x0FFFFFFF` (4-byte VLQ), and the tempo is in 1…`0xFFFFFF`. Rust asserts
  neither; a port should.
- `tpq` is in 1…`0x7FFF`. Bit 15 set would mean SMPTE division, and 0 is meaningless. Rust
  checks neither.
- `len` fits in u32. Rust casts `as u32` without checking.

## Errata & gotchas

Record these on [errata](errata.md) as [rust] issues.

1. **24-bit tempo truncation.** Below about 3.5763 BPM the µs value exceeds `0xFFFFFF` and only
   the low 24 bits are written. 3 BPM becomes 3,222,784 µs ≈ 18.6 BPM. Non-finite or huge BPM
   values give 0 or `0xFFFFFF`. Suspected bug: should be rejected.
2. **SysEx F7 not appended.** The design plan says "F0 + length + data + F7". The code writes
   `F0 VLQ(len) data`, and the only test passes data that already ends in `F7`. If the caller
   omits `F7`, SMF 1.0 reads the event as the first packet of a multi-packet SysEx that never
   finishes. A port should require `data` to end with `F7` (or append it and count it in `len`)
   and should reject a leading `F0`.
3. **No running status** **[rust]**. Every channel event carries its status byte. This is valid
   SMF and simpler to parse, but larger.
4. **Unmasked controller numbers** in `ten_bit_cc`, `stepped_cc` and `on_off_cc`. A CC number
   ≥ 128 writes a status byte where a data byte belongs, corrupting the stream, and also adds
   `CC 200  (unknown)` to the legend.
5. **VLQ over 4 bytes** for deltas ≥ 2^28 ticks (about 155 hours at 480 tpq and 120 BPM). Not
   reachable in practice, but a port should assert against it.
6. **`tick + duration` overflow** in `note()`: a u64 add that wraps (silently in release
   builds, panics in debug).
7. Track name is not first. SMF 1.0 recommends the sequence/track name at time 0, ideally first.
   Rust puts it after tempo and time signature. Common readers accept this.

## Test vectors

Exact from Rust tests:

| Vector | Expectation | Citation |
|---|---|---|
| VLQ | `0 → 00`, `127 → 7F`, `128 → 81 00`, `16383 → FF 7F`, `0x200000 → 81 80 80 00` | `midi_file.rs:932-956` |
| Tempo | 120 BPM → 500000; 60 BPM → 1000000 | `midi_file.rs:962-970` |
| Header | `MThd`, len 6, format 0, 1 track, tpq 480; tpq 960 → `03 C0` at 12..14 | `midi_file.rs:976-994` |
| Track length | `file[14..18] == "MTrk"`, `u32_be(file[18..22]) == len − 22` | `midi_file.rs:1000-1007` |
| End of track | last 3 bytes `FF 2F 00` | `midi_file.rs:1009-1014` |
| Note | `90 3C 64` and `80 3C 40` present | `midi_file.rs:1020-1030` |
| Note duration 480 | `83 60 80 3C 40` | `midi_file.rs:1032-1052` |
| Equal-tick CCs | `B0 4A 40 00 B0 47 64` (insertion order, delta 0) | `midi_file.rs:1094-1117` |
| Masking | `cc(0, 200, 200)` → `B0 48 48` | `midi_file.rs:1119-1128` |
| Pitch bend 0 | `E0 00 40` | `midi_file.rs:1134-1141` |
| Program change 42 | `C0 2A` | `midi_file.rs:1147-1153` |
| Channel | `.channel(5)` → `95`; `.channel(255)` → `9F` | `midi_file.rs:1278-1300` |

Full-file vectors derived by applying the documented algorithm. Each matches the tests' partial
assertions:

```text
Empty builder, 120 BPM (41 bytes):
4D 54 68 64 00 00 00 06 00 00 00 01 01 E0 4D 54 72 6B 00 00 00 13
00 FF 51 03 07 A1 20 00 FF 58 04 04 02 18 08 00 FF 2F 00

Module-doc quick start (72 bytes):
  new(120).track_name("My Track").note(0,60,100,240).note(480,64,90,240)
4D 54 68 64 00 00 00 06 00 00 00 01 01 E0 4D 54 72 6B 00 00 00 32
00 FF 51 03 07 A1 20 00 FF 58 04 04 02 18 08
00 FF 03 08 4D 79 20 54 72 61 63 6B
00 90 3C 64  81 70 80 3C 40  81 70 90 40 5A  81 70 80 40 40
00 FF 2F 00

Length-consistency test (84 bytes; shows stable ordering — CC added last but sorts at tick 0):
  track_name("Length Test").note(0,60,100,240).note(480,64,90,240).cc(0,74,64).pitch_bend(960,1000)
… 4D 54 72 6B 00 00 00 3E 00 FF 51 03 07 A1 20 00 FF 58 04 04 02 18 08
00 FF 03 0B 4C 65 6E 67 74 68 20 54 65 73 74
00 90 3C 64  00 B0 4A 40  81 70 80 3C 40  81 70 90 40 5A  81 70 80 40 40
81 70 E0 68 47  00 FF 2F 00

SysEx test track data (data = 42 30 00 01 51 F7):
00 FF 51 03 07 A1 20 00 FF 58 04 04 02 18 08 00 F0 06 42 30 00 01 51 F7 00 FF 2F 00

ten_bit_cc(0, 43, 1023) → 00 B0 3F 07 00 B0 2B 7F
ten_bit_cc(0, 43, 600)  → 00 B0 3F 00 00 B0 2B 4B
set_cutoff(0, 0.35)     → raw 358 → 00 B0 3F 06 00 B0 2B 2C

VLQ edges: 0x2000 → C0 00; 0x4000 → 81 80 00; 0x0FFFFFFF → FF FF FF 7F;
           0x10000000 → 81 80 80 80 00 (5 bytes: invalid SMF)
```

## Rust design

- The file is built in memory: `build()` returns `Vec<u8>`. `write_to` calls `build()` and then
  `write_all`, so nothing streams despite the wording in the doc comment.
- Events are stored as `Vec<MidiFileEvent { tick: u64, bytes: Vec<u8> }>`, one heap allocation
  per event. `slice::sort_by_key` is a stable merge sort.
- The VLQ encoder returns a fresh `Vec<u8>` per call. Tempo math uses `f64` with `round()` and a
  saturating `as u32` cast.
- Essential: the chunk layout, the meta order, the stable tick ordering, VLQ, and status bytes on
  every event. Incidental: the per-event `Vec`, `f64` tempo, and the builder's by-value chaining.

## Zig / TigerStyle notes

- Size math for a fixed buffer:
  - channel events ≤ 3 bytes plus a ≤ 4-byte VLQ = 7 bytes each;
  - header 22 bytes, metas 15, End of Track 4;
  - `smf_file_size_max = 41 + events_max * 7 + meta_pool_size`.

  With `events_max = 4096` (the bundled example uses 1167) and a 4 KiB pool for track
  name/SysEx, the file is ≤ 32,809 bytes. See [memory budget](../zig/memory-budget.md).
- Write in two passes with no allocation. The first pass computes `len` by summing
  `vlq_len(delta) + event_len`. The second writes header, `len`, then the data straight to a
  caller-supplied writer or buffer. This removes the intermediate `track_data` buffer that Rust
  uses.
- `vlq_len(value)` is `1 + (bits(value) − 1) / 7` for value > 0. Assert `value <= 0x0FFFFFFF`,
  then emit into a `[4]u8`.
- Integer tempo, no floats: take the BPM as a fixed-point integer (e.g. `tempo_milli_bpm: u32`),
  compute `tempo_us_per_quarter = (60_000_000_000 + tempo_milli_bpm / 2) / tempo_milli_bpm` in
  u64, and assert the result is in `1…0xFFFFFF`. This rounds half up, which matches Rust for
  positive inputs.
- Error set: `error{ TempoOutOfRange, DeltaTooLarge, InvalidDivision, DataByteOutOfRange,
  SysexMissingTerminator, EventCapacityExceeded, MetaPoolExhausted }`.

```zig
// sketch
fn vlq_write(out: *[4]u8, value: u32) []const u8 {
    std.debug.assert(value <= 0x0FFF_FFFF);
    var len: usize = 1;
    var rest = value >> 7;
    while (rest != 0) : (rest >>= 7) len += 1;
    std.debug.assert(len <= 4);
    var shift: u5 = @intCast((len - 1) * 7);
    for (out[0..len], 0..) |*byte, index| {
        const group: u8 = @truncate((value >> shift) & 0x7F);
        byte.* = if (index + 1 < len) group | 0x80 else group;
        if (shift >= 7) shift -= 7;
    }
    return out[0..len];
}
```

## Open questions

- Should a Zig port order a note-off before a note-on at equal tick and pitch (a deterministic
  tie-break), or keep Rust's insertion order for byte-identical output? Byte-identical output
  needs insertion order. Correct playback needs the tie-break.
- Should the time signature be configurable? XD programs store their own sequencer length and
  resolution. See [program blob](program-blob.md).

## Sources

- `crates/minilogue-xd/src/midi_file.rs:1-19` — module doc and quick start.
- `crates/minilogue-xd/src/midi_file.rs:34-58` — `encode_vlq`, `tempo_microseconds`.
- `crates/minilogue-xd/src/midi_file.rs:116-191` — `track_name`, `note`, `cc`,
  `program_change`, `pitch_bend`, `sysex`.
- `crates/minilogue-xd/src/midi_file.rs:220-271` — `ten_bit_cc`, `stepped_cc`, `on_off_cc`.
- `crates/minilogue-xd/src/midi_file.rs:857-917` — `build`, `write_to`.
- `crates/minilogue-xd/src/midi_file.rs:924-1301` — tests.
- `docs/design/05-active/0002-midi-file-export-implementation-plan.md:74-87`, `112-123` — planned
  SMF structure and tests.
- SMF 1.0 (MIDI Association, "Standard MIDI Files 1.0") — chunk, VLQ and meta-event definitions.
