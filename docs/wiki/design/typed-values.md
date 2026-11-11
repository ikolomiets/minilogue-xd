---
title: Range-checked value types (U2, U4, U7, U14, I14 and friends)
category: design
sources:
  - crates/minilogue-xd/src/message/types.rs
  - crates/minilogue-xd/src/sysex/poly_chain.rs
  - crates/minilogue-xd/src/message/channel.rs
  - crates/minilogue-xd/src/param/encoding.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/src/error.rs
related:
  - design/errors.md
  - protocol/midi-messages.md
  - protocol/parameter-encodings.md
  - protocol/poly-chain.md
  - design/stepped-params.md
  - design/realtime-controller.md
  - zig/tigerstyle-mapping.md
updated: 2026-10-11
---

# Range-checked value types (U2, U4, U7, U14, I14 and friends)

> The crate uses five small integer newtypes for MIDI fields: `U2` for the poly-chain voice
> slot, `U4` for the channel, `U7` for data bytes, `U14` for Song Position and `I14` for
> signed pitch bend. Each is valid only within its range, so out-of-range values can't exist
> after construction. Every constructor failure becomes `Error::OutOfRange`. In Zig these
> become the native types `u2`, `u4`, `u7`, `u14` and `i14`. Runtime range checks are then
> needed only where untrusted bytes or user numbers enter the system.

## Facts

### Foundation newtypes [rust]

| Type | Inner repr | MIN | MAX | Constructor input | `type_name` in error | Used for | Source |
|---|---|---|---|---|---|---|---|
| `U2` | `u8` | 0 | 3 | `u8` | `"U2"` | Poly-chain voice slot index | `sysex/poly_chain.rs:34-89` |
| `U4` | `u8` | 0 | 15 | `u8` | `"U4"` | MIDI channel (**0-based**), SysEx `3g` nibble | `message/types.rs:10-61` |
| `U7` | `u8` | 0 | 127 | `u8` | `"U7"` | Note, velocity, CC number/value, program, pressure, bank MSB/LSB | `message/types.rs:63-114` |
| `U14` | `u16` | 0 | 16383 | `u16` | `"U14"` | Song Position Pointer (MIDI beats) | `message/types.rs:116-167` |
| `I14` | `i16` | -8192 | 8191 | `i16` | `"I14"` | Pitch bend, centred at 0 | `message/types.rs:169-220` |

All five have the same API surface:

| Item | Behaviour |
|---|---|
| `pub const MIN`, `pub const MAX` | Inclusive bounds, typed as the inner primitive |
| `fn new(v) -> Result<Self>` | `v` outside `MIN..=MAX` → `Error::OutOfRange { type_name, value: v as i64, min: MIN as i64, max: MAX as i64 }` |
| `fn value(self) -> inner` | Returns the stored value. It never fails. |
| `impl TryFrom<inner>` | Calls `new`. |
| `impl From<T> for inner` | Widening back to the primitive. It never fails. |
| `impl Display` | Prints the decimal inner value, e.g. `U4(7)` → `"7"` and `I14(-42)` → `"-42"`. |
| derives | `Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord`. There is **no `Default`**. |

The tuple field is private, so `new`/`try_from` is the only way to build a value. **No
unchecked constructor** exists. Code that builds values from masked bits still calls the
checked `new`, for example `U4::new(status & 0x0F)?` (`message/channel.rs:67`).

The error displays as `"{type_name} value {value} out of range {min}..={max}"`, for example
`U7 value 128 out of range 0..=127` (`error.rs:94-101`, test at `error.rs:137-148`). See
[errors](errors.md).

### Wire mapping of the 14-bit types [spec]

| Type | Wire value `w` (0..16383) | LSB byte | MSB byte | Message |
|---|---|---|---|---|
| `U14` | `w = value` | `w & 0x7F` | `w >> 7` | `F2 lsb msb` (SPP) |
| `I14` | `w = value + 8192` (centre `0x2000`) | `w & 0x7F` | `w >> 7` | `En lsb msb` (pitch bend) |

`I14::MIN` (-8192) maps to `00 00`, 0 maps to `00 40`, and `I14::MAX` (8191) maps to `7F 7F`.
See [MIDI messages](../protocol/midi-messages.md) for the full parse/encode rules.

### Other range-checked values in the crate (owned elsewhere)

| Value | Valid range | Failure | Where | Documented in |
|---|---|---|---|---|
| `TenBitParam` | 0..=1023 | `OutOfRange("TenBitParam")` | `param/encoding.rs:28-56` | [parameter encodings](../protocol/parameter-encodings.md) |
| `EightBitHighRes` | 0..=200 | `OutOfRange("EightBitHighRes")` | `param/encoding.rs:116-144` | same |
| `FourteenBitParam` | 0..=16383 | `OutOfRange("FourteenBitParam")` | `param/encoding.rs:203-231` | same |
| `TenBitSysex` | 0..=1023 | `OutOfRange("TenBitSysex")` | `param/encoding.rs:283-311` | same |
| Stepped enums (`from_rx_value`, `from_program_value`) | Per-enum bands | `OutOfRange(stringify!(Enum))`, `min 0 max 127` for RX | `param/enums.rs:63-107` | [stepped params](stepped-params.md) |
| SysEx enums `from_byte` | Per enum | `OutOfRange(stringify!(Enum))` | `sysex/enums.rs:40-52` | [program blob](../protocol/program-blob.md) |
| `BankSelect.lsb` | 0..=4 | `OutOfRange("BankSelect LSB")` | `message/channel.rs:485-505` | [MIDI messages](../protocol/midi-messages.md) |
| `ProgramNumber` | 0..=499 (bank = n/100, slot = n%100) | `SysexError::InvalidProgramNumber(n)`, **not** `OutOfRange` | `sysex/program/mod.rs:30-63` | [program blob](../protocol/program-blob.md) |
| Poly-chain pitch | 0..=`0x1FFFFF` (21 bits) | `OutOfRange("pitch")` | `sysex/poly_chain.rs:96, 144-151` | [poly chain](../protocol/poly-chain.md) |
| Blob `voice_mode_type` / `mod_fx_type` (1-based) | 1..=4 / 1..=5 | `OutOfRange("VoiceModeType" / "ModFxType")` when 0 | `sysex/program/synth.rs:657-686` | [program blob](../protocol/program-blob.md) |
| `f32_to_10bit` | 0.0..=1.0 → `round(v × 1023)` | `OutOfRange("f32 (0.0-1.0)", v×1000, 0, 1000)` | `controller.rs:24-39` | [realtime controller](realtime-controller.md) |
| `f32_to_pitch_bend` | -1.0..=1.0 → `round(v × 8191)` | `OutOfRange("f32 (-1.0-1.0)", v×1000, -1000, 1000)` | `controller.rs:41-57` | same |
| Bend range +/- | 0..=12 semitones | `OutOfRange("bend_range_plus"/"_minus")` | `controller.rs:485-512` | same |
| Program level | -18.0..=6.0 dB → `(db + 18) × 5` | `OutOfRange("program_level_db", db×10, -180, 60)` | `controller.rs:526-538` | same |

## Invariants

These translate directly into Zig asserts or types:

- `0 <= u2 <= 3`, `0 <= u4 <= 15`, `0 <= u7 <= 127`, `0 <= u14 <= 16383`,
  `-8192 <= i14 <= 8191`. In Zig the type enforces these, so no asserts are needed.
- Every byte produced from a `U7` has bit 7 clear: `assert(byte & 0x80 == 0)` after encoding.
- Pitch bend: `wire = @as(u14, @intCast(@as(i16, value) + 8192))` and the reverse are exact
  inverses for all 16384 values.
- SPP / U14: `(msb << 7) | lsb == value` with `lsb, msb <= 0x7F`.
- A status byte's low nibble is always a valid `u4`, so building a channel from it cannot
  fail. The Rust error path there is dead code.
- `ProgramNumber`: `n <= 499`. Also `bank(n) * 100 + slot(n) == n`, `bank <= 4` and
  `slot <= 99`.

## Errata & gotchas

- **Channels are 0-based.** `U4(0)` is MIDI channel 1. `Display` prints the raw 0-based value,
  so `U4(7)` shows as `"7"`, which is channel 8 [rust]. The XD's Global "MIDI Ch" is shown as
  1..16 and stored as 0..15 (TABLE 1 offset 16) [spec].
- **Inconsistent error for wire bytes with bit 7 set** [rust]. Message parsers call
  `U7::new(byte)?`. A data byte `>= 0x80` therefore raises `OutOfRange("U7")`, not
  `InvalidMessage`. Pitch bend and SPP mask with `& 0x7F` instead and never fail
  (`message/channel.rs:195-196`). See [MIDI messages](../protocol/midi-messages.md).
- **Poly-chain voice slot** is parsed with `U2::new(bytes[7])?`. A bad slot byte raises
  `OutOfRange("U2")`, not a `SysexError` (`sysex/poly_chain.rs:233, 283`).
- **The `BankSelect` invariant can be bypassed.** `BankSelect::new` checks `lsb <= 4`, but the
  struct's fields are `pub`, so a struct literal skips the check (`message/channel.rs:475-483`).
  The parser does call `new` (`message/channel.rs:551`).
- **Pitch-bend float mapping is asymmetric** [rust]. `f32_to_pitch_bend(-1.0)` gives -8191,
  never -8192. 0.0 gives 0 and 1.0 gives 8191 (`controller.rs:47-57`).
- **Rounding in the float helpers**: Rust's `f32::round` rounds half away from zero, so
  `0.5 × 1023 = 511.5` becomes 512. A Zig port must use the same rounding (`@round`) to
  reproduce byte-exact output.
- **Lossy error payloads.** The f32 helpers report `value × 1000` truncated to `i64`. NaN fails
  the range check, but its reported value is `0` because Rust's float→int cast saturates NaN
  to 0.
- **ProgramNumber's error type differs** from all the others: it raises a `SysexError`, not
  `OutOfRange`.
- `U2` lives in `sysex/poly_chain.rs`, not with the other newtypes in `message/types.rs`. It is
  not re-exported from the crate root. This is an incidental layout choice.

## Test vectors

From `crates/minilogue-xd/src/message/types.rs:222-390` and
`crates/minilogue-xd/src/sysex/poly_chain.rs:339-384`:

| Type | Accepted | Rejected | Display |
|---|---|---|---|
| `U2` | 0, 1, 2, 3 | 4, 128, 255 | `U2(1)` → `"1"` |
| `U4` | 0, 15 (also `try_from(15)`) | 16 | `U4(7)` → `"7"` |
| `U7` | 0, 127 | 128 | `U7(42)` → `"42"` |
| `U14` | 0, 16383, 8192 | 16384 | `U14(1000)` → `"1000"` |
| `I14` | -8192, 0, 8191 | -8193, 8192 | `I14(-42)` → `"-42"` |

Conversions: `u8::from(U4(10)) == 10`, `u16::from(U14(8192)) == 8192`,
`i16::from(I14(-100)) == -100`.

Error display (`crates/minilogue-xd/src/error.rs:137-148`):
`OutOfRange { "U7", 128, 0, 127 }` → `U7 value 128 out of range 0..=127`.

Derived vectors (not in the Rust tests), useful for a Zig port:

| Input | Output |
|---|---|
| `I14(-8192)` as pitch-bend wire | `lsb 0x00, msb 0x00` |
| `I14(0)` | `lsb 0x00, msb 0x40` |
| `I14(8191)` | `lsb 0x7F, msb 0x7F` |
| `f32_to_10bit(0.5)` | 512 |
| `f32_to_pitch_bend(-1.0)` | -8191 |
| `ProgramNumber(250)` | bank 2, slot 50 |

## Rust design

- These are **essential**:
  - Construction-time validation.
  - One canonical error variant with the type name and bounds.
  - Storing values in fields typed by meaning. `NoteOn { channel: U4, key: U7, velocity: U7 }`
    makes invalid messages impossible to represent.
- These are **incidental**:
  - The four nearly identical hand-written impls in `types.rs`, with no macro or generics.
  - `Display`, `Hash` and `Ord` derives.
  - `TryFrom`/`From` impls.
  - The private-field newtype pattern itself. It exists only because Rust has no `u7`.
- Storage costs nothing extra: each newtype is one `u8`/`u16`/`i16`. All are `Copy` and stored
  inline, with no allocation.
- The crate checks at **every** construction, including internal ones where the value is
  already known to be valid (masked nibbles, encoder outputs). Those checks return `Result`s
  that can never be `Err`. TigerStyle would make them asserts or remove them through types.

## Zig / TigerStyle notes

- Use **native arbitrary-width integers**: `u2`, `u4`, `u7`, `u14` and `i14`. The type system
  then gives the guarantee that Rust gets from newtypes, at no runtime cost. Note that
  `@sizeOf(u7) == 1` and `@sizeOf(u14) == 2`. Arrays of `u7` are not bit-packed unless they
  are fields of a `packed struct`.
- Do **runtime range checks only at trust boundaries**, and return an operating error there:
  - Bytes received from a port, with the data-byte rule `byte & 0x80 == 0`, else
    `error.DataByteInvalid`.
  - Bytes read from files or SysEx payloads.
  - Numbers from the user or API, such as floats, program numbers and bend ranges.
- Inside the library, convert with `@intCast`, which is safety-checked in Debug/ReleaseSafe,
  or with `@truncate` after masking. Assert the precondition first.
- Use `std.math.cast(u7, x)`, which returns `?u7`, for "check and convert" at boundaries.
- Keep **one policy for data bytes with bit 7 set**: reject them everywhere with the same error.
  The Rust code masks in some places and rejects in others.
- Name channels explicitly: `channel: u4` is 0-based. Convert only at UI or log edges
  (`channel_display = @as(u5, channel) + 1`).
- Zig errors carry no payload. If callers need `value`, `min` and `max` for diagnostics, pass
  an optional `*Diagnostics` out-parameter instead of attaching data to the error. See
  [errors](errors.md).
- The float helpers are the only floating point in the foundation. Keep them at the API edge.
  Reject NaN explicitly (`std.math.isNan`), then range-check, then use `@round`. This matches
  Rust's round-half-away-from-zero.

```zig
// sketch — wire boundary helpers
pub fn data_byte(byte: u8) error{DataByteInvalid}!u7 {
    return std.math.cast(u7, byte) orelse error.DataByteInvalid;
}

pub fn pitch_bend_from_wire(lsb: u7, msb: u7) i14 {
    const wire: u14 = (@as(u14, msb) << 7) | lsb;
    return @intCast(@as(i16, wire) - 8192);
}

pub fn pitch_bend_to_wire(value: i14) struct { lsb: u7, msb: u7 } {
    const wire: u14 = @intCast(@as(i16, value) + 8192);
    return .{ .lsb = @truncate(wire), .msb = @truncate(wire >> 7) };
}

pub const ProgramNumber = struct {
    value: u9, // 0..499 — u9 holds 0..511, so assert the tighter bound
    pub fn init(n: u16) error{ProgramNumberInvalid}!ProgramNumber {
        if (n > 499) return error.ProgramNumberInvalid;
        return .{ .value = @intCast(n) };
    }
    pub fn bank(self: ProgramNumber) u3 { return @intCast(self.value / 100); }
    pub fn slot(self: ProgramNumber) u7 { return @intCast(self.value % 100); }
};
```

- For values with ranges that are not powers of two (`ProgramNumber` 0..499, `BankSelect`
  LSB 0..4, `EightBitHighRes` 0..200):
  - Use the next native width, a constructor that validates, and an `assert` in every method
    that relies on the tighter bound.
  - Alternatively, model them as an `enum(u3)` when the set is small (bank 0..4).

## Open questions

- Should `ProgramNumber` failures be `OutOfRange`, to be consistent? Rust uses
  `SysexError::InvalidProgramNumber`. A Zig port can choose one error name per layer. See
  [errors](errors.md).
- Is the pitch-bend float asymmetry (no -8192) intentional? Hardware treats -8192 and -8191 as
  practically the same, but byte-exact tests would differ.

## Sources

- `crates/minilogue-xd/src/message/types.rs:1-220` — `U4`, `U7`, `U14`, `I14`.
- `crates/minilogue-xd/src/message/types.rs:222-390` — tests.
- `crates/minilogue-xd/src/sysex/poly_chain.rs:34-89` — `U2`.
- `crates/minilogue-xd/src/sysex/poly_chain.rs:339-384` — `U2` tests.
- `crates/minilogue-xd/src/sysex/poly_chain.rs:96, 144-151, 233, 283` — pitch bound, slot parse.
- `crates/minilogue-xd/src/message/channel.rs:67, 195-204, 472-553` — newtype use in parsers,
  `BankSelect`.
- `crates/minilogue-xd/src/param/encoding.rs:28-56, 116-144, 203-231, 283-311`
- `crates/minilogue-xd/src/sysex/program/mod.rs:30-63` — `ProgramNumber`.
- `crates/minilogue-xd/src/controller.rs:24-57, 485-538` — float and range helpers.
- `crates/minilogue-xd/src/error.rs:94-101, 137-148` — `OutOfRange` and its display.
- Korg MIDI Impl rev 1.01 §2-1 (pitch bend), §2-2 (SPP) and TABLE 1 offset 16 (MIDI Ch).
