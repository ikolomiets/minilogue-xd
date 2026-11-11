---
title: Error taxonomy (Error, SysexError) and Zig error sets
category: design
sources:
  - crates/minilogue-xd/src/error.rs
  - crates/minilogue-xd/src/codec.rs
  - crates/minilogue-xd/src/message/channel.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/src/sysex/program/file.rs
  - crates/minilogue-xd/src/transport.rs
  - crates/minilogue-xd/src/device.rs
  - crates/minilogue-xd/Cargo.toml
related:
  - design/typed-values.md
  - design/transport.md
  - design/sysex-transactions.md
  - protocol/sysex-framing.md
  - protocol/midi-messages.md
  - protocol/korg-7bit-codec.md
  - zig/tigerstyle-mapping.md
  - zig/module-layout.md
updated: 2026-11-11
---

# Error taxonomy (Error, SysexError) and Zig error sets

> The crate has **one public error enum**, `Error`, with 4–7 variants depending on Cargo
> features. One variant wraps a second enum, `SysexError`, which has 13 variants. Every
> fallible function returns `Result<T> = std::result::Result<T, Error>`. Most payloads are
> `String`s or small integers, and the `Display` text comes from `thiserror`. A Zig port
> replaces this with **per-layer error sets** plus an optional fixed-size `Diagnostics`
> out-parameter. It also turns every error that a correct caller can never trigger into an
> `assert`.

## Facts

### `Error` (`error.rs:90-129`) [rust]

| Variant | Payload | `Display` | Feature gate | Raised by |
|---|---|---|---|---|
| `OutOfRange` | `type_name: &'static str, value: i64, min: i64, max: i64` | `{type_name} value {value} out of range {min}..={max}` | — | Newtype constructors, encodings, stepped enums, controller helpers (≈40 sites). See [typed values](typed-values.md). **Also raised for wire data bytes `>= 0x80`** in message parsers. |
| `Codec` | `String` | `codec error: {0}` | — | `decode_7bit` only, when a wire byte has bit 7 set: `"wire byte at offset {pos} has high bit set (0x{b:02X})"` (`codec.rs:74-81`) |
| `InvalidMessage` | `String` | `invalid MIDI message: {0}` | — | Channel, real-time and common message parsers, plus `parse_midi_bytes` (≈45 sites in `message/*`). A test-only `FailingOutput` uses it to fake a send failure (`sysex/transaction.rs:365-376`). |
| `Sysex` | `#[from] SysexError` | `SysEx error: {0}` | — | Everything in `sysex/*`. `?` converts it automatically. |
| `MidiIo` | `String` | `MIDI I/O error: {0}` | `midi-io` | midir client creation, port not found (`"no MIDI output port matching '…'"`), connect, send, input disconnected (`"MIDI input disconnected"`), from `transport.rs`, `device.rs:132-166` and `connection.rs:62` |
| `Io` | `#[from] std::io::Error` | transparent (inner message) | `file-formats` | `zip.write_all(..)?` when writing `.mnlgxdprog` / `.mnlgxdlib` (`sysex/program/file.rs:58, 127, 142`) |
| `Zip` | `String` | `zip error: {0}` | `file-formats` | 12 sites in `sysex/program/file.rs`: open archive, missing entry, read entry, create entry, finalize |

`pub type Result<T> = std::result::Result<T, Error>;` (`error.rs:131-132`). It is re-exported
at the crate root (`lib.rs:91`).

### `SysexError` (`error.rs:3-88`) [rust]

| Variant | Payload | `Display` | Raised by (examples) | Cause |
|---|---|---|---|---|
| `InvalidHeader` | `String` | `invalid SysEx header: {0}` | `frame.rs:269-325` (7 cases), `identity.rs:59-194`, `program/mod.rs:275-288`, `tuning.rs:343-352, 436-445`, `poly_chain.rs:230-317` | Too short, missing `F0`/`F7`, wrong manufacturer, wrong `3g` byte, wrong device ID, wrong MTS sub-IDs |
| `WrongFunctionId` | `expected: u8, found: u8` | `unexpected function ID 0x{found:02X}, expected 0x{expected:02X}` | `global.rs:348`, `program/mod.rs:213, 291`, `tuning.rs:233, 264`, `user_module.rs:497, 514, 532`, `poly_chain.rs:222, 272` | A typed parser was handed a frame with a different function ID |
| `PayloadTooShort` | `expected: usize, actual: usize` | `payload too short: expected {expected} bytes, got {actual}` | `global.rs:148`, `program/mod.rs:133`, `sequencer.rs:54, 167`, `synth.rs:374`, `tuning.rs:99, 164, 335, 428, 452`, `user_module.rs:121, 184, 264, 276`, `identity.rs:52, 148`, `poly_chain.rs:211, 262` | Blob or message shorter than its fixed size. `expected` is a **minimum**. |
| `InvalidMagic` | `expected: String, actual: String` | `invalid magic: expected {expected}, got {actual}` | `global.rs:157` (`GLOB`), `synth.rs:383` (`PROG`), `sequencer.rs:176` (`PREDSQ`, 6 bytes) | Blob magic mismatch |
| `ChecksumMismatch` | `expected: u8, actual: u8` | `checksum mismatch: expected 0x{expected:02X}, got 0x{actual:02X}` | `tuning.rs:364` | MTS bulk-dump XOR checksum |
| `InvalidProgramNumber` | `u16` | `invalid program number {0} (must be 0-499)` | `program/mod.rs:44` | `ProgramNumber::new(n > 499)` |
| `InvalidProgramNameChar` | `u8` | `invalid program name character: 0x{0:02X}` | `synth.rs:46-65` | Name byte outside the allowed set |
| `Crc32Mismatch` | `expected: u32, actual: u32` | `CRC32 mismatch: expected 0x{expected:08X}, got 0x{actual:08X}` | `user_module.rs:288` | User-slot payload CRC |
| `InvalidSlotIndex` | `slot: u8, max: u8` | `slot index {slot} out of range for module (max {max})` | `user_module.rs:343-350` | `slot >= max_slots(module)`. `max` is **exclusive** (16 or 8). |
| `InvalidModuleId` | `u8` | `invalid user module ID: {0}` | `user_module.rs:64-71` | Module byte not in 1..=4 |
| `Timeout` | `std::time::Duration` | `transaction timed out after {0:?}` | `transaction.rs:110, 115` | No complete SysEx before the deadline. The payload is the **configured** timeout, not the elapsed time. |
| `NakReceived` | `SysexStatus` | `device returned error status: {0}` | `transaction.rs:153, 181, 199` | Reply was a status frame with a NAK code (`0x24`, `0x26..=0x2F`) |
| `UnexpectedResponse` | `u8` | `unexpected response function ID 0x{0:02X}` | `transaction.rs:158, 205` | Reply function ID was neither the expected one nor a NAK |

The NAK codes and their `Display` names (`"Data Format Error"`, …) are listed in
[SysEx framing](../protocol/sysex-framing.md) (Korg MIDI Impl rev 1.01 NOTE 2). `0x25` is
unassigned, and `SysexStatus::from_byte` returns `None` for it.

### Error flow by layer

| Layer | Errors it can produce | Typical origin |
|---|---|---|
| Values ([typed values](typed-values.md)) | `OutOfRange`, `Sysex(InvalidProgramNumber)` | API arguments, and also wire bytes |
| Messages ([MIDI messages](../protocol/midi-messages.md)) | `InvalidMessage`, `OutOfRange` (high-bit data bytes) | Bytes received from a port |
| Codec ([7-bit codec](../protocol/korg-7bit-codec.md)) | `Codec` | SysEx payload bytes |
| Frames and blobs ([SysEx framing](../protocol/sysex-framing.md)) | `Sysex(InvalidHeader/WrongFunctionId/PayloadTooShort/InvalidMagic/…)`, `Codec`, `OutOfRange` | SysEx frames, file contents |
| Transactions ([SysEx transactions](sysex-transactions.md)) | `Sysex(Timeout/NakReceived/UnexpectedResponse)`, plus everything above, plus `MidiIo` | Device behaviour |
| Transport ([transport](transport.md)) | `MidiIo` | OS MIDI subsystem |
| Files | `Zip`, `Io`, plus blob errors | Archive contents, the writer |

### Feature dependence

- `Error` has 4 variants without features, 5 with `midi-io` and 7 with both defaults
  (`Cargo.toml:12-16`). Code that exhaustively matches `Error` therefore depends on
  features. `#[non_exhaustive]` forces a wildcard arm anyway.
- Both enums derive only `Debug` and `thiserror::Error`. They are **not** `Clone`,
  `PartialEq` or `Eq`, because `std::io::Error` is neither. Tests use `matches!` or
  `to_string()`.
- `SysexError` depends on `sysex::frame::SysexStatus` (`error.rs:83-84`). This creates a
  layering cycle: the error module imports from a higher layer.

## Invariants

- Encoders never fail when given values of the right types. All `to_midi_bytes` and
  `encode_7bit` functions are infallible. A Zig port keeps them error-free and asserts their
  output.
- `PayloadTooShort`: `actual < expected`.
- `InvalidSlotIndex`: `slot >= max`.
- `ChecksumMismatch` and `Crc32Mismatch`: `expected != actual`.
- `NakReceived(s)` implies `s.is_error()`, so `s != 0x23`.
- `WrongFunctionId`: `found != expected`. `UnexpectedResponse(f)`: `f` is not a known status
  code and not the expected function ID.
- `Timeout(d)`: `d` equals the transaction's configured timeout (default 5 s,
  `sysex/transaction.rs:48`).
- Every `Err` returned by a parser is caused by its input bytes. With the dead paths listed
  below removed, no parser error depends on internal state.

## Errata & gotchas

- **Inconsistent error for high-bit data bytes**:
  - Message parsers raise `OutOfRange("U7")`.
  - The codec raises `Codec`.
  - Frame parsing raises `Codec`, through the decoder.
  - The `FromMidiBytes` docs promise `InvalidMessage`.
  See [MIDI messages](../protocol/midi-messages.md).
- **`OutOfRange` mixes two kinds of error.** One kind is a programmer error: a caller passes
  `U4::new(16)`. The other is an operating error: a device or file sends a bad byte. The
  variant can't tell them apart.
- **Stringly-typed payloads** (`Codec`, `InvalidMessage`, `InvalidHeader`, `MidiIo`, `Zip`).
  Callers can't tell "port not found" from "disconnected", or "missing F0" from "wrong device
  ID", except by matching strings.
- **Misleading variant names.** `PayloadTooShort` is used for whole-message length checks,
  e.g. identity reply (`identity.rs:51-57`) and MTS (`tuning.rs:334-340`), not just payloads.
  `InvalidSlotIndex.max` is exclusive, yet `Display` prints it as "max".
- **File I/O errors are split.** Read failures inside the zip become `Zip(String)`
  (`file.rs:38-40, 101-102`). Write failures become `Io` (`file.rs:58`).
  `midi_file::…::write_to` returns `std::io::Result`, not the crate's `Result`
  (`midi_file.rs:914`).
- **Two names for the same thing.** Typed parsers use `WrongFunctionId`, and the transaction
  layer uses `UnexpectedResponse` (`transaction.rs:157-159`).
- **`ProgramNumber` uses a `SysexError`** (`InvalidProgramNumber`). Every other range check
  uses `OutOfRange`.
- **Dead error paths** that are really invariants:
  - The `PitchBend` parser's `i16::try_from(wire - 8192)` can never fail, since the wire value
    is at most 16383 (`message/channel.rs:198-200`).
  - `parse_sysex` maps a `U4::new(ch & 0x0F)` failure to `InvalidHeader`
    (`frame.rs:312-315`), which also can never happen.
- **Timeout semantics.** `receive_sysex` returns `Timeout` when the deadline passes or when
  `receive` returns `None`. A `MockMidiInput` that has run dry therefore produces `Timeout`
  immediately (`transaction.rs:103-142`).
- **Unknown status bytes** (`0x25`, `0x20..0x22`) are not treated as NAKs. They surface as
  `UnexpectedResponse` (`transaction.rs:150-159, 196-206`).
- **A NAK can be missed in raw replies.** `raw_request_response` checks for a NAK only when the
  reply is exactly 8 bytes long (`transaction.rs:169-188`).

## Test vectors

Exact `Display` strings, from `crates/minilogue-xd/src/error.rs:134-287` (the tests assert
substrings; the full strings follow from the `#[error]` attributes):

| Value | `to_string()` | Test line |
|---|---|---|
| `OutOfRange { "U7", 128, 0, 127 }` | `U7 value 128 out of range 0..=127` | 139 |
| `Codec("high bit set")` | `codec error: high bit set` | 153 |
| `InvalidMessage("bad status byte")` | `invalid MIDI message: bad status byte` | 159 |
| `InvalidHeader("missing F0")` | `invalid SysEx header: missing F0` | 169 |
| `WrongFunctionId { expected: 0x51, found: 0x40 }` | `unexpected function ID 0x40, expected 0x51` | 175 |
| `PayloadTooShort { 63, 10 }` | `payload too short: expected 63 bytes, got 10` | 186 |
| `InvalidMagic { "GLOB", "PROG" }` | `invalid magic: expected GLOB, got PROG` | 197 |
| `ChecksumMismatch { 0xAB, 0xCD }` | `checksum mismatch: expected 0xAB, got 0xCD` | 208 |
| `InvalidProgramNumber(500)` | `invalid program number 500 (must be 0-499)` | 219 |
| `InvalidProgramNameChar(0xFF)` | `invalid program name character: 0xFF` | 225 |
| `Crc32Mismatch { 0xDEADBEEF, 0xCAFEBABE }` | `CRC32 mismatch: expected 0xDEADBEEF, got 0xCAFEBABE` | 231 |
| `InvalidSlotIndex { 20, 16 }` | `slot index 20 out of range for module (max 16)` | 242 |
| `InvalidModuleId(99)` | `invalid user module ID: 99` | 250 |
| `Error::from(InvalidHeader("test"))` | `SysEx error: invalid SysEx header: test` | 256 |
| `Timeout(5 s)` | `transaction timed out after 5s` | 264 |
| `NakReceived(DataFormatError)` | `device returned error status: Data Format Error` | 272 |
| `UnexpectedResponse(0x40)` | `unexpected response function ID 0x40` | 281 |

Codec error (`crates/minilogue-xd/src/codec.rs:150-162`): `decode_7bit([0x00, 0x80])` returns
`Err(Codec(msg))`, where `msg` contains `"high bit set"`.

## Rust design

- `thiserror` derives `Display` and `std::error::Error`. `#[from]` provides the `?`
  conversions `SysexError → Error` and `io::Error → Error`. This is **incidental**: it is just a
  convenient way to write messages.
- What is **essential** is the classification: which inputs fail, at which layer, and with
  which distinguishing data. The Zig error sets below keep that.
- `#[non_exhaustive]` on both enums is about API stability and has no Zig equivalent.
  Zig error sets are open by construction and are merged with `||`.
- Errors allocate: `String`/`format!` builds a message at every failure site. That cost only
  matters on error paths, but it is still an allocation that TigerStyle forbids.

## Zig / TigerStyle notes

**Split each Rust error into an operating error or a programmer error:**

| Rust error | Origin | Zig treatment |
|---|---|---|
| `OutOfRange` from typed constructors (`U4`/`U7`/`U14`/`I14`/`U2`) | Caller | **Removed by types** (`u4`, `u7`, …). There is nothing to check. |
| `OutOfRange` for non-power-of-two ranges (program 0..499, bend 0..12, `EightBitHighRes` 0..200, LSB 0..4) | Caller | Public API boundary: `error.ValueOutOfRange`. Internal: `assert`. |
| `OutOfRange` / `InvalidMessage` / `Codec` from received bytes | Wire | Operating error: `MessageError` / `CodecError` |
| `InvalidHeader`, `WrongFunctionId`, `PayloadTooShort`, `InvalidMagic`, `ChecksumMismatch`, `Crc32Mismatch`, `InvalidModuleId`, `InvalidProgramNameChar` on parse | Wire or file | Operating error: `FrameError` / `BlobError` |
| `InvalidProgramNameChar`, `InvalidSlotIndex`, `InvalidModuleId` on build | Caller | Operating error only at the public API, otherwise `assert` |
| `Timeout`, `NakReceived`, `UnexpectedResponse` | Device | Operating error: `TransactionError` |
| `MidiIo` | OS | Operating error: `TransportError`, with one tag per cause |
| `Io`, `Zip` | OS or file | Operating error: `FileError` (the zip reader is hand-written, see [dependency elimination](../zig/dependency-elimination.md)) |
| Dead paths (pitch-bend `try_from`, frame channel nibble) | — | `assert` / `unreachable` |

**Proposed error sets (sketch).** Tag names follow the "noun + condition" style used on the
protocol pages. Merge sets with `||` only at the public surface, as described in
[TigerStyle mapping](../zig/tigerstyle-mapping.md).

```zig
// sketch
pub const MessageError = error{ MessageEmpty, MessageTruncated, StatusUnsupported, DataByteInvalid };
pub const CodecError = error{ WireByteInvalid, WireLengthInvalid, MsbPaddingInvalid };
pub const FrameError = error{
    FrameTooShort, StartByteMissing, EndByteMissing, ManufacturerMismatch,
    ChannelByteInvalid, DeviceIdMismatch, FunctionIdMismatch, FrameLengthMismatch,
} || CodecError;
pub const BlobError = error{
    BlobTooShort, MagicMismatch, ChecksumMismatch, Crc32Mismatch,
    ProgramNameByteInvalid, ProgramNumberInvalid, ModuleIdInvalid, SlotIndexInvalid,
    FieldValueInvalid,
};
pub const TransactionError = error{
    Timeout, Nak, ResponseUnexpected, FramesUnrelatedExceeded, FrameOverflow, InputOverrun,
};
pub const TransportError = error{
    ClientCreateFailed, PortNotFound, PortBusy, PortOpenFailed, WriteFailed, Disconnected,
};
pub const ValueError = error{ValueOutOfRange};
```

**Diagnostics without allocation.** Zig errors carry no payload. Wherever Rust attaches data,
fill an optional caller-owned struct instead, and leave the error set unchanged:

```zig
// sketch
pub const Diagnostics = struct {
    offset: u32 = 0,      // byte offset of the failing byte in the input
    expected: u32 = 0,    // expected length / function ID / checksum
    actual: u32 = 0,      // found value
    status: u8 = 0,       // NAK status byte (0x24..0x2F)
    os_code: i32 = 0,     // OSStatus / errno / MMRESULT
};
pub fn parse_frame(bytes: []const u8, diag: ?*Diagnostics) FrameError!FrameView { ... }
```

- No `String` and no formatting at failure sites. Format messages, if needed at all, in
  application code from the tag and the `Diagnostics` fields.
- Assert the paired invariants instead of trusting error paths:
  - `parse(encode(x)) == x` in tests.
  - Encoders assert `byte < 0x80` on output.
  - Decoders assert the consumed length equals the expected length.
- Keep error sets **closed and small per layer** so callers can `switch` exhaustively. Avoid
  `anyerror`.
- Use one canonical tag for "data byte with bit 7 set" per layer: `DataByteInvalid` in messages
  and `WireByteInvalid` in the codec. Never mask silently.
- `Timeout` is computed by the caller's loop against a monotonic clock. See
  [transport](transport.md).

## Open questions

- Should a NAK with an unknown code (`0x25`, `0x20..0x22`, `0x30+`) be reported as
  `error.Nak` with the raw status in `Diagnostics`? Rust reports `UnexpectedResponse` instead.
- Should the transaction layer distinguish "deadline passed while receiving a partial SysEx"
  from "nothing received"? Rust reports both as `Timeout`.
- Should a raw reply always be checked for a status frame, whatever its length? See
  [SysEx transactions](sysex-transactions.md).

## Sources

- `crates/minilogue-xd/src/error.rs:1-132` — `SysexError`, `Error`, `Result`.
- `crates/minilogue-xd/src/error.rs:134-287` — `Display` tests.
- `crates/minilogue-xd/src/codec.rs:69-81, 150-162` — the codec error.
- `crates/minilogue-xd/src/message/channel.rs:195-204` — the pitch-bend dead path.
- `crates/minilogue-xd/src/sysex/frame.rs:89-185` — `SysexStatus`.
- `crates/minilogue-xd/src/sysex/frame.rs:262-325` — `parse_sysex` checks.
- `crates/minilogue-xd/src/sysex/transaction.rs:48, 100-208, 362-376` — timeout, NAK and
  unexpected response, `FailingOutput`.
- `crates/minilogue-xd/src/sysex/program/file.rs:29-145` — `Zip` and `Io`.
- `crates/minilogue-xd/src/transport.rs:149-321` — `MidiIo` sites.
- `crates/minilogue-xd/src/device.rs:132-166` — `MidiIo` sites.
- `crates/minilogue-xd/Cargo.toml:12-23` — features and the `thiserror` dependency.
- Korg MIDI Impl rev 1.01 NOTE 2 — status codes.
