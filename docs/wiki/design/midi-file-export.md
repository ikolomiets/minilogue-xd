---
title: MidiFileBuilder — SMF export design
category: design
sources:
  - crates/minilogue-xd/src/midi_file.rs
  - crates/minilogue-xd/src/device.rs
  - crates/minilogue-xd/src/sysex/program/synth.rs
  - crates/minilogue-xd/examples/export_midi.rs
  - crates/minilogue-xd/examples/berlin_school-legend.txt
  - crates/minilogue-xd/examples/quintal.rs
  - crates/minilogue-xd/Cargo.toml
  - crates/minilogue-xd/src/lib.rs
  - docs/design/05-active/0002-midi-file-export-implementation-plan.md
related:
  - protocol/standard-midi-file.md
  - protocol/cc-map.md
  - protocol/parameter-encodings.md
  - protocol/program-blob.md
  - protocol/errata.md
  - design/stepped-params.md
  - design/realtime-controller.md
  - design/typed-values.md
  - design/examples.md
  - zig/memory-budget.md
  - zig/tigerstyle-mapping.md
  - zig/test-plan.md
updated: 2026-10-11
---

# MidiFileBuilder — SMF export design

> `MidiFileBuilder` turns a performance into an SMF Format 0 byte vector, with no hardware and no
> dependencies. It collects `(absolute tick, raw event bytes)` pairs through a by-value fluent
> API, sorts them stably by tick, and serializes them as described on
> [Standard MIDI File](../protocol/standard-midi-file.md). On top of the generic MIDI calls it
> adds XD helpers: 10-bit CC pairs (CC63 then the parameter CC), stepped-enum CCs using TX
> values, on/off CCs, six named setters, and `patch_ccs`, which turns a `SynthParams` program
> into **78 CC events over 50 distinct CCs** at one tick. It also writes a plain-text **CC
> legend** naming every CC used. The plan (design doc 0002) was implemented with a few
> divergences, listed below.

## API

All builder methods take `self` and return `Self`. `legend` borrows; `build` and `write_to`
consume.

| Method | Effect | Notes **[rust]** |
|---|---|---|
| `new(tempo_bpm: f64)` | tpq 480, channel 0, no events | tempo stored as f64, converted in `build` |
| `channel(ch: u8)` | sets channel for **subsequent** events | clamped `min(15)` |
| `ticks_per_quarter(tpq: u16)` | sets SMF division | not validated (0 or ≥ 0x8000 accepted) |
| `track_name(name)` | `FF 03` meta at tick 0 | UTF-8 bytes |
| `note(tick, note, vel, dur)` | note-on at `tick`, note-off (vel 64) at `tick + dur` | masks note/vel |
| `cc(tick, controller, value)` | `Bn c v` | masks both; records `c & 0x7F` in the legend set |
| `program_change(tick, program)` | `Cn p` | masks |
| `pitch_bend(tick, value: i16)` | `En lsb msb` | clamps −8192…8191 |
| `sysex(tick, data)` | `F0 VLQ(len) data` | caller supplies trailing `F7` |
| `ten_bit_cc(tick, cc, value: u16)` | `Bn 3F lo3` + `Bn cc hi7` | clamps 0…1023; `cc` unmasked |
| `ten_bit_cc_f32(tick, cc, value: f32)` | `round(clamp(v, 0, 1) × 1023)` then `ten_bit_cc` | NaN → 0 |
| `stepped_cc(tick, cc, value: impl SteppedParam)` | `Bn cc value.to_tx_value()` | see [stepped params](stepped-params.md) |
| `on_off_cc(tick, cc, on)` | `Bn cc 7F` / `Bn cc 00` | |
| `set_cutoff` / `set_resonance` / `set_lfo_rate` / `set_lfo_int` / `set_delay_dry_wet` / `set_reverb_dry_wet` | `ten_bit_cc_f32` on CC 43 / 44 / 24 / 26 / 107 / 110 | value 0.0–1.0 |
| `patch_ccs(tick, &SynthParams)` | 78-event program snapshot (table below) | |
| `legend() -> String` | CC legend text | |
| `build() -> Vec<u8>` | complete SMF | sorts, then serializes |
| `write_to(&mut impl Write)` | `build()` then `write_all` | not streaming |

## State and algorithm

```text
state:
  ticks_per_quarter: u16 = 480
  tempo_bpm: f64
  channel: u8 = 0                    # applied at event-creation time
  events: list of (tick: u64, bytes) # append-only, unsorted
  used_ccs: sorted set of u8         # for legend()

build():
  stable_sort(events, key = tick)    # equal ticks keep insertion order
  track = [00 FF 51 03 tempo24] ++ [00 FF 58 04 04 02 18 08]
  prev = 0
  for e in events: track ++= vlq(e.tick - prev) ++ e.bytes; prev = e.tick
  track ++= [00 FF 2F 00]
  return "MThd" 00000006 0000 0001 tpq_be16 ++ "MTrk" len_be32(track) ++ track
```

The tempo conversion is `round(60_000_000.0 / bpm) as u32`, written as its low 24 bits
big-endian. The range and truncation rules are on
[Standard MIDI File](../protocol/standard-midi-file.md#tempo).

Each event's bytes are fully formed when it is added. `build()` does no channel or status
logic. It only sorts, adds delta times, and frames. This keeps ordering deterministic: output
depends only on the call sequence.

## patch_ccs — program to CC snapshot

All events share one `tick`, so the stable sort keeps them in this order. 10-bit values are
**not clamped** in `patch_ccs`: its local `emit_10bit` sends `v & 7` and `(v >> 3) & 0x7F`, so
bits above 10 are lost. `ten_bit_cc` clamps. Stepped values use `to_tx_value()`; see
[stepped params](stepped-params.md) and [CC map](../protocol/cc-map.md).

**1. Stepped and switch CCs (17 events)**

| # | CC | Source field | Encoding |
|---:|---:|---|---|
| 1 | 50 | `vco1_wave` | TX value |
| 2 | 48 | `vco1_octave` | TX value |
| 3 | 51 | `vco2_wave` | TX value |
| 4 | 49 | `vco2_octave` | TX value |
| 5 | 80 | `sync` | 127 / 0 |
| 6 | 81 | `ring` | 127 / 0 |
| 7 | 53 | `multi_type` | TX value |
| 8 | 23 | `eg_target` | TX value |
| 9 | 57 | `lfo_wave` | TX value |
| 10 | 58 | `lfo_mode` | TX value |
| 11 | 56 | `lfo_target` | TX value |
| 12 | 84 | `cutoff_drive` | TX value |
| 13 | 83 | `cutoff_keytrack` | TX value |
| 14 | 88 | `mod_fx_type_enum()` | TX value. If `mod_fx_type` is not 1–5, sends **0** |
| 15 | 96 | `mod_fx_chorus` / `_ensemble` / `_phaser` / `_flanger` / `_user`, chosen by raw `mod_fx_type` 1–5; else 0 | **raw program value** `& 0x7F` (suspect, see errata) |
| 16 | 89 | `delay_sub_type` | TX value |
| 17 | 90 | `reverb_sub_type` | TX value |

**2. 10-bit pairs (29 pairs = 58 events, each `CC63 lo3` then `CC hi7`)**

| CC | Source field | CC | Source field |
|---:|---|---:|---|
| 34 | `vco1_pitch` | 18 | `amp_eg_sustain` |
| 36 | `vco1_shape` | 19 | `amp_eg_release` |
| 39 | `vco1_level` | 20 | `eg_attack` |
| 35 | `vco2_pitch` | 21 | `eg_decay` |
| 37 | `vco2_shape` | 22 | `eg_int` |
| 40 | `vco2_level` | 24 | `lfo_rate` |
| 41 | `cross_mod_depth` | 26 | `lfo_int` |
| 54 | `shape_noise` / `shape_vpm` / `shape_user` by `multi_type` | 28 | `mod_fx_time` |
| 104 | `shift_shape_noise` / `_vpm` / `_user` by `multi_type` | 29 | `mod_fx_depth` |
| 33 | `multi_level` | 105 | `delay_time` |
| 43 | `cutoff` | 106 | `delay_depth` |
| 44 | `resonance` | 107 | `delay_dry_wet` |
| 16 | `amp_eg_attack` | 108 | `reverb_time` |
| 17 | `amp_eg_decay` | 109 | `reverb_depth` |
| | | 110 | `reverb_dry_wet` |

The emission order is the left column top to bottom, then the right column top to bottom.

**3. Effect switches (3 events)**: CC92 `mod_fx_on`, CC93 `delay_on`, CC94 `reverb_on`, each
127/0. CC93/94 match the TXT spec. The E8 printed chart lists 94/95; see
[CC map](../protocol/cc-map.md) and [errata](../protocol/errata.md).

Totals: 17 + 58 + 3 = **78 events**. Distinct CCs: 17 + 29 + 3 + CC63 = **50**.

**Not emitted**, though `cc_name` knows them and `SynthParams` has fields for some:
- CC5 portamento, CC27/59 voice-mode depth, CC103 multi sub-type select (`select_noise`,
  `select_vpm`, `select_user`), CC64 damper.
- All NRPN parameters (voice mode type, etc.).

So a `patch_ccs` snapshot does not fully reproduce a program. Use a SysEx program dump for
that ([program blob](../protocol/program-blob.md)).

## CC legend

```text
line 1: "Minilogue XD MIDI CC Legend"
line 2: "===========================" (27 '=')
then, for each cc in used_ccs ascending:
        format("CC {:>3}  {}", cc, cc_name(cc) or "(unknown)")   # right-aligned width 3, two spaces
joined with "\n"; no trailing newline
```

`used_ccs` gets an entry from `cc`, `ten_bit_cc` (both 63 and the CC), `stepped_cc`,
`on_off_cc` and `patch_ccs`. It gets nothing from notes, pitch bend, program change or SysEx.
The names come from `device::cc_name` (64 known CCs), the same table the
[CC map](../protocol/cc-map.md) uses. `examples/berlin_school-legend.txt` is the legend for one
`patch_ccs` plus `set_cutoff` calls. It has 52 lines: 2 header lines and 50 CC lines from
`CC  16  Amp EG Attack` to
`CC 110  Reverb Dry/Wet`, with `CC  63  Data Entry LSB (10-bit low 3 bits)` among them. The
file is 1076 bytes with no final newline.

## Example: export_midi

`examples/export_midi.rs` (no `required-features`; the `midi_file` module is not gated):

1. Sets 120 BPM, tpq 480, a 16th-note step of 120 ticks, and a track name.
2. Calls `patch_ccs(0, …)` with saw VCOs, cutoff 358, resonance 460, an amp EG, and tape delay
   and hall reverb switched on.
3. Plays 10 passes, 32 repetitions in total. Each repetition does one `set_cutoff` (2 events)
   and 16 notes (32 events), with gates shorter than the step so the same pitch never overlaps.

That is 1 + 78 + 32 × 34 = **1167 events**. The example writes the legend first, because
`build()` consumes the builder, then `berlin_school.mid`. It takes note numbers from the
`music-comp-mt` theory crate, a convenience that a Zig port replaces with integer constants. See
[examples](examples.md).

`examples/quintal.rs` also uses the builder. It adds a note when it ends, which is safe for
re-triggers. Notes still held at the end come from iterating a `HashMap`, so their order at
equal ticks is nondeterministic between runs.

## Plan (design doc 0002) vs implementation

| Item | Plan | Implemented | Verdict |
|---|---|---|---|
| Struct | `ticks_per_quarter`, `tempo_bpm`, `channel`, `events` | same + `used_ccs: BTreeSet<u8>` | extended (legend, commit 417056f) |
| Core API (12 methods) | `new` … `write_to` | all present, same signatures | ✔ |
| XD helpers | not planned | `ten_bit_cc(_f32)`, `stepped_cc`, `on_off_cc`, 6 `set_*`, `legend` | added (d77798b, 417056f) |
| Internal helpers | `write_header`, `write_track`, `tempo_to_microseconds → [u8; 3]` | only `encode_vlq` and `tempo_microseconds → u32`; header/track inline in `build` | diverged; the `[u8; 3]` design would have avoided silent truncation |
| Meta order | name, tempo, time sig | tempo, time sig, then name as a tick-0 event | diverged |
| patch_ccs size | "~45 CCs" | 50 distinct CCs, 78 events; also sync/ring and CC96 | diverged (larger) |
| On/off set | mod FX, delay, reverb | same + sync (80), ring (81) | extended |
| SysEx test | "F0 + length + data + F7" | `F0 VLQ(len) data`; caller provides F7 | diverged |
| Tests | ~25 | 28 | ✔ |
| Example features | no `required-features` | none (`Cargo.toml:49-50`) | ✔ |
| Dependencies | none | none for SMF (`std::io::Write` only) | ✔ |

History: ec1222b (SMF export), d77798b (public CC helpers + full performance), 417056f (legend),
cdc0a73 (`cc_name` moved to `device.rs`), 9ea5ecf (example uses theory note names). The design
doc is still under `05-active/`, although the work is done.

## Invariants

- Each event's byte list is non-empty and its first byte is a status byte (`0x80`–`0xFF`). All
  later channel-event bytes are < 0x80. Rust guarantees the second part only for the masked
  methods.
- `used_ccs ⊆ 0..=127`. Rust breaks this when an unmasked `cc_number ≥ 128` is passed.
- After `build`'s sort, `events[i].tick <= events[i+1].tick`, and equal ticks keep insertion
  index order (stability).
- `patch_ccs` appends exactly 78 events, and afterwards `used_ccs` has ≥ 50 entries.
- The `ten_bit_cc` pair: first event `[Bn, 0x3F, v & 7]`, second `[Bn, cc, (v >> 3) & 0x7F]`,
  same tick, adjacent.
- The legend has `2 + |used_ccs|` lines.

## Errata & gotchas

1. **CC96 sends raw sub-type indices.** The spec gives CC96 TX values: chorus, phaser and
   flanger use `0,16,32,48,64,80,96,127`; ensemble uses `0,64,127` **[spec]** (Korg MIDI Impl
   rev 1.01 §1-1 *2-18…*2-21). On receive, the bands are 16 wide (chorus, phaser, flanger), 43
   wide (ensemble, `0~42`, `43~85`, `86~127`) and 8 wide (user, 16 bands) **[spec]** (§2-1 *6-18).
   `patch_ccs` sends the raw program index (0–7, or 0–15 for user), so the device decodes
   every non-user sub-type as the first one (STEREO). User indices 8–15 decode as the second.
   Suspected bug **[rust]**. Convert through the sub-type's TX table, as the other stepped CCs
   do.
2. **CC88 fallback.** `mod_fx_type_enum()` fails on 0 and on raw values > 5. `patch_ccs` then
   sends CC88 = 0, which the device decodes as CHORUS (`0~25`, §2-1 *6-12) **[spec]**. It does not
   leave out the CC.
3. **`emit_10bit` does not clamp.** A `u16` field > 1023 wraps its high bits. `ten_bit_cc` clamps
   to 1023. Use the clamp in both places.
4. **Unmasked CC numbers** in `ten_bit_cc`, `stepped_cc` and `on_off_cc`. See
   [Standard MIDI File gotchas](../protocol/standard-midi-file.md#errata--gotchas).
5. **`channel()` is positional.** Events added before the call keep the old channel. This is
   intentional, but it surprises users of a builder.
6. **`write_to` is not streaming.** It allocates the full file first.
7. **Same-tick note-on/note-off**: insertion order decides. See
   [Standard MIDI File](../protocol/standard-midi-file.md#event-ordering-at-equal-ticks).

## Test vectors

- Default `SynthParams`: the snapshot contains `B0 32 …` (CC50), `B0 2B …` (CC43) and
  `B0 5C …` (CC92) — `midi_file.rs:1176-1194`.
- `VcoWave::Sqr` on CC50, `VcoOctave::Sixteen` on CC48, `B0 5D 7F` (delay on) and the
  `DelaySubType::Tape` TX value on CC89 — `midi_file.rs:1195-1225`.
- `write_to` output equals `build` output — `midi_file.rs:1261-1273`.
- Exact helper bytes (derived): `ten_bit_cc(0, 43, 1023)` → `00 B0 3F 07 00 B0 2B 7F`;
  `set_cutoff(0, 0.35)` → `00 B0 3F 06 00 B0 2B 2C`. Full-file vectors are on
  [Standard MIDI File](../protocol/standard-midi-file.md#test-vectors).
- Golden legend: `examples/berlin_school-legend.txt` (50 CC lines, byte-exact, no trailing
  newline).

## Rust design

- The builder is a by-value fluent struct. Events are `Vec<MidiFileEvent>`, each owning a
  `Vec<u8>`. `BTreeSet<u8>` keeps the legend sorted. `sort_by_key` is stable. The output is
  `Vec<u8>`, and `std::io::Write` is used only by `write_to`.
- `patch_ccs` defines four nested helper `fn`s (`emit_10bit`, `emit_stepped`, `emit_bool`,
  `emit_cc`) so the `&mut self.events` and `&mut self.used_ccs` borrows stay split. That is a
  Rust borrow-checker idiom, not part of the design.
- It depends on `SteppedParam::to_tx_value` (CC TX bands), `SynthParams` (program model),
  `MultiType` and `device::cc_name`.
- Essential: absolute-tick storage plus a stable sort; the event list in `patch_ccs` and its
  order; the 10-bit pair order (LSB first); the legend format. Incidental: f64 tempo, per-event
  heap `Vec`, `String` legend, by-value chaining.

## Zig / TigerStyle notes

- **Fixed capacity.** `events: [events_max]Event` with `events_max = 4096` (the example needs
  1167; a 16-step × 8-note × 4-motion sequence export is far smaller). `Event` is
  `{ tick: u32, sequence: u16, len: u8, bytes: [3]u8 }` for channel events. Track name and SysEx
  bodies go in a separate `meta_pool: [4096]u8` with `(offset, len)` references, so the common
  event stays at 12 bytes or less. Return `error.EventCapacityExceeded` when full; never grow.
- **Ticks as u32.** At 480 tpq and 120 BPM (960 ticks/s), 2^32 ticks is about 51 days, which
  is ample. A 4-byte VLQ holds deltas up to `0x0FFFFFFF` (about 77 hours at that rate), so
  assert every delta is ≤ `0x0FFFFFFF`. Rust's `u64` ticks allow longer, invalid VLQs.
- **Stable order without allocation.** Either (a) sort in place by the composite key
  `(tick, sequence)`, where `sequence` is the insertion index (the key is unique, so even an
  unstable sort such as a heap sort gives Rust's order), or (b) keep the array sorted on insert
  with a binary search for the upper bound of `tick`. Option (a) with an explicit `sequence`
  makes stability a checked property instead of something inherited from the algorithm.
  After sorting, assert `a.tick < b.tick or (a.tick == b.tick and a.sequence < b.sequence)`.
- **Tempo** as an integer `tempo_us_per_quarter: u24` computed from fixed-point milli-BPM. See
  [Standard MIDI File](../protocol/standard-midi-file.md#zig--tigerstyle-notes).
- **Writer.** `write(builder, writer) !void` takes a writer parameter (`anytype`, or the std
  writer interface of your Zig version), plus `write_to_buffer(builder, buf: []u8) ![]u8` for
  callers without I/O. Compute the track length in a first pass so no intermediate buffer is
  needed.
- **CC tables** are comptime data: `cc_names: [128]?[]const u8` and
  `patch_cc_plan: [49]PatchCc`, a tagged union of `.stepped`, `.ten_bit`, `.switch` and
  `.mod_fx_sub`, each with a CC number and a field accessor. Generate `patch_ccs` by looping
  over the plan with a bounded `for`. A comptime assertion checks that the plan yields 78
  events and 50 distinct CCs.
- **Legend** writes to a caller buffer. The worst case is 2 header lines (55 bytes) plus 128 ×
  (`"CC 127  "` + the longest name + `\n`), so allocate it from the longest name length at
  comptime.
- **Naming**: `ticks_per_quarter`, `tempo_us_per_quarter`, `duration_ticks`, and `value_10bit`
  (or `value_raw` plus a `u10` type). Let the type system keep CC numbers in range: use `u7`
  for CC numbers and data bytes, so the unmasked-CC bug cannot happen.

```zig
// sketch
const Event = struct { tick: u32, sequence: u16, len: u8, bytes: [3]u8 };

fn event_less_than(_: void, a: Event, b: Event) bool {
    if (a.tick != b.tick) return a.tick < b.tick;
    return a.sequence < b.sequence;
}

pub fn ten_bit_cc(self: *MidiFileBuilder, tick: u32, cc: u7, value: u10) Error!void {
    const status: u8 = 0xB0 | @as(u8, self.channel);
    try self.push(tick, .{ status, 63, @as(u8, @as(u3, @truncate(value))) });
    try self.push(tick, .{ status, cc, @as(u8, @as(u7, @truncate(value >> 3))) });
}
```

## Open questions

- Should the port add the trailing F7 to SysEx, or keep Rust's caller-supplied behavior for
  byte-identical output?
- Should `patch_ccs` also emit CC103 (multi sub-type select) before CC54? The meaning of the
  shape depends on the selected sub-type.
- Is CC96 meant to carry the TX band value? Fixing it changes the bytes compared with the Rust
  output.
- Should the Zig `patch_ccs` match Rust byte-for-byte, keeping the CC96 bug, so cross-language
  golden tests pass? Or diverge and record the divergence on
  [errata](../protocol/errata.md)?

## Sources

- `crates/minilogue-xd/src/midi_file.rs:60-114` — event struct, builder state, defaults.
- `crates/minilogue-xd/src/midi_file.rs:116-191` — generic MIDI event methods.
- `crates/minilogue-xd/src/midi_file.rs:193-305` — XD helpers and named setters.
- `crates/minilogue-xd/src/midi_file.rs:307-828` — `patch_ccs` (helpers 327-392; stepped
  394-536; 10-bit 538-797; switches 799-826).
- `crates/minilogue-xd/src/midi_file.rs:829-917` — `legend` (846-856), `build`, `write_to`.
- `crates/minilogue-xd/src/midi_file.rs:1176-1273` — `patch_ccs` and integration tests.
- `crates/minilogue-xd/src/device.rs:271-339` — `cc_name`.
- `crates/minilogue-xd/src/sysex/program/synth.rs:676-686` — `mod_fx_type_enum`.
- `crates/minilogue-xd/examples/export_midi.rs:17-117` — Berlin School export.
- `crates/minilogue-xd/examples/berlin_school-legend.txt` — golden legend.
- `crates/minilogue-xd/examples/quintal.rs:523-570` — second builder user (HashMap order).
- `crates/minilogue-xd/Cargo.toml:49-50`, `crates/minilogue-xd/src/lib.rs:84` — example entry
  and ungated module.
- `docs/design/05-active/0002-midi-file-export-implementation-plan.md:25-123` — the plan.
- Korg MIDI Impl rev 1.01 §1-1 transmitted CCs (CC 0x58 MOD FX TYPE *2-12; CC 0x60 MOD FX SUB TYPE
  *2-18…*2-21; CC 0x5C–0x5E switches) and §2-1 received CCs (*6-12, *6-18).
