---
title: Global parameter blob (TABLE 1, "GLOB")
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/global.rs
  - crates/minilogue-xd/src/sysex/enums.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/tests/sysex_flows.rs
  - docs/dev/0003-phase-3-implementation-plan-sysex-layer.md
  - README.md
related:
  - protocol/sysex-framing.md
  - protocol/korg-7bit-codec.md
  - protocol/errata.md
  - protocol/poly-chain.md
  - protocol/program-blob.md
  - protocol/usb-ports.md
  - design/sysex-transactions.md
  - design/byte-fidelity.md
  - design/errors.md
  - design/typed-values.md
  - zig/memory-budget.md
  - sources/korg-midi-implementation.md
updated: 2026-10-11
---

# Global parameter blob (TABLE 1, "GLOB")

> The global blob holds the XD's system-wide settings: master tune, transpose, MIDI channel,
> routing and filters, sync, clock source, favorites, poly chain, and display options. It is
> **63 bytes**, starts with the ASCII magic `GLOB`, and travels inside a Korg SysEx frame with
> function `0x51` (dump). Function `0x0E` requests it. On the wire the blob is 7-bit encoded to
> 72 bytes, so a full dump message is **80 bytes**. The Rust crate models every byte as a
> typed field. It does not keep the raw bytes, so a round trip normalises booleans and loses
> any value it does not model. Three fields disagree with the Korg spec: Master Tune and
> Transpose (signed or offset?), Oscilloscope (the Rust boolean is inverted relative to the
> spec), and Favorites (0–499 stored in one byte). The dump-size diagram also disagrees with
> TABLE 1 (32 bytes vs 63).

## Messages

Framing (`F0 42 3g 00 01 51 <func> … F7`, where `g` is the global MIDI channel 0–15) is defined
on [SysEx framing](sysex-framing.md). The 7-bit packing is defined on
[Korg 7-bit codec](korg-7bit-codec.md). ACK/NAK status codes are listed on
[SysEx framing](sysex-framing.md).

| Message | Func | Direction | Wire bytes | Payload | Length |
|---|---|---|---|---|---|
| Global data dump request | `0x0E` | host → XD | `F0 42 3g 00 01 51 0E F7` | none | 8 |
| Global data dump | `0x51` | both | `F0 42 3g 00 01 51 51 <72 bytes> F7` | 63-byte blob, 7-bit encoded | 80 |

Device behavior **[spec]** (Korg MIDI Impl rev 1.01 §3(3), §3(6), §1-4, §2-5):

| Event | Device response |
|---|---|
| Receives `0x0E` | Sends a `0x51` dump of the stored globals, or `0x24` (DATA LOAD ERROR) when busy. |
| Receives `0x51` | Saves the blob to internal (persistent) memory, then replies `0x23` (ACK) or `0x24` (NAK). |
| User runs "All Dump" from the MIDI dump page | Sends `0x51` unsolicited, along with the other dumps. |

The channel nibble `g` must match the XD's global MIDI channel. Community captures show the XD
ignores a request sent on any other channel (timeout, no NAK) **[community]**. See
[user modules](user-modules.md#errata--gotchas) and [USB ports](usb-ports.md).

## Layout (63 bytes)

Offsets are decimal; every field is one byte. "Spec range" is the TABLE 1 *Range* column.
"Rust parse" is what `GlobalParams::from_bytes` accepts. "Default" is `GlobalParams::default()`
**[rust]**. That is the crate's idea of factory settings, not a [hw] capture.

| Off | Field (Rust name) | Spec range and meaning | Rust parse | Default |
|---:|---|---|---|---:|
| 0–3 | magic | ASCII `GLOB` = `47 4C 4F 42` | must equal `GLOB`, else `InvalidMagic` | `GLOB` |
| 4 | `master_tune` | −50~50 = −50…+50 cent | raw `u8`, not validated; Rust documents 0–100, centre 50 | 50 |
| 5 | `transpose` | −12~12 = −12…+12 note | raw `u8`, not validated; Rust documents 0–24, centre 12 | 12 |
| 6 | `metronome` | 0,1 = Off, On | bool, `!= 0` | 0 |
| 7 | `damper_polarity` | 0,1 = −, + | `DamperPolarity` | 0 |
| 8 | `local_sw` | 0,1 = Off, On | bool, `!= 0` | 1 |
| 9 | `velocity_curve` | 0~8, *note G1 | `VelocityCurve` | 3 (Type 4) |
| 10 | `knob_mode` | 0~2, *note G2 | `KnobMode` | 0 (Jump) |
| 11 | `sync_in_unit` | 0,1 = 16th note, 8th note | `SyncUnit` | 0 |
| 12 | `sync_out_unit` | 0,1 = 16th note, 8th note | `SyncUnit` | 0 |
| 13 | `sync_in_polarity` | 0,1 = Rise, Fall | `SyncPolarity` | 0 |
| 14 | `sync_out_polarity` | 0,1 = Rise, Fall | `SyncPolarity` | 0 |
| 15 | `midi_route` | 0,1 = USB+MIDI, USB | `MidiRoute` | 0 |
| 16 | `midi_channel` | 0~15 = channel 1~16 | `U4`; 16–255 → `OutOfRange` | 0 |
| 17 | `clock_source` | 0~2, *note G3 | `ClockSource` | 0 (Auto USB) |
| 18 | `en_rx_transport` | 0~1 = Off, On | bool, `!= 0` | 1 |
| 19 | `midi_rx_prog_chg` | 0,1 = Off, On | bool, `!= 0` | 1 |
| 20 | `midi_rx_cc` | 0,1 = Off, On | bool, `!= 0` | 1 |
| 21 | `midi_rx_pitch_bend` | 0,1 = Off, On | bool, `!= 0` | 1 |
| 22 | `midi_tx_prog_chg` | 0,1 = Off, On | bool, `!= 0` | 1 |
| 23 | `midi_tx_cc` | 0,1 = Off, On | bool, `!= 0` | 1 |
| 24 | `midi_tx_pitch_bend` | 0,1 = Off, On | bool, `!= 0` | 1 |
| 25 | `parameter_disp` | 0~1 = Normal, All | `ParameterDisp` | 0 |
| 26 | `brightness` | 0~9 = 1~10 | raw `u8`, not validated | 5 |
| 27 | `auto_power_off` | 0,1 = Off, On | bool, `!= 0` | 0 |
| 28 + 2i | `favorites[i].lower`, i = 0…15 | 0~499 = Program 1~500 ("Favorite i+1 Lower") | raw `u8` | 0 |
| 29 + 2i | `favorites[i].upper`, i = 0…15 | 0~499 = Program 1~500 ("Favorite i+1 Upper") | raw `u8` | 0 |
| 60 | `poly_chain` | 0~2 = Off, Master, Slave | `PolyChainMode` | 0 |
| 61 | `oscilloscope` | 1~2: 1 = Disable, 2 (or 0) = Enable | bool, `!= 0` (**inverted vs spec**, see errata) | 1 |
| 62 | `shift_function` | 0,1 = Favorite, Active Step | `ShiftFunction` | 0 |

The favorites occupy bytes 28–59: 16 entries × 2 bytes. The last entry is at 58/59. Poly chain
behavior itself is described on [poly chain](poly-chain.md).

### Enum value tables

All global enums map one byte to one variant (no TX/RX bands, unlike the CC
[stepped params](../design/stepped-params.md)). Any other byte is an error. The display string
is what the Rust `Display` impl prints **[rust]**. The values match TABLE 1 and its notes
G1–G3 **[spec]**.

| Enum | Offset(s) | Value → variant ("display") |
|---|---|---|
| `DamperPolarity` | 7 | 0 `Normal` ("-"), 1 `Reversed` ("+") |
| `VelocityCurve` | 9 | 0 `Type1` … 7 `Type8` ("Type1"…"Type8"), 8 `Const127` ("Const127") |
| `KnobMode` | 10 | 0 `Jump`, 1 `Catch`, 2 `Scale` |
| `SyncUnit` | 11, 12 | 0 `Sixteenth` ("1/16"), 1 `Eighth` ("1/8") |
| `SyncPolarity` | 13, 14 | 0 `Rise`, 1 `Fall` |
| `MidiRoute` | 15 | 0 `UsbAndMidi` ("USB+MIDI"), 1 `UsbOnly` ("USB Only") |
| `ClockSource` | 17 | 0 `AutoUsb` ("Auto (USB)"), 1 `AutoMidi` ("Auto (MIDI)"), 2 `Internal` |
| `ParameterDisp` | 25 | 0 `Normal`, 1 `All` |
| `PolyChainMode` | 60 | 0 `Off`, 1 `Master`, 2 `Slave` |
| `ShiftFunction` | 62 | 0 `Favorite`, 1 `ActiveStep` ("Active Step") |

When the display string is omitted, it is the variant name. The error for an unknown byte is
`OutOfRange { type_name, value, min: 0, max }`, where `max` is the largest defined value. See
[errors](../design/errors.md).

### FavoriteEntry

`FavoriteEntry { lower: u8, upper: u8 }` is two raw bytes at `28 + 2i` and `29 + 2i`
**[rust]**. The crate does not interpret them. Read the errata before assigning meaning.

### Parse algorithm (`GlobalParams::from_bytes`)

```text
parse(data):
  if len(data) < 63:            error PayloadTooShort{expected 63, actual len}
  if data[0..4] != "GLOB":      error InvalidMagic
  for each offset 4..62 in ascending order:
    raw u8 fields  (4, 5, 26, 28..59): copy
    bool fields    (6, 8, 18..24, 27, 61): value = (byte != 0)
    enum fields    (7, 9..15, 17, 25, 60, 62): lookup, error OutOfRange on miss
    channel        (16): byte <= 15 else OutOfRange
  ignore data[63..]                           # trailing bytes are not an error
```

The first failing field wins. Enum checks run in offset order, so a blob with two bad fields
reports the lower offset.

### Serialize algorithm (`GlobalParams::to_bytes`)

Start from 63 zero bytes, write `GLOB`, then write each field at its offset. Booleans become
exactly `0` or `1`, and enums become their value byte. No byte is left unwritten. The blob is
fully modelled, so the zero fill is never visible.

### Round-trip fidelity

The crate keeps no raw copy of the blob. See [byte fidelity](../design/byte-fidelity.md).
`to_bytes(from_bytes(b)) == b` holds only when:

- every boolean byte is `0` or `1` (any other nonzero byte comes back as `1`), and
- `data.len() == 63` (trailing bytes are dropped).

The oscilloscope byte is the case that matters in practice. The spec's "Enable" value `2`
parses as `true` and is written back as `1`, which the spec calls "Disable".

## Invariants

- `global_blob_size == 63`. The magic occupies bytes 0..4 and the last field is at offset 62.
- `korg_7bit_encoded_len(63) == 72`, so a dump frame is `7 + 72 + 1 == 80` bytes.
- The request frame is exactly 8 bytes, and byte 6 is `0x0E`.
- In a dump frame, byte 6 is `0x51` and the decoded payload has at least 63 bytes.
- After serialize: `blob[0..4] == "GLOB"`, every boolean byte ∈ {0, 1}, and every enum byte
  ≤ its enum's maximum (`velocity_curve` ≤ 8, `knob_mode` ≤ 2, `clock_source` ≤ 2,
  `poly_chain` ≤ 2, all two-valued enums ≤ 1), and `midi_channel` ≤ 15.
- Pair parse and serialize: `parse(serialize(x)) == x` for every valid `x`. The Rust tests
  assert this for defaults, all-maximum values and favorites.
- Favorites: `16 * 2 == 60 - 28` (comptime check that the favorites region ends where
  `poly_chain` begins).

## Errata & gotchas

Record these on [errata](errata.md). No [hw] capture of a global dump exists in the repo: there
are no fixtures, and the only commits touching `global.rs` are scaffold/refactor/lint commits.
Each conflict below is therefore [spec] vs [rust], unresolved.

1. **Dump size: 32 vs 63 bytes.** The §3(6) diagram says "37 Bytes (7bit) → 32 Bytes (8bit)".
   TABLE 1 defines offsets 0–62 (63 bytes) **[spec]**. Rust requires ≥ 63 decoded bytes and
   rejects a 32-byte dump with `PayloadTooShort` **[rust]**. The phase-3 plan said "if a 32-byte
   dump is received, parse what's available and default the rest". That fallback was never
   implemented. The program dump has the same kind of diagram/table mismatch (336 vs 1024; see
   [program blob](program-blob.md)). In both cases the table is the more detailed source. The
   README asserts "Full 63-byte parse/serialize". Verify the real length with a hardware
   capture.
2. **Master Tune and Transpose signedness.** TABLE 1 gives the *Range* column as `-50~50` and
   `-12~12` **[spec]**. Elsewhere, TABLE 2 writes offset-binary fields as stored range plus
   meaning, e.g. "0~100 = −50 Cent~+50 Cent" (Program Tuning) and "0~24 = −12 Note~+12 Note"
   (Scale Key). TABLE 1's different notation suggests the global bytes are **signed int8**
   (two's complement). Rust instead assumes offset-binary (0–100 centre 50, 0–24 centre 12)
   and defaults to 50 and 12 **[rust]**. If the bytes are signed, the Rust default writes
   +50 cents and +12 semitones to the device. Parsing is still byte-exact either way because
   the bytes are not validated. Only the meaning and the defaults are at risk.
3. **Oscilloscope polarity is inverted.** The spec says `1 = Disable, 2 (or 0) = Enable`
   **[spec]**. Rust reads any nonzero byte as "on" and writes `true → 1`, `false → 0`
   **[rust]**. Mapped back to the spec, Rust's `true` (1) means *Disable* and `false` (0) means
   *Enable*. The default `oscilloscope: true` therefore sends "Disable". A device byte of `2`
   (Enable) parses as `true` and is written back as `1` (Disable). Suspected bug.
4. **Favorites 0–499 do not fit in one byte.** Each of bytes 28–59 is listed with range 0~499
   ("Program 1~500") **[spec]**. Rust stores raw `u8`s and documents the conflict **[rust]**.
   The most plausible reading is that "Lower" and "Upper" are the **low and high bytes of one
   little-endian u16 program number per favorite**. The XD has no lower/upper timbres, 500
   programs need 9 bits, and the program-dump request also splits a program number into two
   bytes. Under that reading, favorite *i* = `blob[28+2i] | blob[29+2i] << 8`. The phase-3
   plan considered this reading and left it unverified. Store raw bytes until a capture decides.
5. **Lenient booleans.** Any nonzero byte reads as true. The device presumably writes only 0/1,
   so the risk is limited to the oscilloscope field (item 3).
6. **No range check on bytes 4, 5, 26.** `brightness = 200` parses without error. A strict
   implementation should reject it, or at least preserve it, consistently with the program blob
   policy on [byte fidelity](../design/byte-fidelity.md).
7. **Channel byte.** A global dump stores the XD's own MIDI channel (offset 16). Writing a new
   value changes which `3g` byte the device answers to. After `send_global` with a different
   channel, later requests must use the new `g`.

## Test vectors

From Rust tests (exact):

| Vector | Bytes / expectation | Citation |
|---|---|---|
| Request, channel 0 | `F0 42 30 00 01 51 0E F7` | `global.rs:714-718` |
| Request, channel 15 | byte 2 = `0x3F`, byte 6 = `0x0E` | `global.rs:720-725` |
| Minimal blob (`GLOB` + 59 zeros) | parses: tune 0, transpose 0, all bools false, `Type1`, `Jump`, channel 0, brightness 0, `Off`, `Favorite` | `global.rs:365-370`, `398-414` |
| All-fields blob | bytes 4..27 = `64 18 01 01 01 08 02 01 01 01 01 01 0F 02 01 01 01 01 01 01 01 01 09 01`; favorites `lower = i+1`, `upper = i+100`; 60–62 = `02 01 01` → every field at its max | `global.rs:416-488` |
| Rejected | len 10; magic `PROG`; byte 9 = 99; byte 10 = 5; byte 16 = 16; byte 17 = 5; byte 60 = 10; byte 62 = 5; byte 7 = 3; bytes 11–15 = 5; byte 25 = 5 | `global.rs:490-594` |
| Lenient booleans | bytes 6=2, 8=255, 18=42, 27=100, 61=5 → all `true` | `global.rs:803-817` |
| Trailing bytes | 63-byte blob + ten `FF` → parses, trailing ignored | `global.rs:791-797` |
| Wrong function ID | blob framed with `0x40` → `WrongFunctionId` | `global.rs:759-767` |
| Transaction | queued `0x51` dump → `request_global` returns it; queued `0x23` → `send_global` Ok | `tests/sysex_flows.rs:19-35`, `111-124` |

Derived from the Rust defaults by applying the documented algorithm (not a [hw] capture). Use
it as a golden vector for the Zig serializer and encoder:

```text
default blob (63 bytes):
47 4C 4F 42 32 0C 00 00 01 03 00 00 00 00 00 00
00 00 01 01 01 01 01 01 01 00 05 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 01 00

default dump, channel 0 (80 bytes; every 7-bit group MSB byte is 00):
F0 42 30 00 01 51 51 00 47 4C 4F 42 32 0C 00 00
00 01 03 00 00 00 00 00 00 00 00 00 01 01 01 00
01 01 01 01 00 05 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 01 00 F7
```

A blob with a byte ≥ `0x80` (e.g. a favorite of 255, as in the all-maximum round-trip test,
`global.rs:620-658`) exercises the MSB byte of the codec.

## Rust design

- `GlobalParams` is a plain struct with one typed field per byte. `FavoriteEntry` is a
  `Copy` pair. The ten enums come from a `sysex_value_enum!` macro (`enums.rs:20-100`) that
  generates `from_byte`, `to_byte`, `TryFrom<u8>`, `From<_> for u8` and `Display`, all
  `#[non_exhaustive]`.
- `from_bytes(&[u8]) -> Result<Self>` and `to_bytes(&self) -> Vec<u8>` are pure. The `Vec`
  return is incidental: the size is a constant 63.
- The SysEx helpers are thin wrappers. `build_global_request` calls
  `frame::build_sysex_request(ch, 0x0E)`. `build_global_dump` calls
  `frame::build_sysex(ch, 0x51, blob)`. `parse_global_dump` calls `parse_sysex`, checks the
  function ID, then calls `from_bytes`.
- `SysexTransaction::request_global` / `send_global` (`transaction.rs:265-287`) add I/O:
  - Send the request.
  - Reassemble a complete `F0…F7` frame, skipping realtime bytes.
  - Map a NAK function ID to `NakReceived`.
  - Require `0x51` for request/response, or any status for send-and-wait-ack.
  - The default timeout is 5 s.

  See [SysEx transactions](../design/sysex-transactions.md).
- Essential: the offset table, the enum value tables, the strict enum and channel checks, and
  rejecting short blobs. Incidental: `Vec`, `String` in error messages, and `Display` strings.
  The lenient-bool choice and the missing raw-byte preservation are worth *not* copying (see
  Zig notes).

## Zig / TigerStyle notes

- Sizes: `global_blob_size = 63`, `global_dump_encoded_size = 72`,
  `global_dump_frame_size = 80`, `global_request_frame_size = 8`. Assert them at comptime
  against the codec's `encoded_len` function. See [memory budget](../zig/memory-budget.md).
- Keep the **raw blob as the source of truth** (`[63]u8`) and expose typed accessors. This makes
  round-trips byte-exact (oscilloscope `2`, signed tune) without deciding the open questions
  now. Validation is a separate `validate(blob) GlobalError!void` pass that the parser calls.
  That way a strict mode and a preserving mode share one code path.
- Enums: `enum(u8)` with explicit values. Convert with a `switch` that returns
  `error.ValueOutOfRange`, not `@enumFromInt` (UB-safe only if checked first). Use a
  non-exhaustive `enum(u8) { …, _ }` only for fields where preserving unknown values matters.
- Error set: `error{ PayloadTooShort, InvalidMagic, ValueOutOfRange, WrongFunctionId }` plus
  the framing/codec errors from [SysEx framing](sysex-framing.md).
- Naming: `master_tune_raw`, `transpose_raw`, `favorite_program_number(i)` (u16 helper under
  the little-endian reading, clearly marked unverified), `midi_channel` as `u4`.
- Paired assertions: assert the magic and size before encoding and again after decoding.
  Assert `parse(serialize(x))` in tests over all enum values.

```zig
// sketch
pub const global_blob_size = 63;
pub const GlobalBlob = struct {
    raw: [global_blob_size]u8,

    pub fn parse(data: []const u8) GlobalError!GlobalBlob {
        if (data.len < global_blob_size) return error.PayloadTooShort;
        if (!std.mem.eql(u8, data[0..4], "GLOB")) return error.InvalidMagic;
        var blob: GlobalBlob = .{ .raw = data[0..global_blob_size].* };
        try blob.validate();
        return blob;
    }
    pub fn velocity_curve(blob: *const GlobalBlob) VelocityCurve {
        return @enumFromInt(blob.raw[9]); // validated in parse
    }
};
```

## Open questions

- What is the real decoded dump length: 63 or 32? Capture `0x0E` → `0x51` on hardware.
- Are bytes 4 and 5 signed int8 or offset-binary? Capture at Master Tune −1 cent and Transpose −1.
- How is a favorite encoded? Capture with favorite 1 set to program 300 (`0x12C`). If bytes
  28/29 read `2C 01`, the little-endian u16 reading is right.
- Which oscilloscope byte values does the device send for on and off?
- Does the XD ACK a `0x51` whose bytes are out of range, or NAK `0x26` (DATA FORMAT ERROR)?

## Sources

- `crates/minilogue-xd/src/sysex/global.rs:1-44` — module doc layout table and favorites note.
- `crates/minilogue-xd/src/sysex/global.rs:51-131` — `FavoriteEntry`, `GlobalParams` fields.
- `crates/minilogue-xd/src/sysex/global.rs:133-232` — constants and `from_bytes`.
- `crates/minilogue-xd/src/sysex/global.rs:234-275` — `to_bytes`.
- `crates/minilogue-xd/src/sysex/global.rs:278-312` — `Default`.
- `crates/minilogue-xd/src/sysex/global.rs:318-355` — request/dump builders and parser.
- `crates/minilogue-xd/src/sysex/global.rs:357-832` — tests.
- `crates/minilogue-xd/src/sysex/enums.rs:14-100` — `sysex_value_enum!` macro.
- `crates/minilogue-xd/src/sysex/enums.rs:103-225` — the ten global enums.
- `crates/minilogue-xd/src/sysex/frame.rs:21-68` — function IDs `0x0E` and `0x51`.
- `crates/minilogue-xd/src/sysex/frame.rs:209-249` — `build_sysex`, `build_sysex_request`.
- `crates/minilogue-xd/src/sysex/transaction.rs:144-206`, `265-287` — request/response and
  send-and-ack flows.
- `crates/minilogue-xd/tests/sysex_flows.rs:19-35`, `83-124` — transaction tests.
- `docs/dev/0003-phase-3-implementation-plan-sysex-layer.md:99-118` — M3.2 plan (32 vs 63,
  favorites-as-u16).
- `README.md:150` — "Full 63-byte parse/serialize (TABLE 1)".
- Korg MIDI Impl rev 1.01 §1-4, §2-5 (function lists), §3(3) (request), §3(6) (dump),
  TABLE 1 with *note G1–G3, TABLE 2 (offset-binary notation, offsets 123–124), NOTE 2.
- Community capture of channel-matched replies:
  [irgendwatt/logue-manager](https://github.com/irgendwatt/logue-manager)
  `doc/analyze/probe_mnlgxd_output.txt`.
