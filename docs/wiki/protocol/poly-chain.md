---
title: Poly chain note on/off SysEx
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/poly_chain.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/global.rs
  - docs/korg-user-guide/book.md
related:
  - protocol/sysex-framing.md
  - protocol/global-blob.md
  - protocol/usb-ports.md
  - protocol/errata.md
  - design/typed-values.md
  - design/errors.md
updated: 2026-11-11
---

# Poly chain note on/off SysEx

> Two minilogue xd units can be chained for 8-voice polyphony. Set one unit's global setting
> **Poly Chain** to Master and the other's to Slave, then connect the Master's MIDI OUT to the
> Slave's MIDI IN. The Master forwards voices 5–8 to the Slave as two Korg exclusive messages:
> - function `0x60` **NOTE ON**: voice slot, note, velocity and a 3-byte pitch.
> - function `0x61` **NOTE OFF**: voice slot and a mute flag.
>
> The payload is **raw 7-bit bytes, not NOTE-1 encoded**. The Korg spec gives the pitch's high
> byte only 2 bits, which makes the pitch a 16-bit value. The Rust crate treats it as 21 bits.
> What the pitch number means has not been documented or measured.

## Behavior

| Global setting | Effect | Provenance |
|----------------|--------|------------|
| Off | No poly-chain messages sent or acted on | [spec] user guide |
| Master | Transmits `0x60`/`0x61` "when required" (transmit condition **P**). Plays voices 1–4 locally; voices 5–8 sound on the Slave | [spec] §1-4 *4, (22), (23); user guide |
| Slave | On receiving `0x60`/`0x61`, generates note-on/off events. The panel is inoperable except global settings and the MASTER knob; "tone parameters are determined by the Master unit" | [spec] (22), (23); user guide |

- The setting is stored at global-blob offset 60 (`0–2` = Off/Master/Slave) [rust]
  (`crates/minilogue-xd/src/sysex/global.rs:37`, `198`). See [global blob](global-blob.md).
- Physical link: Master **MIDI OUT** → Slave **MIDI IN** (`docs/korg-user-guide/book.md:2158-2165`).
  Nothing documents whether poly-chain messages also appear on USB ports (see
  [USB ports](usb-ports.md)).
- How the Master conveys "tone parameters" to the Slave (program dumps? CCs?) is not documented.

## Layout — NOTE ON (`0x60`)

| Offset | Spec bits | Value | Rust field / check |
|-------:|-----------|-------|--------------------|
| 0–5 | — | `F0 42 3g 00 01 51` | `validate_header`. `g` is **not returned** by the parser |
| 6 | — | `0x60` | must equal `0x60`, else `WrongFunctionId` |
| 7 | `0000 00ss` | voice slot 0–3 | `U2::new` → `OutOfRange` if `> 3` |
| 8 | `0nnn nnnn` | note number 0–127 | `U7::new` → `OutOfRange` if `> 127` |
| 9 | `0vvv vvvv` | velocity 0–127 | `U7::new` |
| 10 | spec `0000 00hh`; Rust `0hhh hhhh` | pitch H | masked `& 0x7F`, never rejected |
| 11 | `0mmm mmmm` | pitch M | masked `& 0x7F` |
| 12 | `0lll llll` | pitch L | masked `& 0x7F` |
| 13 | — | `0xF7` | the *last* byte must be `F7` |

Length: 14 bytes.

## Layout — NOTE OFF (`0x61`)

| Offset | Spec bits | Value | Rust field / check |
|-------:|-----------|-------|--------------------|
| 0–5 | — | `F0 42 3g 00 01 51` | `validate_header` |
| 6 | — | `0x61` | else `WrongFunctionId` |
| 7 | `0000 00ss` | voice slot 0–3 | `U2::new` |
| 8 | `0000 000m` | mute 0/1 | Rust: `mute = byte != 0`. Builds `0`/`1` |
| 9 | — | `0xF7` | last byte |

Length: 10 bytes. The Rust doc describes mute as "mute the voice immediately", presumably skipping
the release stage. This meaning is unverified.

## Pitch encoding

The pitch is split MSB-first into 7-bit groups:

```text
H = (pitch >> 14) & mask_h      spec: mask_h = 0x03  (pitch is u16, max 0xFFFF)
M = (pitch >>  7) & 0x7F        Rust: mask_h = 0x7F  (pitch is u21, max 0x1FFFFF)
L =  pitch        & 0x7F
pitch = (H << 14) | (M << 7) | L
```

- For `pitch <= 0xFFFF`, the spec and Rust encodings produce **identical bytes**. The two differ
  only when Rust emits `H >= 4`, which the spec does not allow.
- `build_note_on` returns `Error::OutOfRange { type_name: "pitch", max: 0x1FFFFF }` when
  `pitch > 0x1FFFFF` (`poly_chain.rs:145-157`).
- Semantics are **unknown**:
  - Nothing in the spec or code says how `pitch` relates to `note`. It could be absolute pitch,
    pitch-bend-adjusted pitch, or a micro-tuning offset.
  - The Rust doc claims "sub-semitone precision for the assigned voice" (`poly_chain.rs:98-101`).
    This is an assumption, not a measurement.
  - A port must treat pitch as an opaque `u16` that is passed through.

## Parse algorithm (Rust)

`parse_note_on` (`poly_chain.rs:207-248`); `parse_note_off` is the same with length 10, function
`0x61` and the mute byte (`poly_chain.rs:258-287`):

1. `len >= 14`, else `PayloadTooShort { expected: 14, actual }`.
2. `validate_header` (`poly_chain.rs:294-325`): `[0] == F0`, `[1] == 42`,
   `[2] & F0 == 30`, `[3..6] == 00 01 51`. Any failure gives `InvalidHeader`.
3. `[6] == 0x60`, else `WrongFunctionId { expected: 0x60, found }`.
4. `[len-1] == F7`, else `InvalidHeader`.
5. Slot via `U2`, note and velocity via `U7`, then H/M/L masked and combined.

Longer frames pass as long as the last byte is `F7`. Payload bytes are then read from fixed offsets
7–12 and anything in between is ignored.

## Invariants

- Note-on frame: `len == 14`, `[6] == 0x60`, `[7] <= 3`, `[8] <= 127`, `[9] <= 127`,
  `[10] <= 3` (spec), `[11] <= 127`, `[12] <= 127`.
- Note-off frame: `len == 10`, `[6] == 0x61`, `[7] <= 3`, `[8] <= 1`.
- `decode(encode(pitch)) == pitch` for every `pitch <= 0xFFFF`.
- `encode(pitch)` yields `H < 4`.
- The poly-chain payload never passes through the 7-bit codec. Assert that the frame length is
  exactly 14/10 and not the encoded length.

## Errata & gotchas

| # | Issue | Provenance | Resolution |
|---|-------|------------|------------|
| 1 | The pitch is 16-bit (H = `0000 00hh`) per the spec, 21-bit in Rust (`PITCH_MAX = 0x1F_FFFF`, `poly_chain.rs:96`) | [spec] vs [rust] | Use `u16` and reject (or assert) `H > 3`. Wire-compatible with Rust for valid values. |
| 2 | The spec's description column labels `0x60`/`0x61` as "USER SLOT DATA" | [spec] copy-paste | Ignore. The names are POLY CHAIN NOTE ON/OFF (§1-4, §2-5). |
| 3 | The parser accepts a non-zero mute byte `> 1` as `true` | [rust] lenient | Strict port: reject `> 1`; lenient port: `!= 0`. |
| 4 | The parser discards `g` (channel) | [rust] | Return it; a Slave may want to filter by the Master's global channel. |
| 5 | `parse_sysex` must **not** be used here: it would 7-bit-*decode* the raw payload | [rust] design | Use the per-message parser (see [SysEx framing](sysex-framing.md)). |

## Test vectors

| Case | Bytes | Citation |
|------|-------|----------|
| Note on: ch 0, slot 0, note 60, vel 100, pitch 0 (14 bytes) | `F0 42 30 00 01 51 60 00 3C 64 00 00 00 F7` | `poly_chain.rs:391-401`, `639-650` |
| Note on: ch 0, slot 1, note 69, vel 64, pitch `0b1010101_1001100_0110011` = `0x156633` | `F0 42 30 00 01 51 60 01 45 40 55 4C 33 F7`. H = `0x55` is **invalid per spec** | `poly_chain.rs:418-438` |
| Note on: ch 15, slot 3, note 127, vel 127, pitch `0x1FFFFF` | `F0 42 3F 00 01 51 60 03 7F 7F 7F 7F 7F F7` (spec-invalid H) | `poly_chain.rs:405-415`, `653-664` |
| Note on: pitch `0x200000` | build error `OutOfRange` | `poly_chain.rs:667-675` |
| Note on: channel byte for every `c` | `[2] == 0x30 \| c` | `poly_chain.rs:456-470` |
| Note off: ch 0, slot 2, mute false (10 bytes) | `F0 42 30 00 01 51 61 02 00 F7` | `poly_chain.rs:474-483` |
| Note off: ch 7, slot 1, mute true | `F0 42 37 00 01 51 61 01 01 F7` | `poly_chain.rs:486-494` |
| Too short | `F0 42 30 00 01 51 60 F7` → error; `… 61 F7` → error | `poly_chain.rs:515-518`, `582-585` |
| Slot 5 | `F0 42 30 00 01 51 60 05 3C 64 00 00 00 F7` → error | `poly_chain.rs:561-579` |
| Wrong fid / `[1] = 43` / missing `F7` | error | `poly_chain.rs:521-557`, `588-608` |

Spec-maximum vector for a port (derived): pitch `0xFFFF`, which encodes as `H M L = 03 7F 7F`.

## Rust design

- `U2` is a range-checked newtype local to this module (`poly_chain.rs:40-69`). See
  [typed values](../design/typed-values.md).
- `PolyChainNoteOn { voice_slot: U2, note: U7, velocity: U7, pitch: u32 }` and
  `PolyChainNoteOff { voice_slot: U2, mute: bool }` are `Copy` value types
  (`poly_chain.rs:103-127`).
- `build_note_on` returns `Result<Vec<u8>>`; the only possible error is the pitch range.
  `build_note_off` returns `Vec<u8>` (`poly_chain.rs:145-193`).
- No caller uses these functions. The crate does not implement Master or Slave behavior; it only
  provides the codec. The constants live in `frame.rs:75-77`.

## Zig / TigerStyle notes

- Use fixed arrays: `fn note_on_write(target: *[14]u8, channel: u4, note_on: NoteOn) void` and
  `fn note_off_write(target: *[10]u8, …) void`. No error is possible when the fields are typed:
  `slot: u2`, `note: u7`, `velocity: u7`, `pitch: u16`.
- Assert `target[10] <= 3` after writing. In tests, assert that parse after build returns the
  same struct.
- Parse error set: `error{ FrameLengthInvalid, HeaderInvalid, FunctionIdMismatch,
  EndByteMissing, SlotInvalid, WireByteInvalid, PitchHighInvalid }`.
- Require `len == 14` or `len == 10` exactly. This is stricter than Rust.
- A Slave implementation must keep voice state in a fixed `[4]VoiceState` indexed by `slot`. Note
  off for an idle slot is a no-op, not an error.

## Open questions

- What does `pitch` encode, and in what units? Capture `0x60` from a Master while playing:
  - a single key with no bend;
  - the same key with a full pitch bend;
  - a non-equal-temperament micro-tuning.
- Does the Master send anything else (program data, CCs) to keep the Slave's sound in sync?
- Does the Slave filter by the header's `g`?
- Are the messages also sent over USB, or only over 5-pin MIDI OUT?
- What does mute = 1 mean exactly: immediate cut, or release skipped?

## Sources

- Korg MIDI Impl rev 1.01 §1-4 (*4 condition P), §2-5, §Exclusive format (22) POLY CHAIN NOTE ON,
  (23) POLY CHAIN NOTE OFF.
- `docs/korg-user-guide/book.md:1925-1934` — Poly Chain Off/Master/Slave; `2158-2165` — wiring
  and the voices 5–8 split; `344`.
- `crates/minilogue-xd/src/sysex/poly_chain.rs:1-25` — module doc; `40-69` — `U2`; `96-127` —
  types; `145-193` — builders; `207-325` — parsers; `391-675` — tests.
- `crates/minilogue-xd/src/sysex/frame.rs:75-77` — function IDs.
- `crates/minilogue-xd/src/sysex/global.rs:37`, `198` — Poly Chain Mode at global offset 60.
