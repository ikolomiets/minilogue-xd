---
title: Multi-byte parameter encodings (10-bit CC, 8-bit hi-res, 14-bit)
category: protocol
sources:
  - crates/minilogue-xd/src/param/encoding.rs
  - crates/minilogue-xd/src/param/cc.rs
  - crates/minilogue-xd/src/param/nrpn.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/tests/cc_coverage.rs
  - crates/minilogue-xd/examples/read_edit_buffer.rs
related:
  - protocol/cc-map.md
  - protocol/nrpn-map.md
  - protocol/midi-messages.md
  - protocol/program-blob.md
  - protocol/errata.md
  - design/stepped-params.md
  - design/realtime-controller.md
  - design/typed-values.md
  - sources/korg-midi-implementation.md
  - glossary.md
updated: 2026-10-11
---

# Multi-byte parameter encodings (10-bit CC, 8-bit hi-res, 14-bit)

> A MIDI data byte holds only 7 bits, so the minilogue xd splits wider values over two Control
> Change messages. The **low part always goes first, on CC63** ("LSB VALUE"). The **high part
> follows** on the parameter's own CC (for the 31 10-bit CC parameters) or on CC6 Data Entry MSB
> (for NRPN values). There are three widths: 10-bit (3 + 7 bits), 8-bit hi-res (3 + 5 bits) and
> 14-bit (7 + 7 bits). The most important fact: a receiver keeps the CC63 value **pending**, adds
> it to the next high part, and **assumes 0** when no CC63 came first. That makes a plain 7-bit CC
> a coarse 10-bit value (`high × 8`). This page also defines the controller's float-to-integer
> mappings.

## Facts

### Encoding table

| Encoding (Rust type) | Range | Low part, sent **first** | High part, sent **second** | Low carrier | High carrier | Used by |
|---|---|---|---|---|---|---|
| 10-bit CC (`TenBitParam`) | 0..1023 | bits 2..0: `v & 0x07` (0..7) | bits 9..3: `(v >> 3) & 0x7F` (0..127) | CC63 `0x3F` | the parameter's own CC | 31 CCs, see [CC map](cc-map.md) |
| 10-bit NRPN (`TenBitSysex`) | 0..1023 | bits 2..0 | bits 9..3 | CC63 | CC6 `0x06` | 6 NRPNs (multi shape / shift shape), see [NRPN map](nrpn-map.md) |
| 8-bit hi-res (`EightBitHighRes`) | 0..200 | bits 2..0: `v & 0x07` | bits 7..3: `(v >> 3) & 0x1F` (0..25) | CC63 | CC6 | 16 NRPNs (joystick/CV ranges, VPM 1–6, user 1–6) |
| 14-bit (`FourteenBitParam`) | 0..16383 | bits 6..0: `v & 0x7F` | bits 13..7: `(v >> 7) & 0x7F` | CC63 | CC6 | 1 NRPN (master volume) |

[spec] The 10-bit CC rule is §1-1 `*1-4` (transmit) and §2-1 `*5-4` (receive). The three NRPN
widths are `*3-1` (10-bit, "lower 3 bits … then the upper 7 bits"), `*3-2` (8-bit, "lower 3 bits
… upper 5 bits") and `*3-3` (14-bit, "lower 7 bits … upper 7 bits"). Each NRPN note adds "(or
assumed 0)" for a missing CC63. The `vv=0~127` in the spec's CC tables is the **high part**
only.

Two facts set this apart from General MIDI:

- **CC63 is not the LSB of CC31.** GM pairs CC *n* with CC *n*+32. The xd instead uses CC63 as one
  shared low-bits register for every wide parameter [spec].
- **The low part is right-aligned and comes first.** The 10-bit low part is the value's bits 2..0,
  placed in CC63 data bits 2..0. Data bits 6..3 are ignored. GM 14-bit sends MSB first and LSB
  second. Here the order is reversed [spec][rust].

### Encode (sender)

```text
encode_ten_bit(v: 0..1023)  -> [ (CC63, v & 0x07), (CCn, (v >> 3) & 0x7F) ]
encode_eight_bit(v: 0..200) -> [ (CC63, v & 0x07), (CC6, (v >> 3) & 0x1F) ]
encode_fourteen(v: 0..16383)-> [ (CC63, v & 0x7F), (CC6, (v >> 7) & 0x7F) ]
```

[rust] The sender **always** emits CC63, even when the low part is 0. This is required for
correctness: the receiver may still hold a pending low value from an earlier, unpaired CC63. The
rule is in `cc.rs:253-301` and `nrpn.rs:216-277`. For the NRPN address bytes that come before
these pairs, see [NRPN map](nrpn-map.md#wire-sequence).

### Decode (receiver)

| Encoding | Formula (`low` = CC63 data, `high` = second data byte) | Result range | Out-of-range handling [rust] |
|---|---|---|---|
| 10-bit CC (stateful receiver) | `((high & 0x7F) << 3) \| (low & 0x07)`; `low = 0` if absent | 0..1023 always | impossible |
| 10-bit `from_parts` | `(high << 3) \| (low & 0x07)`; **high not masked** | 0..2047 | > 1023 → `OutOfRange` (only reachable if `high > 127`) |
| 8-bit `from_parts` | `((high & 0x1F) << 3) \| (low & 0x07)` | 0..255 | > 200 → `OutOfRange` (NRPN receiver drops the message) |
| 14-bit `from_parts` | `((high & 0x7F) << 7) \| (low & 0x7F)` | 0..16383 always | impossible |

Sources: `encoding.rs:74-77`, `encoding.rs:161-164`, `encoding.rs:248-251`, `encoding.rs:401-406`.

The 8-bit decoder masks `high` to 5 bits **before** the range check. So `high = 26..31` gives
208..255 and is rejected, but `high = 32..127` wraps around (`32 → 0`) and is accepted as a
smaller value. See [Errata](#errata--gotchas).

### Pending-low receiver (`TenBitReceiver`)

The 10-bit CC receiver holds one piece of state, `pending_low: optional u3`. It starts empty.

| Event | Action | Output |
|---|---|---|
| CC63 with data `d` | `pending_low = d & 0x07` (overwrites any earlier pending value) | — (the CC map receiver also reports `DataEntryLsb(d)`) |
| 10-bit parameter CC with data `h` | `value = ((h & 0x7F) << 3) \| (pending_low or 0)`; `pending_low = empty` | `value` (0..1023) |
| `reset()` | `pending_low = empty` | — |

Source: `encoding.rs:355-411`. The surrounding CC map receiver decides which other CCs call
`reset()`: only CCs it does not recognise do. Stepped, on/off, raw and NRPN CCs **leave the
pending low untouched**. That table, and the cross-talk it causes, is on
[CC map](cc-map.md#receiver-ccparamreceiver). The NRPN receiver keeps its own data-low slot
inside its state machine; see [NRPN map](nrpn-map.md#receiver-fsm).

Decoding needs no timeout and no channel check. A pending low lives until it is consumed or reset.

### Value semantics of the wide encodings

| Encoding | Meaning of raw value | Source |
|---|---|---|
| 10-bit CC | Linear knob position 0..1023. Parameter-specific curves (for example VCO pitch → cents) live in the program-blob notes, see [program blob](program-blob.md). | [spec] `*1-4`, TABLE 2 notes |
| 8-bit hi-res, bipolar ranges | `0..200` = −100 %..+100 %, so percent = `raw − 100` | [spec] `*3` rows 0x23/0x25/0x2A/0x2C/0x40–0x45 |
| 8-bit hi-res, user params | Depends on the user-oscillator param type (`*note P24`): Percent 0..101, Bipolar 0..200, Select 0..100 (shown as 1..101). All fit in 0..200. | [spec] `*note P23`, `*note P24` |
| 14-bit | Master volume 0..16383 | [spec] `*3-3`, `*3-4` |

### Float ↔ integer mappings (controller API)

[rust] `RealtimeController` takes floats for continuous values and converts them **before**
sending anything. These are the canonical rules; the [realtime controller](../design/realtime-controller.md)
page links here.

| Mapping | Accepted input | Formula | Output range | Error (`Error::OutOfRange`) | Source |
|---|---|---|---|---|---|
| unit → 10-bit | `0.0 ≤ v ≤ 1.0` (NaN rejected, because the range test is false) | `round_half_away_from_zero(v × 1023)` in `f32` | 0..1023 | `{type_name:"f32 (0.0-1.0)", value:(v×1000) as i64, min:0, max:1000}` | `controller.rs:24-39` |
| bipolar → pitch bend | `-1.0 ≤ v ≤ 1.0` | `bend = round(v × 8191)`; `wire = bend + 8192`; bytes `En (wire & 0x7F) (wire >> 7)` | bend −8191..8191, wire 1..16383 | `{"f32 (-1.0-1.0)", (v×1000) as i64, -1000, 1000}` | `controller.rs:41-57`, `controller.rs:452-460` |
| dB → program level | `-18.0 ≤ db ≤ 6.0` | `round((db + 18) × 5)`, so 0.2 dB per step | 0..120 | `{"program_level_db", (db×10) as i64, -180, 60}` | `controller.rs:526-538` |

Notes:

- `0.5 × 1023 = 511.5` is exact in `f32` and rounds **up** to 512. Rounding is half away from zero,
  not half to even.
- Pitch-bend wire value 0 (−8192) is never produced. Full down is `E0 01 00`, not `E0 00 00`. For
  the pitch-bend message itself, see [MIDI messages](midi-messages.md).
- The program blob stores program level with an offset (12..132). See
  [program blob](program-blob.md).
- The inverse mapping used for code generation is `raw / 1023` (`read_edit_buffer.rs:22-25`). It is
  printed with **3 decimals**, which loses information: 23 of the 1024 raw values (22, 67, 111, …,
  511, …) do not survive `raw → "%.3f" → round(×1023)`. With 4 decimals, all 1024 survive (checked
  by brute force while writing this page).

## Invariants

- `encode_ten_bit(v)`: `v <= 1023`; `low <= 7`; `high <= 127`; `(high << 3) | low == v`.
- `encode_eight_bit(v)`: `v <= 200`; `low <= 7`; `high <= 25`; `(high << 3) | low == v`.
- `encode_fourteen(v)`: `v <= 16383`; `low <= 127`; `high <= 127`; `(high << 7) | low == v`.
- Every encoded pair is ordered `[CC63, carrier]`. CC63 is never the last message of a write.
- Decoding the output of encoding returns the input for every value in range (pair assertion).
- The 10-bit receiver output is always `<= 1023`. After it produces a value, `pending_low` is
  empty.
- `pending_low`, when present, is `<= 7`.
- `comptime`: `(0x7F << 3) | 0x07 == 1023`, `(25 << 3) | 7 == 207 >= 200`,
  `(0x7F << 7) | 0x7F == 16383`.
- Float input to the unit mapping is finite and in `[0, 1]`. The output is `<= 1023`.
  `unit(0.0) == 0`, `unit(1.0) == 1023`.
- Pitch-bend wire value is in `1..16383`, and the center (0.0) maps to 8192 (`E0 00 40`).

## Errata & gotchas

- **Standard MIDI tools misread CC63.** A DAW that treats CC63 as "LSB of CC31" (or ignores it)
  shows the xd's wide parameters as unrelated messages. A sender that puts the LSB after the MSB
  (GM order) has its low bits applied to the *next* wide value instead [spec].
- **NRPN data LSB is CC63, not CC38.** Standard MIDI uses CC38 for the data-entry LSB. The xd
  documents only CC63 (`*3-1`…`*3-3`) [spec]. The Rust NRPN receiver treats CC38 as an unrelated
  CC and **resets** its state machine. See [NRPN map](nrpn-map.md).
- **CC59 (alternate voice mode depth) resolution is ambiguous.** The transmit table lists CC59
  *without* `*1-4`, while the receive table lists it *with* `*5-4` [spec]. Rust treats CC59 as
  10-bit in both directions [rust]. See [CC map](cc-map.md#errata--gotchas).
- **8-bit high part > 25.** Rust masks the high part to 5 bits before the range check
  (`encoding.rs:161-164`). High 26..31 is rejected; high 32..127 silently aliases to a small value
  [rust]. The device's behaviour is unknown. Recommendation for the port: reject `high > 25`.
- **`TenBitParam::from_parts` does not mask the high part**, unlike `TenBitReceiver::take_value`
  and the 14-bit decoder (`encoding.rs:74-77` vs `encoding.rs:401-406`). Inputs coming from 7-bit
  data bytes can never trigger this. It is an inconsistency, not a live bug [rust].
- **Stale pending low.** A CC63 that is never followed by a 10-bit CC stays pending indefinitely
  and changes the next 10-bit value. The low part of an NRPN multi-byte write is one example. See
  [CC map](cc-map.md#receiver-ccparamreceiver) [rust]. Whether the device behaves the same way is
  not documented.
- **Doc comment says "CC parameters" for `EightBitHighRes`** (`encoding.rs:104-107`). It is used
  only by NRPN. Comment drift [rust].

## Test vectors

Bytes are on MIDI channel 1 (`B0`/`E0`). "Asserted" means a test checks these exact values.
"Derived" means the bytes are computed from asserted parts using the algorithm above.

| Case | Bytes / values | Status | Source |
|---|---|---|---|
| 10-bit split of 517 | low 5, high 64 | asserted | `encoding.rs:442-448` |
| Cutoff = 517 | `B0 3F 05` `B0 2B 40` (controllers 63, 43) | controllers asserted, bytes derived | `cc.rs:1180-1194` |
| CC63=5, CC43=100 → Cutoff | 805 | asserted | `tests/cc_coverage.rs:101-121` |
| 10-bit `from_parts` round trip | 0, 1, 7, 8, 100, 511, 512, 1023 | asserted | `encoding.rs:451-457` |
| `from_parts(0xFF, 0)` | 7 (low masked) | asserted | `encoding.rs:460-464` |
| 8-bit split of 200 | low 0, high 25 | asserted | `encoding.rs:512-518` |
| 8-bit `from_parts(0x07, 0xFF)` | error (`(0xFF & 0x1F) << 3 \| 7 = 255`) | asserted | `encoding.rs:530-539` |
| 8-bit `from_parts(0, 25)` | 200 | asserted | `encoding.rs:530-539` |
| 14-bit split of 16383 | low 127, high 127 | asserted | `encoding.rs:579-585` |
| 14-bit `from_parts(0xFF, 0xFF)` | 16383 | asserted | `encoding.rs:597-601` |
| Receiver: CC63=5, high 64 | 517 | asserted | `encoding.rs:678-684` |
| Receiver: high 64, no CC63 | 512 | asserted | `encoding.rs:686-692` |
| Receiver: CC63=7, reset, high 10 | 80 | asserted | `encoding.rs:694-701` |
| Receiver: CC63=3, high 1, high 1 | 11, then 8 (low consumed) | asserted | `encoding.rs:703-712` |
| Receiver: CC63=0xFF, high 0 | 7 | asserted | `encoding.rs:714-720` |
| Receiver: high 0xFF, no CC63 | 1016 (high masked) | asserted | `encoding.rs:722-729` |
| Receiver: CC63=7, high 127 | 1023 | asserted | `encoding.rs:731-737` |
| NRPN 10-bit, value 500 | `B0 63 00` `B0 62 10` `B0 3F 04` `B0 06 3E` | controller order asserted, bytes derived | `nrpn.rs:1404-1414` |
| NRPN 8-bit, value 150 | `… B0 3F 06` `B0 06 12` | round trip asserted, bytes derived | `tests/nrpn_coverage.rs:51-57` |
| NRPN 14-bit, value 10000 | `B0 63 00` `B0 62 7F` `B0 3F 10` `B0 06 4E` | round trip asserted, bytes derived | `tests/nrpn_coverage.rs:59-65` |
| unit → 10-bit | 0.0 → 0, 0.5 → 512, 1.0 → 1023; −0.001 and 1.001 rejected | asserted | `controller.rs:1232-1252` |
| `set_cutoff(0.5)` | `B0 3F 00` `B0 2B 40` | derived from 512 | `controller.rs:598-602` |
| Pitch bend | 0.0 → `E0 00 40`; 1.0 → `E0 7F 7F`; −1.0 → `E0 01 00`; ±1.1 rejected | asserted | `controller.rs:1083-1115` |
| Program level | 0 dB → 90 (`0x5A`); −18 dB → 0; +6 dB → 120 (`0x78`) | asserted | `controller.rs:1193-1220` |

Property tests (`encoding.rs:753-800`) check: round trip for every in-range value of all three
widths, rejection of every out-of-range value, and receiver output `<= 1023` for any `(low, high)`
pair.

## Rust design

- **Four newtypes** wrap a private integer (`u16` or `u8`) behind a range-checked `new`. Each has
  `value()`, `lsb()`, `msb()`, `from_parts(lsb, msb)`, `TryFrom`, `From<…> for u16/u8` and
  `Display`. The value is always valid by construction. That part is essential; the newtype
  machinery is incidental Rust idiom.
- **`TenBitParam` vs `TenBitSysex`** have identical encodings (`encoding.rs:276-331`). They exist
  only to keep CC and NRPN values from being mixed up in the type system. In Zig a single `u10`
  is enough; the carrier (CC vs NRPN) belongs to the message, not the value.
- **`TenBitReceiver`** is `Option<u8>` with three methods (`encoding.rs:381-411`). It is embedded
  in `CcParamReceiver`. The NRPN receiver does not use it; it keeps the data low in its own state.
- **Float helpers are private** to `controller.rs`. The protocol layer works only with integers.
  Keep that split.
- Errors are a single `Error::OutOfRange { type_name, value: i64, min, max }`. See
  [errors](../design/errors.md).

## Zig / TigerStyle notes

- Use exact-width integers: `u3` (10/8-bit low part), `u5` (8-bit high part), `u7` (14-bit parts,
  10-bit high, every data byte), `u10`, `u14`. Store 8-bit hi-res as `u8` and assert `<= 200`.
  The type cannot express that bound, so an assertion has to.
- Split and join as pure functions, each paired with an assertion that the other undoes it:

  ```zig
  // sketch
  const TenBitParts = struct { low: u3, high: u7 };

  fn ten_bit_split(value: u10) TenBitParts {
      const parts = TenBitParts{ .low = @truncate(value), .high = @truncate(value >> 3) };
      assert(ten_bit_join(parts) == value);
      return parts;
  }

  fn ten_bit_join(parts: TenBitParts) u10 {
      return (@as(u10, parts.high) << 3) | parts.low;
  }

  comptime {
      assert((@as(u10, maxInt(u7)) << 3 | maxInt(u3)) == maxInt(u10));
  }
  ```

- Decoding from wire bytes takes `u7` inputs. There is nothing to mask except dropping CC63 data
  bits 6..3 for the 3-bit low part. For 8-bit hi-res, return `error.ValueOutOfRange` when
  `high > 25` or the joined value exceeds 200. Do not copy Rust's 5-bit wrap.
- Receiver state is `pending_low: ?u3 = null`. That is one byte with no allocation and no timeout.
  Decide explicitly (and document) which CCs clear it; see [CC map](cc-map.md#zig--tigerstyle-notes).
- Keep float conversion at the API edge (`unit_to_ten_bit(value: f32) error{ValueOutOfRange}!u10`).
  Reject NaN with `std.math.isNan` or a range test that is false for NaN. Compute in `f32` to stay
  bit-compatible with the Rust tests. Zig's `@round` is documented to round halfway cases away from
  zero, matching Rust's `f32::round`. Pin it with the 0.5 → 512 test anyway.
- Name the units: `program_level_db`, `bend_range_semitones`, `value_unit` (0..1).
- Maximum wire output for one wide value is 2 CCs (6 bytes). With the NRPN address it is 4 CCs
  (12 bytes). See [memory budget](../zig/memory-budget.md).

## Open questions

- Does the device's own receiver keep a CC63 pending across unrelated CCs, stepped CCs, or an NRPN
  sequence, as the Rust receiver does? Does it ever time out? This needs a hardware test: send
  `B0 3F 07`, `B0 50 7F`, `B0 2B 40` and read the cutoff from a program dump.
- Does the device send CC63 when the low part is 0? The spec says the low bits are "first sent";
  no capture is in the repo.
- What does the device do with an 8-bit high part > 25, or a joined value > 200?
- Is CC59 sent as 7-bit (no CC63) by the hardware? See [CC map](cc-map.md#open-questions).

## Sources

- `crates/minilogue-xd/src/param/encoding.rs:16-77` — `TenBitParam` split, join, range.
- `crates/minilogue-xd/src/param/encoding.rs:104-164` — `EightBitHighRes`.
- `crates/minilogue-xd/src/param/encoding.rs:191-251` — `FourteenBitParam`.
- `crates/minilogue-xd/src/param/encoding.rs:276-331` — `TenBitSysex`.
- `crates/minilogue-xd/src/param/encoding.rs:355-411` — `TenBitReceiver`.
- `crates/minilogue-xd/src/param/encoding.rs:442-800` — unit and property tests.
- `crates/minilogue-xd/src/param/cc.rs:253-301` — CC63-first emission for 10-bit CCs.
- `crates/minilogue-xd/src/param/nrpn.rs:216-277` — CC63 + CC6 emission for NRPN wide values.
- `crates/minilogue-xd/src/param/nrpn.rs:632-646` — NRPN wide-value decode helpers (missing low = 0).
- `crates/minilogue-xd/src/controller.rs:24-57`, `:452-460`, `:526-538` — float mappings.
- `crates/minilogue-xd/src/controller.rs:1083-1115`, `:1193-1252` — float mapping tests.
- `crates/minilogue-xd/tests/cc_coverage.rs:101-121` — 805 vector.
- `crates/minilogue-xd/examples/read_edit_buffer.rs:22-25` — 3-decimal float code generation.
- Korg MIDI Impl rev 1.01 §1-1 `*1-4`; §2-1 `*5-4`; `*3` NRPN `*3-1`, `*3-2`, `*3-3`, `*3-4`;
  TABLE 2 `*note P23`, `*note P24` — see [source page](../sources/korg-midi-implementation.md).
