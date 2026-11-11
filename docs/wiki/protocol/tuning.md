---
title: Tuning — user scale, user octave, MIDI Tuning Standard
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/tuning.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/src/param/enums.rs
  - docs/korg-user-guide/book.md
related:
  - protocol/sysex-framing.md
  - protocol/korg-7bit-codec.md
  - protocol/program-blob.md
  - protocol/nrpn-map.md
  - protocol/errata.md
  - design/sysex-transactions.md
  - design/stepped-params.md
  - design/errors.md
  - zig/memory-budget.md
updated: 2026-11-11
---

# Tuning — user scale, user octave, MIDI Tuning Standard

> The xd stores 6 **user scales** and 6 **user octaves**:
> - A user scale is 128 notes × 3 bytes (TABLE 3).
> - A user octave is 12 notes × 3 bytes (TABLE 4).
>
> Each note is a semitone byte plus a 14-bit fraction of a semitone (1/16384 semitone ≈ 0.0061 cent).
> These tables can be moved four ways:
> - Korg exclusive requests `0x14`/`0x15` and dumps `0x44`/`0x45`.
> - The universal MIDI Tuning Standard (MTS) **bulk dump** (`F0 7E … 08 01`).
> - MTS **single note change** (`F0 7F … 08 02`).
>
> The device accepts both MTS messages, but only while a user scale or octave edit page is shown.
>
> The Rust crate diverges from the Korg spec in four ways, none verified on hardware:
> - It omits the scale-number byte `ss`.
> - It 7-bit-encodes the `0x44`/`0x45` payload.
> - It leaves `0x7E` out of the MTS checksum.
> - It sends single-note changes as `7E` (non-realtime) instead of `7F`.

## Note encoding (`CentOffset`)

Each note is 3 bytes; all are 7-bit data bytes [spec] TABLE 3/4:

| Byte | Bits | Meaning |
|-----:|------|---------|
| 0 | `0sss ssss` | semitone `xx` (TABLE 3: 0–127; TABLE 4: see below) |
| 1 | `0hhh hhhh` | fraction bits 13..7 |
| 2 | `0lll llll` | fraction bits 6..0 |

```text
fraction (u14) = (byte1 & 0x7F) << 7 | (byte2 & 0x7F)       0..16383
pitch          = semitone + fraction / 16384   semitones     (unit: 100/16384 = 0.006103515625 cent)
encode: byte1 = (fraction >> 7) & 0x7F ; byte2 = fraction & 0x7F
```

The spec's ".0061-cent units" is `100/16384` cent [spec] §2-7. In MTS (MMA) terms, `xx` is the
equal-tempered MIDI note at or below the target pitch and `yyzz` is the fraction above it.

### Float conversion (Rust)

The Rust crate provides two conversions:

| Function | Algorithm (all `f32`) | Citation |
|----------|-----------------------|----------|
| `to_cents()` | `semitone·100 + fraction·(100/16384)` | `crates/minilogue-xd/src/sysex/tuning.rs:61-63` |
| `from_cents(semitone, cents)` | `c = clamp(cents, 0, 100 − 100/16384)` → `round(c·16384/100)` (half away from zero) → `min(·, 16383)` | `tuning.rs:69-74` |

- `cents` in `from_cents` is the **fractional part only** (0 ≤ c < 100), not absolute cents.
- `100 − 100/16384 = 99.993896484375` and `16384·99.993896484375/100 = 16383` are both exact in `f32`.
  The `min(16383)` is therefore defensive.
- `to_cents` is exact only while the result fits a 24-bit mantissa. For example, semitone 127 with
  fraction 1 (12700.0061…) rounds in `f32`.
- `to_cents` does **not** interpret the signed user-octave semitone; for example 116 yields 11600.

| `from_cents(_, c)` | fraction | Note |
|-------------------:|---------:|------|
| 0.0 | 0 | |
| 25.0 | 4096 | |
| 50.0 | 8192 | `tuning.rs:557-562` |
| 33.333 | 5461 | 5461.28 → 5461 |
| 0.0030517578125 | 1 | exactly 0.5 step, rounds away from zero |
| 99.99 | 16382 | 16382.36 |
| 200.0 | 16383 | clamped (`tuning.rs:571-575`) |
| −10.0 | 0 | clamped (`tuning.rs:565-568`) |

## User scale — TABLE 3 (384 bytes)

- `offset = note·3` for `note = 0..127`. Each entry uses the 3-byte note encoding above; the
  semitone range is 0–127 [spec] TABLE 3.
- Equal temperament: `semitone = note`, `fraction = 0` [rust] (`tuning.rs:130-139`).
- The scale is absolute: key `n` sounds at pitch `semitone + fraction/16384`. Program-level
  **Scale Key** transposes the tuning; it has no effect with Equal Temp
  (`docs/korg-user-guide/book.md:1474-1478`).

## User octave — TABLE 4 (36 bytes)

- `offset = pc·3` for pitch class `pc` = C, C♯, D, … B (0..11).
- The semitone byte is a **signed 7-bit note offset** [spec] TABLE 4:

| Wire value | Meaning |
|-----------:|---------|
| 0–23 | +0 … +23 notes |
| 24–115 | not allowed |
| 116–127 | −12 … −1 notes (`value − 128`) |

- Encode `offset` (−12…+23) as `offset & 0x7F`. Decode as `v ≥ 64 ? v − 128 : v`, then reject
  anything outside −12…+23.
- Equal temperament in Rust is `semitone = pc` (0..11), `fraction = 0` (`tuning.rs:195-204`). This
  reading treats the offset as relative to the octave's C. The spec does not say whether the
  offset is relative to C or to the key itself; in the latter reading equal temperament would be
  all zeros. See open questions.
- The octave pattern repeats across the keyboard (`docs/korg-user-guide/book.md:2023`, `2097`).

## Korg exclusive messages `0x14` / `0x15` / `0x44` / `0x45`

The header is `F0 42 3g 00 01 51` (see [SysEx framing](sysex-framing.md)). The spec prints
`00 01 44` in these sections; this is a typo.

| Fn | Name | Spec payload [spec] | Rust payload [rust] | Spec len | Rust len |
|----|------|---------------------|---------------------|---------:|---------:|
| `0x14` | USER SCALE DATA DUMP REQUEST | `ss` = scale no. (bits 2..0) | none | 9 | 8 |
| `0x15` | USER OCTAVE DATA DUMP REQUEST | `ss` = octave no. (bits 2..0) | none | 9 | 8 |
| `0x44` | USER SCALE DATA DUMP | `ss` + 384 data bytes (NOTE 3), no NOTE-1 reference → raw | 384 bytes → [7-bit encoded](korg-7bit-codec.md) to 439 bytes, no `ss` | 393 | 447 |
| `0x45` | USER OCTAVE DATA DUMP | `ss` + 36 data bytes (NOTE 4) → raw | 36 → 42 encoded, no `ss` | 45 | 50 |

Device behavior per the spec [spec] §1-4 *4, (7)–(10-2):

- **Requests.** `0x14` is answered with `0x44`, and `0x15` with `0x45`. On error the device
  answers `0x24` DATA LOAD ERROR.
- **Dump with `ss` 0–5.** `0x44`/`0x45` are saved to **internal memory**. This number most likely
  maps to USER SCALE/OCTAVE 1–6; the spec does not state the range.
- **Dump with `ss = 0x7F`.** "Current" — saved to the **edit buffer** ((9-2), (10-2)).
- **Reply to a received dump.** `0x23` ACK or `0x24` NAK.
- **Transmission.** The device sends `0x44`/`0x45` on request (R) and from the MIDI dump page (D),
  i.e. the SCALE DUMP button (`docs/korg-user-guide/book.md:2043-2055`).

What the Rust parser sees if the device sends spec-shaped data. This is derived: the payload is
run through `parse_sysex` (NOTE-1 decode), then `UserScale::from_bytes` / `UserOctave::from_bytes`:

| Device sends | Payload bytes on wire | After Rust decode | Rust result |
|--------------|----------------------:|------------------:|-------------|
| scale, `ss` + raw 384 | 385 | 336 | `PayloadTooShort { expected: 384, actual: 336 }` |
| scale, `ss` + encoded 385 | 440 | 385 | **silent garbage** (every note shifted by one byte) |
| octave, `ss` + raw 36 | 37 | 32 | `PayloadTooShort { expected: 36, actual: 32 }` |
| octave, `ss` + encoded 37 | 43 | 37 | **silent garbage** |

Transactions in Rust (see [SysEx transactions](../design/sysex-transactions.md)):
- `request_user_scale` sends `0x14` and expects `0x44`; `request_user_octave` sends `0x15` and
  expects `0x45`.
- `send_user_scale` / `send_user_octave` send the dump and wait for an ACK.
- See `crates/minilogue-xd/src/sysex/transaction.rs:299-339`.

## MTS bulk tuning dump (universal non-realtime, received only)

Fixed 408 bytes [spec] §2-7:

| Offset | Bytes | Value | Device use |
|-------:|------:|-------|------------|
| 0 | 1 | `F0` | |
| 1 | 1 | `7E` universal non-realtime | |
| 2 | 1 | `nn` device ID | ignored |
| 3 | 1 | `08` MIDI Tuning Standard | |
| 4 | 1 | `01` bulk dump | |
| 5 | 1 | `tt` tuning program | ignored |
| 6–21 | 16 | name, ASCII | ignored |
| 22–405 | 384 | 128 × (`xx yy zz`) — same layout as TABLE 3 | applied |
| 406 | 1 | checksum `ss` | |
| 407 | 1 | `F7` | |

- **Checksum** [spec] §2-7, which agrees with the MMA MTS definition:
  `ss = (XOR of bytes[1..406)) & 0x7F`. That covers `7E` through the last data byte, excluding
  `F0`, `ss` and `F7`.
- **Rust checksum** = `XOR of bytes[2..406)`. It **omits `0x7E`**, so `rust_cs = spec_cs ^ 0x7E`
  for every message (`tuning.rs:314-319`, `355-369`). Rust and spec checksums can never agree:
  - Rust rejects every spec-correct dump with `ChecksumMismatch`.
  - The device will most likely reject Rust-built dumps (not tested on [hw]).
- **Reception.** Only while the USER SCALE or USER OCTAVE edit page is displayed. The data
  overwrites the scale being edited, whatever `nn`/`tt` say. For a user octave only notes 60–71
  are used [spec] §2-7; `docs/korg-user-guide/book.md:2017`, `2027`.
- The device never transmits MTS; it is absent from §1.

Rust builder `build_mts_bulk_dump(device_id, program, name, scale)` (`tuning.rs:285-322`):
- Masks `device_id` and `program` with `0x7F`.
- Writes the name as `name.as_bytes()[i] & 0x7F` for i < 16, padded with spaces. Truncation is by
  **byte**, so UTF-8 is mangled rather than rejected.
- Masks every note byte with `0x7F`, which silently strips semitone bit 7.

Rust parser `parse_mts_bulk_dump` (`tuning.rs:330-384`), in order:
1. `len >= 408`, else `PayloadTooShort`.
2. `[0..2] == F0 7E`.
3. `[3..5] == 08 01`.
4. Last byte is `F7` (these three checks fail with `InvalidHeader`).
5. Checksum at `len−2`, else `ChecksumMismatch { expected: computed, actual: stored }`.
6. Notes are read from offset 22.

Longer input is accepted. Note that a frame longer than 408 bytes still reads notes from 22 but the
checksum from `len−2`.

## MTS single note tuning change (universal **realtime**, received only)

`F0 7F nn 08 02 tt ll (kk xx yy zz)×ll F7` [spec] §2-8. Length = `8 + 4·ll`, with
`ll` 0–127, so the maximum is 516 bytes.

| Offset | Value | Notes |
|-------:|-------|-------|
| 1 | `7F` realtime | **Rust uses `7E`** (`tuning.rs:399`) and requires `7E` when parsing (`tuning.rs:435`) |
| 2 | `nn` device ID | ignored by the device |
| 3–4 | `08 02` | |
| 5 | `tt` tuning program | ignored by the device |
| 6 | `ll` count | |
| 7 + 4i | `kk` key 0–127, `xx`, `yy`, `zz` | same 3-byte note encoding |
| last | `F7` | |

- Reception conditions match the bulk dump: edit page only, and octave = notes 60–71 [spec] §2-8.
- **Rust builder** (`tuning.rs:391-416`):
  - `count = min(changes.len(), 127)`. Extra changes are **silently dropped**.
  - Every byte is masked with `0x7F`.
- **Rust parser** (`tuning.rs:425-468`), in order:
  1. `len >= 8`.
  2. `F0 7E`.
  3. `08 02`.
  4. Last byte is `F7`.
  5. `len >= 7 + 4·ll + 1`.
  - `kk` is **not** masked, and trailing bytes are ignored.
- MMA defines `xx yy zz = 7F 7F 7F` as "no change". Neither the Korg spec nor Rust mention it.

## Selecting a tuning (cross-reference)

A program selects its tuning through the MICRO TUNING value. The spec value lists are:

| Context | Named scales | USER SCALE 1–6 | USER OCTAVE 1–6 |
|---------|--------------|----------------|-----------------|
| NRPN `0x30` value (*3-6) | 0–22 | 112–117 | 118–123 |
| Program blob offset 122 (*note P21) | 0–22 | 128–133 | 134–139 |

The named scales are Equal Temp … Reverse, AFX001–006, DC001–003. The Rust `MicroTuning` enum
uses a **different** dense 0–38 list:
- Named scales include Blues, Diminished and four Ragas.
- UserScale1 = 21 and UserOctave1 = 27.
- Reverse1/2, then AFX001–004.

See `crates/minilogue-xd/src/param/enums.rs:551-593`. This needs [hw] verification; details are on
[program blob](program-blob.md), [NRPN map](nrpn-map.md) and
[stepped params](../design/stepped-params.md).

## Invariants

- `CentOffset`: `semitone <= 127`, `fraction <= 16383`, and every wire byte `<= 0x7F`.
- Encoding: `from_bytes(to_bytes(x)) == x` for every valid `x` (`tuning.rs:509-519`).
- User scale blob length is 384; user octave blob length is 36.
- `encode(scale) == out[0..384]` with `out[3n] == semitone[n]`.
- User octave semitone byte ∈ `0..=23 ∪ 116..=127`.
- MTS bulk: `len == 408`, `[1] == 0x7E`, `[3..5] == 08 01`, `[406] == xor(msg[1..406]) & 0x7F`.
- MTS single note: `[1] == 0x7F`, `[3..5] == 08 02`, `len == 8 + 4·[6]`, `[6] <= 127`.
- Korg `0x44` frame (spec) length 393; `0x45` length 45; requests length 9.

## Errata & gotchas

| # | Issue | Provenance | Resolution |
|---|-------|------------|------------|
| 1 | MTS bulk checksum omits `0x7E` in Rust | [rust] vs [spec] (+MMA) | Include `0x7E`: XOR `bytes[1..len-2]`. |
| 2 | Single note change built and parsed as `7E` | [rust] vs [spec] | Use `7F` (realtime). Optionally accept `7E` on parse for Rust interop. |
| 3 | `0x14`/`0x15` lack `ss`; `0x44`/`0x45` lack `ss` and are NOTE-1 encoded | [rust] vs [spec] | Follow the spec (raw, `ss` first) until a [hw] capture says otherwise. Make the encoding a single switch so a capture can settle it. |
| 4 | Spec headers in (7)–(10-2) read `00 01 44`. Function binary columns show `0x18`/`0x19`/`0x48`/`0x49` for `14`/`15`/`44`/`45`. (9-1) says "Func=1C". (8)'s function row and (10-2)'s title say USER SCALE for octave messages | [spec] typos | Trust the hex function IDs and `00 01 51`. `0x44` is probably another Korg product's model ID (copy-paste) — unverified. |
| 5 | TABLE 4 semitone is signed (−12…+23); Rust stores raw `u8` and `to_cents` treats 116 as +116 semitones | [spec] vs [rust] | Model it as a signed offset type in Zig. |
| 6 | MTS is accepted only on the scale/octave edit page, and always overwrites the scale being edited | [spec] [user guide] | A host must tell the user to open the page first. A transaction that expects an ACK will time out: MTS has no ACK. |
| 7 | For user octaves, MTS keys 60–71 supply the 12 pitch classes | [spec] | When converting an octave to MTS, write keys 60–71. Semantics of `xx` there (absolute 60+k?) are unverified. |
| 8 | MicroTuning value lists differ between spec and Rust enum | [spec] vs [rust] | See [program blob](program-blob.md); not owned here. |
| 9 | The "SCALE DUMP" button "transmits as general SysEx" | user guide vs [spec] §1-4 | Per §1-4 (*4 = D) this means `0x44`/`0x45`, not MTS. |

## Test vectors

| Case | Bytes / result | Citation |
|------|----------------|----------|
| `CentOffset::from_bytes([60, 7F, 7F])` | semitone 60, fraction 16383 | `tuning.rs:490-495` |
| `fraction = 0xFFFF` → `to_bytes` → `from_bytes` | 16383 (masked to 14 bits) | `tuning.rs:578-586` |
| `{60, 8192}.to_cents()` | 6050.0 | `tuning.rs:540-547` |
| Rust scale request, ch 0 | `F0 42 30 00 01 51 14 F7` | `tuning.rs:676-680` |
| Rust octave request, ch 3 | `F0 42 33 00 01 51 15 F7` | `tuning.rs:697-701` |
| Spec scale request, scale #3 (`ss = 2`), ch 0 (derived) | `F0 42 30 00 01 51 14 02 F7` | §(7) |
| Rust ET octave dump, ch 7 (50 bytes, derived) | `F0 42 37 00 01 51 45 00 00 00 00 01 00 00 02 00 00 00 03 00 00 04 00 00 00 05 00 00 06 00 00 00 07 00 00 08 00 00 09 00 00 00 0A 00 00 0B 00 00 00 F7` | `tuning.rs:704-709` (round trip) |
| Spec ET octave dump, `ss = 0`, ch 0 (45 bytes, derived) | `F0 42 30 00 01 51 45 00` + `00 00 00 01 00 00 02 00 00 … 0B 00 00` + `F7` | §(10-1) |
| MTS bulk header (`0x10`, prog 5, "Hello") | `F0 7E 10 08 01 05 48 65 6C 6C 6F 20 …`, len 408, last `F7` | `tuning.rs:732-749` |
| MTS bulk name "AB" | `[6]=41 [7]=42 [8..=21]=20` | `tuning.rs:752-759` |
| MTS bulk ET, dev `7F`, prog 0, "Test" | Rust checksum `0x40`; spec checksum `0x3E` | derived from `tuning.rs:771-777` |
| MTS bulk ET, dev `00`, prog 0, "AB" | Rust `0x0A`; spec `0x74` | derived |
| MTS bulk ET, dev `7F`, prog 0, "Test Scale" | Rust `0x38`; spec `0x46` | derived |
| MTS bulk with `msg[30] ^= 1` | `ChecksumMismatch` | `tuning.rs:771-777` |
| MTS bulk, `[1] = 7D` or `[3] = 09` | `InvalidHeader` | `tuning.rs:785-801` |
| Single note, Rust: dev `7F`, prog 3, key 60 → {69, 4096} | `F0 7E 7F 08 02 03 01 3C 45 20 00 F7` (spec form: `F0 7F …`) | `tuning.rs:824-835` |
| Single note, empty | `F0 7E 7F 08 02 00 00 F7` (len 8) | `tuning.rs:817-821` |
| Single note, `F0 7E 00 08 02 00 01 F7` | error: claims 1 change | `tuning.rs:888-892` |
| Single note, `F0 7E 00 08 02 00 00 00` | error: no `F7` | `tuning.rs:895-898` |

## Rust design

- `CentOffset { semitone: u8, fraction: u16 }` is a plain `Copy` struct (`tuning.rs:33-38`). Its
  range is enforced only by masking in `to_bytes`, not at construction.
- `UserScale(pub [CentOffset; 128])` and `UserOctave(pub [CentOffset; 12])` are public-field
  newtypes (`tuning.rs:86`, `151`).
- `from_bytes(&[u8])` reads the first N·3 bytes and **accepts longer input**; `to_bytes` returns a
  `Vec<u8>`.
- The Korg wrappers delegate to `frame::build_sysex` / `build_sysex_request` / `parse_sysex`
  (`tuning.rs:212-271`). That choice is what forces NOTE-1 encoding and the missing `ss`. It is an
  incidental reuse, not a protocol decision.
- The MTS functions hand-build `Vec`s and take a `&str` name (`tuning.rs:285-468`). The single-note
  parser returns `Vec<(u8, CentOffset)>`.
- No caller outside `transaction.rs` uses these functions. MTS has no transaction wrapper.
- Essential: the 3-byte note encoding, the table sizes and the MTS layouts. Incidental: `Vec`
  returns, `f32` helpers and `&str` names.

## Zig / TigerStyle notes

- Types (sketch):

```zig
// sketch
pub const Pitch = struct { semitone: u7, fraction: u14 };          // TABLE 3 entry
pub const OctavePitch = struct { offset: i8, fraction: u14 };     // TABLE 4 entry, offset -12..23
pub const UserScale = [128]Pitch;
pub const UserOctave = [12]OctavePitch;
pub const user_scale_size = 128 * 3;   // 384
pub const user_octave_size = 12 * 3;   // 36
```

- Encoders take `*[384]u8` / `*[36]u8` and return nothing. Decoders take `*const [N]u8` and return
  `error{ DataByteInvalid, OctaveOffsetInvalid }!T`. Require the exact length; do not accept longer
  slices.
- **No floats in the core.** Convert in integer milli-cents with round half up:
  - `fraction = min(16383, (millicents · 16384 + 50_000) / 100_000)`. Use `u64` intermediates and
    `millicents < 100_000`.
  - `millicents = (fraction · 100_000 + 8_192) / 16_384`.
  - Results can differ from Rust's `f32` path by at most 1 LSB at exact halves. That is acceptable,
    because the wire format carries `u14`.
- MTS checksum (sketch): `var x: u8 = 0; for (msg[1 .. msg.len - 2]) |b| x ^= b; return x & 0x7F;`.
  Assert `msg[1] == 0x7E` before calling.
- Fixed wire sizes (comptime constants):

  | Message | Length |
  |---------|-------:|
  | Korg scale request (spec) | 9 |
  | Korg octave request (spec) | 9 |
  | `0x44` (spec raw) | 393 |
  | `0x44` (Rust-compat encoded) | 447 |
  | `0x45` (spec raw) | 45 |
  | `0x45` (Rust-compat encoded) | 50 |
  | MTS bulk | 408 |
  | MTS single note | `8 + 4·n`, max 516 |

- The single-note builder takes `changes: []const NoteChange` and **asserts**
  `changes.len <= 127`. Silent truncation is a caller bug.
- Take the name as `[16]u8`, so the caller pads it. Assert each byte is `<= 0x7F`.
- Model `ss` as `enum(u8) { current = 0x7F, _ }` plus a `u3` slot index 0–5. Assert `slot <= 5`.
- Pair every encode with a decode assertion in tests. Fuzz the decoders with random bytes and
  assert they return either a value or an error from the error set, never a crash.

## Open questions

- [hw] Does the device require `ss` in `0x14`/`0x15`? Is the `0x44`/`0x45` payload raw or NOTE-1
  encoded? Capture a SCALE DUMP from the panel to settle both.
- [hw] Is the TABLE 4 offset relative to the octave's C or to the key itself? Compare an
  equal-temperament user octave dump: `00, 01, 02 …` versus all zeros.
- [hw] Does the device verify the MTS checksum? Does a Rust-built dump (checksum off by `0x7E`)
  get rejected?
- [hw] For a user octave, MTS keys 60–71 carry absolute `xx`. How does the device convert those
  to TABLE 4 offsets?
- [hw] Does the device honor MMA "no change" (`7F 7F 7F`) in single note change?
- [spec] Is `ss` 0–5 the same numbering as USER SCALE 1–6?

## Sources

- Korg MIDI Impl rev 1.01 §1-4 (*4 R/D), §2-7 BULK TUNING DUMP, §2-8 SINGLE NOTE TUNING CHANGE,
  §Exclusive format (7), (8), (9-1), (9-2), (10-1), (10-2), NOTE 3, NOTE 4, TABLE 3, TABLE 4,
  *3-6, *note P21.
- `crates/minilogue-xd/src/sysex/tuning.rs:1-18` — module doc; `33-75` — `CentOffset`;
  `86-140` — `UserScale`; `151-205` — `UserOctave`; `212-271` — Korg wrappers; `285-384` — MTS
  bulk; `391-468` — MTS single note; `470-930` — tests.
- `crates/minilogue-xd/src/sysex/transaction.rs:299-339` — tuning transactions.
- `crates/minilogue-xd/src/param/enums.rs:551-593` — `MicroTuning`.
- `docs/korg-user-guide/book.md:1442-1478` — microtuning selection and Scale Key;
  `2009-2055` — USER SCALE/OCTAVE edit pages, SysEx reception, SCALE DUMP; `2093-2097` —
  microtuning overview.
