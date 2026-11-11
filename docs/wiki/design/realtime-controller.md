---
title: Realtime controller — immediate-send CC/NRPN/note API
category: design
sources:
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/src/param/encoding.rs
  - crates/minilogue-xd/src/param/nrpn.rs
  - crates/minilogue-xd/src/param/enums.rs
  - crates/minilogue-xd/src/transport.rs
  - docs/wayfinder/research/xd-midi-capabilities.md
related:
  - protocol/cc-map.md
  - protocol/nrpn-map.md
  - protocol/parameter-encodings.md
  - protocol/midi-messages.md
  - design/stepped-params.md
  - design/typed-values.md
  - design/transport.md
  - design/builders.md
  - protocol/errata.md
  - sources/wayfinder-dsl.md
  - zig/module-layout.md
  - zig/test-plan.md
updated: 2026-11-11
---

# Realtime controller — immediate-send CC/NRPN/note API

> `RealtimeController<O: MidiOutput>` is the crate's live-performance façade. It owns an
> output and a channel. Each setter converts one value into one indivisible message group
> (1–4 MIDI messages) and sends it immediately. There are 66 public methods: 31 ten-bit
> float setters, 16 stepped-enum setters, 3 on/off, 3 raw 7-bit, 5 note/program messages,
> and 4 NRPN setters. The controller is stateless apart from its channel. It does not cache
> parameter values, schedule messages or read anything back. A Zig port should keep the
> catalogue, but split it into **pure group encoders** that write into a caller's buffer. The
> send step stays separate, so a scheduler can timestamp and order the groups.

## Facts

### Shape

- `pub struct RealtimeController<O: MidiOutput> { output: O, channel: U4 }`
  (`controller.rs:84-87`). It owns the output by value. `output()` and `output_mut()`
  expose the output, which tests use to read `MockOutput::messages()`
  (`controller.rs:96-108`).
- Every public method returns `Result<()>`. Errors come from value conversion
  (`Error::OutOfRange`) or from `MidiOutput::send` (`controller.rs:113-147`).
- The controller does **not** remember what it sent. It does no deduplication, keeps no
  running status and tracks no notes [rust].

### Private helpers (`controller.rs:110-147`)

| Helper | Wire output | Notes |
|---|---|---|
| `send_cc(n, v)` | `Bn nn vv` | Builds a `ControlChange` from `U7::new(n)` and `U7::new(v)`. The values are always < 128 at call sites, so the `?` only guards against misuse. |
| `send_10bit_cc(n, f)` | `Bn 3F lo` then `Bn nn hi` | `f32_to_10bit(f)`, then `lo = v & 7`, `hi = v >> 3`. **Low part first**, as `*1-4` requires. See [parameter encodings](../protocol/parameter-encodings.md). |
| `send_stepped_cc(n, p)` | `Bn nn tx` | `p.to_tx_value()`, the canonical TX value from [stepped params](stepped-params.md). |
| `send_on_off_cc(n, b)` | `Bn nn 7F` / `Bn nn 00` | 127 or 0. |
| `send_nrpn(p)` | `Bn 63 msb`, `Bn 62 lsb`, [`Bn 3F lo`], `Bn 06 hi` | Each message from `NrpnParam::to_midi_sequence(ch)` goes in a separate `send` call. See [NRPN map](../protocol/nrpn-map.md). |

Each MIDI message is **one `send` call**. A 10-bit setter makes two calls; an NRPN setter
makes three or four.

### Float conversion

- `f32_to_10bit` (`controller.rs:29-39`) accepts `0.0 ≤ v ≤ 1.0` and computes
  `round(v × 1023)` with half away from zero, so 0.5 becomes 512. NaN fails the range test
  and is rejected. The error payload is `value: (v × 1000) as i64, min 0, max 1000`.
- `f32_to_pitch_bend` (`controller.rs:47-57`) accepts `-1.0 ≤ v ≤ 1.0` and computes
  `round(v × 8191)` as an `I14`.
- The canonical formula table is on
  [parameter encodings](../protocol/parameter-encodings.md#float--integer-mappings-controller-api).

### Setter catalogue

**10-bit float setters** (31; `controller.rs:155-310`). Each sends `Bn 3F lo` then `Bn nn hi`.

| CC | Setter | CC | Setter |
|---|---|---|---|
| 16 | `set_amp_eg_attack` | 37 | `set_vco2_shape` |
| 17 | `set_amp_eg_decay` | 39 | `set_vco1_level` |
| 18 | `set_amp_eg_sustain` | 40 | `set_vco2_level` |
| 19 | `set_amp_eg_release` | 41 | `set_cross_mod_depth` |
| 20 | `set_eg_attack` | 43 | `set_cutoff` |
| 21 | `set_eg_decay` | 44 | `set_resonance` |
| 22 | `set_eg_int` | 54 | `set_multi_shape` |
| 24 | `set_lfo_rate` | 59 | `set_voice_mode_depth_alt` |
| 26 | `set_lfo_int` | 104 | `set_multi_shift_shape` |
| 27 | `set_voice_mode_depth` | 105 | `set_delay_time` |
| 28 | `set_mod_fx_time` | 106 | `set_delay_depth` |
| 29 | `set_mod_fx_depth` | 107 | `set_delay_dry_wet` |
| 33 | `set_multi_level` | 108 | `set_reverb_time` |
| 34 | `set_vco1_pitch` | 109 | `set_reverb_depth` |
| 35 | `set_vco2_pitch` | 110 | `set_reverb_dry_wet` |
| 36 | `set_vco1_shape` | | |

**Stepped setters** (16; `controller.rs:316-393`). Each sends one `Bn nn tx`.

| CC | Setter (enum) | CC | Setter (enum) |
|---|---|---|---|
| 23 | `set_eg_target` (`EgTarget`) | 80 | `set_sync` (`Sync`) |
| 48 | `set_vco1_octave` (`VcoOctave`) | 81 | `set_ring` (`Ring`) |
| 49 | `set_vco2_octave` (`VcoOctave`) | 83 | `set_cutoff_keytrack` (`CutoffKeytrack`) |
| 50 | `set_vco1_wave` (`VcoWave`) | 84 | `set_cutoff_drive` (`CutoffDrive`) |
| 51 | `set_vco2_wave` (`VcoWave`) | 88 | `set_mod_fx_type` (`ModFxType`) |
| 53 | `set_multi_type` (`MultiType`) | 89 | `set_delay_sub_type` (`DelaySubType`) |
| 56 | `set_lfo_target` (`LfoTarget`) | 90 | `set_reverb_sub_type` (`ReverbSubType`) |
| 57 | `set_lfo_wave` (`LfoWave`) | | |
| 58 | `set_lfo_mode` (`LfoMode`) | | |

**On/off** (`controller.rs:400-413`): CC92 `set_mod_fx_on`, CC93 `set_delay_on`,
CC94 `set_reverb_on`.

**Raw 7-bit `U7`** (`controller.rs:419-432`): CC1 `set_modulation1`, CC2
`set_modulation2`, CC5 `set_portamento_time`.

**Notes and programs** (`controller.rs:438-474`):

| Method | Wire | Notes |
|---|---|---|
| `play_note(note: U7, vel: U7)` | `9n kk vv` | Built as a raw array, not via `NoteOn`. `vel = 0` is allowed and the device reads it as Note Off. |
| `stop_note(note: U7)` | `8n kk 00` | Velocity 0. The device itself sends `40` ([MIDI messages](../protocol/midi-messages.md)). |
| `pitch_bend(f32)` | `En lsb msb` | `wire = round(v × 8191) + 8192`. Centre `E0 00 40`, +1.0 `E0 7F 7F`, −1.0 `E0 01 00`. |
| `program_change(ProgramNumber)` | `Bn 00 bank`, `Bn 20 slot`, `Cn slot` | `bank = n / 100`, `slot = n % 100` (`sysex/program/mod.rs:52-61`). **Conflicts with the spec**; see errata. |
| `all_notes_off()` | `Bn 7B 00` | |

**NRPN setters** (`controller.rs:485-539`). The address is `MSB 0x00` plus the LSB shown.

| Method | Rust LSB | Spec LSB | Value | Range check |
|---|---|---|---|---|
| `set_bend_range_plus(u8)` | `0x16` | `0x20` | semitones | `≤ 12`, else `OutOfRange` |
| `set_bend_range_minus(u8)` | `0x17` | `0x21` | semitones | `≤ 12` |
| `set_micro_tuning(MicroTuning)` | `0x30` | `0x30` | enum program value | none (enum) |
| `set_program_level(f32 dB)` | `0x3F` | `0x3F` | `round((dB + 18) × 5)` | `-18.0 ≤ dB ≤ 6.0`; 0 dB → 90 |

### What it does **not** cover

- **CCs with no setter:** 0/32 alone, 6/63/98/99 alone, 64 (damper), 96 (mod FX sub type),
  103 (multi select) and 118/119 (CV in). CC96 and CC103 depend on context (their meaning
  follows CC88 and CC53), so they have no typed setter
  ([CC map](../protocol/cc-map.md#context-dependent-ccs-96-103)).
- **NRPN:** only 4 of the 51 Rust variants have setters. All other NRPN parameters need
  `NrpnParam::to_midi_sequence` plus a manual send.
- **Real-time messages, SysEx and reading back:** these are on other pages. See
  [MIDI messages](../protocol/midi-messages.md) and
  [SysEx transactions](sysex-transactions.md).

## Invariants

1. **Each setter sends one message group, in a fixed order.**
   - 10-bit: CC63 low, then the parameter CC high.
   - NRPN: 99, 98, [63], 6.
   - Program select: CC0, CC32, PC.
2. **Validate first, then send.** A conversion error returns before any byte is sent: the
   range check in `f32_to_10bit` or `f32_to_pitch_bend`, or in the bend/level setters
   (`controller.rs:123-127`, `485-494`).
3. **The channel goes into the low nibble** of every status byte (`cc_respects_channel`
   expects `0xB5` on channel 5, `controller.rs:1274-1279`).
4. **Integer outputs are always in range**: 10-bit values 0..1023, bend wire values
   1..16383, program level 0..120, `lo` ≤ 7 and `hi` ≤ 127.

## Errata & gotchas

| # | Item | Detail | Provenance |
|---|---|---|---|
| 1 | Bank select | Rust sends CC0 = bank (0–4) and CC32 = slot (0–99). The spec (§1 `*1-1`) says CC0 = 0 and CC32 = bank 0~4. For program 250 the spec gives `B0 00 00`, `B0 20 02`, `C0 32`; Rust gives `B0 00 02`, `B0 20 32`, `C0 32`. No [hw] capture supports Rust's form, and the crate's own `BankSelect` type follows the spec. | [spec] vs [rust]; `xd-midi-capabilities.md:53` |
| 2 | Bend-range NRPN addresses | Rust uses 0x16/0x17. Spec rev 1.01 puts bend range at 0x20/0x21, and 0x16/0x17 are SHAPE USER and SHIFT SHAPE NOISE. Sending "bend range" from Rust may change an oscillator shape instead. | [spec] vs [rust]; [NRPN map](../protocol/nrpn-map.md) |
| 3 | Micro-tuning values | The address 0x30 matches the spec, but the `MicroTuning` values differ from index 11 up, and user scales (spec 112–123) can't be expressed. | [spec] vs [rust]; [stepped params](stepped-params.md#microtuning-39-variants) |
| 4 | Note Off velocity | Rust sends 0; the device sends 0x40. Both are legal. | [spec] |
| 5 | Pitch bend −8192 | `round(v × 8191)` never produces wire 0. Full down is `E0 01 00`. That is harmless but asymmetric. | [rust] |
| 6 | CC59 resolution | `set_voice_mode_depth_alt` sends 10-bit, but the spec's TX table lists CC59 without `*1-4`. | [spec] inconsistent |
| 7 | Partial groups on I/O failure | Each message is a separate `send`. If the 2nd–4th send fails, the device is left with a pending CC63 low part or a half-set NRPN address. The next 10-bit CC then picks up the stale low bits. | [rust]; see [parameter encodings](../protocol/parameter-encodings.md#errata--gotchas) |
| 8 | Floats in the hot path | Each value passes through `f32` and is rounded. Callers who already hold a 10-bit integer must go through `n / 1023.0` and back. Every integer survives that path exactly, but the step is wasted. | [rust] |
| 9 | Tautological 10-bit tests | `assert_10bit_cc` computes the expected bytes with the same `f32_to_10bit` under test (`controller.rs:586-595`). Only a few tests pin literal bytes. | [rust]; see [test plan](../zig/test-plan.md) |

## Test vectors

| Call (channel 0) | Bytes | Source |
|---|---|---|
| `set_cutoff(0.5)` → 512 | `B0 3F 00` `B0 2B 40` | derived; `controller.rs:598-602` |
| `set_cutoff(1.0)` → 1023 | `B0 3F 07` `B0 2B 7F` | derived |
| `set_cutoff(0.0)` | `B0 3F 00` `B0 2B 00` | derived |
| `set_sync(On)` | `B0 50 7F` | `controller.rs:923-928` |
| `set_mod_fx_on(true)` on ch 5 | `B5 5C 7F` | `controller.rs:1274-1279` |
| `pitch_bend(0.0)` | `E0 00 40` | `controller.rs:1083-1090` |
| `pitch_bend(1.0)` | `E0 7F 7F` | `controller.rs:1092-1099` |
| `pitch_bend(-1.0)` | `E0 01 00` | `controller.rs:1101-1109` |
| `program_change(250)` | `B0 00 02` `B0 20 32` `C0 32` (Rust) | `controller.rs:1118-1130` |
| `all_notes_off()` | `B0 7B 00` | `controller.rs:1133-1140` |
| `set_bend_range_plus(7)` | `B0 63 00` `B0 62 16` `B0 06 07` | `controller.rs:1143-1152` |
| `set_bend_range_minus(5)` | `B0 63 00` `B0 62 17` `B0 06 05` | `controller.rs:1161-1169` |
| `set_micro_tuning(PureMajor)` | `B0 63 00` `B0 62 30` `B0 06 01` | `controller.rs:1178-1190` |
| `set_program_level(0.0)` | `B0 63 00` `B0 62 3F` `B0 06 5A` | `controller.rs:1193-1202` |
| `set_bend_range_plus(13)` | `Err(OutOfRange)`, nothing sent | `controller.rs:1155-1159` |
| `f32_to_10bit(-0.1)`, `f32_to_10bit(1.1)` | `Err(OutOfRange)` | `controller.rs:1244-1252` |
| `set_cutoff(NaN)` | `Err(OutOfRange)`, nothing sent | derived (untested in Rust) |

For the Zig port, use **spec** addresses for the NRPN rows (`B0 62 20` and `B0 62 21` for
bend range) and the **spec** bank-select form. Keep the Rust bytes as "legacy" vectors only
if a compatibility mode is added.

## Rust design

- **A façade over the parameter layer.** Each setter is one line that calls a helper with a
  hard-coded CC number. The CC numbers appear again in `param/cc.rs` (`CcParam`) and
  `device.rs` (`cc_name`). There is no single table, so the controller can drift from the
  receiver.
- **Generic over `MidiOutput`.** `MockOutput` makes the controller fully testable without
  hardware ([transport](transport.md)).
- **Floats for continuous values.** These are friendly for UI sliders but lossy for exact
  automation, and they force the use of `f32` in an otherwise integer protocol.
- **No batching.** Sending a patch over CC (about 60 groups, 120+ messages) is a loop of
  setters. Each message is a separate syscall in the midir backend.

## Zig / TigerStyle notes

- **Split encoding from sending.** Make the core a set of pure functions that return a
  fixed-size group:

  ```zig
  pub const group_bytes_max = 12; // NRPN with CC63: 4 messages × 3 bytes
  pub const Group = struct {
      bytes: [group_bytes_max]u8,
      len: u8,
      pub fn slice(g: *const Group) []const u8 { return g.bytes[0..g.len]; }
  };
  pub fn encode_cc_10bit(channel: u4, cc: CcTenBit, value: u10) Group { ... }
  pub fn encode_cc_stepped(channel: u4, cc: CcStepped, tx: u7) Group { ... }
  pub fn encode_nrpn(channel: u4, param: NrpnParam) Group { ... }
  pub fn encode_program_select(channel: u4, program: ProgramNumber) Group { ... }
  ```

  A thin `Controller` can wrap an output and call `output.write(group.slice())` **once per
  group**. That way a transport that frames per write (CoreMIDI packet lists, ALSA rawmidi
  `write`) sends the group as one unit. This covers the "atomic message groups" need in
  [wayfinder DSL](../sources/wayfinder-dsl.md).
- **Integer API first.** Take `u10` for wide CCs, `u7` for raw CCs and an enum for stepped
  CCs. Put `unit_to_u10(f32) error{OutOfRange}!u10` in a separate helper for UI callers.
  Zig's `u10` makes most range errors impossible at compile time.
- **One comptime parameter table.** Generate the encoder enums (`CcTenBit`, `CcStepped`)
  and the receiver's decode switch from one table. The table holds the CC number, kind,
  stepped enum, TX values and RX bands; see [module layout](../zig/module-layout.md).
  Exhaustive `switch` means a missing CC is a compile error.
- **Spec addresses by default.** The NRPN encoder should use spec rev 1.01 addresses. Tag
  each table row with its evidence level (`spec`, `hw`, `disputed`) so a strict mode can
  refuse disputed parameters ([errata](../protocol/errata.md)).
- **Assertions.** At the end of every encoder, assert that every data byte < 0x80, that the
  status high nibble is in {8,9,B,C,E} and that `group.len` is one of {2,3,6,8,9,12}.
  Assert `lo ≤ 7` before writing CC63.
- **No hidden state.** Like Rust, the Zig controller should be stateless. Note tracking (a
  `[16]u128` bit set for active notes, used for cleanup) belongs to a separate, optional
  `NoteTracker` with a fixed size.
- **Running status is not needed.** USB-MIDI sends whole messages, so always send the full
  status byte. Only a DIN-oriented encoder could add running status as an optimization.

## Open questions

- Which bank-select form does the hardware accept? Capture a Program Change from the
  hardware itself (with MIDI TX on), or test both forms on a device.
- Do the Rust NRPN addresses (0x16/0x17) work on any firmware version, or are they plain
  bugs? No [hw] capture shows them.
- Does the device accept CC59 as 10-bit (CC63 + CC59)?
- Should `stop_note` send velocity 0x40 to match what the device sends?

## Sources

- `crates/minilogue-xd/src/controller.rs:24-57` (float conversion), `:84-147` (struct and
  helpers), `:150-539` (setters), `:545-1280` (tests).
- `crates/minilogue-xd/src/param/nrpn.rs` (`to_midi_sequence`),
  `crates/minilogue-xd/src/param/encoding.rs` (`TenBitParam::lsb`/`msb`).
- `crates/minilogue-xd/src/sysex/program/mod.rs:27-61` (`ProgramNumber`, `bank`,
  `slot_in_bank`).
- Korg MIDI Implementation rev 1.01 §1-1 (`*1-1`, `*1-4`), §3 NRPN table.
- `docs/wayfinder/research/xd-midi-capabilities.md:48-56` (bank select and discrepancies).
