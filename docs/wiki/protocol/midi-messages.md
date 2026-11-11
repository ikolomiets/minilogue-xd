---
title: MIDI channel, system common and real-time messages
category: protocol
sources:
  - crates/minilogue-xd/src/message/mod.rs
  - crates/minilogue-xd/src/message/channel.rs
  - crates/minilogue-xd/src/message/common.rs
  - crates/minilogue-xd/src/message/realtime.rs
  - crates/minilogue-xd/src/message/types.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - docs/wayfinder/research/xd-midi-capabilities.md
  - https://cdn.korg.com/us/support/download/files/5227b0b2f7c1da5acd0911ac3e8c4f38.txt
related:
  - design/typed-values.md
  - design/errors.md
  - protocol/cc-map.md
  - protocol/nrpn-map.md
  - protocol/parameter-encodings.md
  - protocol/sysex-framing.md
  - protocol/global-blob.md
  - protocol/errata.md
  - design/realtime-controller.md
  - design/sysex-transactions.md
updated: 2026-10-11
---

# MIDI channel, system common and real-time messages

> This page covers every non-SysEx MIDI message the minilogue xd sends or receives, including
> the exact bytes and how the Rust crate parses raw bytes into a typed `MidiMessage`. The most
> important fact: the parser takes **one complete message per call**. It does not support
> running status, it ignores trailing bytes, and it switches on the full status byte for
> `0xF0..0xFF` but only on the high nibble for channel messages. CC 64, 120, 122 and 123 come
> out as their own variants. Every other CC, including bank select CC 0 and CC 32, comes out as
> a generic `ControlChange`.

## Facts

### Status byte classes

| Byte range | Class | How the Rust parser dispatches |
|---|---|---|
| `0x00..0x7F` | Data byte | Rejected when it appears first: "unrecognized status byte" |
| `0x80..0xEF` | Channel voice/mode. Bits 7..4 = type, bits 3..0 = channel `n` (0..15 means MIDI ch 1..16) | `status & 0xF0` |
| `0xF0..0xF7` | System exclusive / common | Full byte. Only `0xF2` is accepted. `0xF0` SysEx is handled elsewhere ([SysEx framing](sysex-framing.md)) |
| `0xF8..0xFF` | System real-time (single byte) | Full byte. Only `0xF8 FA FB FC FE` are accepted |

The device's receive/transmit channel is the global "MIDI Ch" setting (0..15 = ch 1..16)
[spec] (Korg MIDI Impl rev 1.01 TABLE 1 offset 16; see [global blob](global-blob.md)).

### Message catalogue

TX = the device transmits it. RX = the device recognizes it. Gate = the global setting that
must be On ([global blob](global-blob.md) TABLE 1 offsets 17-24).

| Message | Bytes | Data layout | TX [spec] | RX [spec] | Gate [spec] | Rust type |
|---|---|---|---|---|---|---|
| Note Off | 3 | `8n kk vv` | yes, `vv` fixed `0x40` | yes, `vv` 0..127 | — | `NoteOff` |
| Note On | 3 | `9n kk vv` | yes, `vv` 1..127 | yes, `vv` 1..127; `9n kk 00` = Note Off | — | `NoteOn` |
| Control Change | 3 | `Bn cc vv` | see [CC map](cc-map.md) | see [CC map](cc-map.md) | Tx CC / Rx CC | `ControlChange` |
| Damper (Hold1) | 3 | `Bn 40 vv` | `vv` = 0 or 127 | `vv` 0..127 | Tx CC / Rx CC | `Damper` |
| All Sound Off | 3 | `Bn 78 00` | no | yes | Rx CC | `AllSoundOff` |
| Local Sw | 3 | `Bn 7A vv` | no | yes, `vv` 0..127 | Rx CC | `LocalControl` |
| All Note Off | 3 | `Bn 7B 00` | no | yes | Rx CC | `AllNotesOff` |
| Bank Select MSB | 3 | `Bn 00 vv` | `vv` = 0 | `vv` = 0 | Tx/Rx Prog Chg | generic `ControlChange`; `BankSelect` pair |
| Bank Select LSB | 3 | `Bn 20 vv` | `vv` 0..4 | `vv` 0..4 | Tx/Rx Prog Chg | generic `ControlChange`; `BankSelect` pair |
| Program Change | 2 | `Cn pp` | `pp` 0..99 | `pp` 0..99 | Tx/Rx Prog Chg | `ProgramChange` |
| Channel Pressure | 2 | `Dn vv` | no | yes ("as After Touch") | none listed | `ChannelPressure` |
| Pitch Bend | 3 | `En ll mm` | yes | yes | Tx/Rx PitchBend | `PitchBend` |
| Song Position Pointer | 3 | `F2 ll mm` | no | yes | none listed | `SongPositionPointer` |
| Timing Clock | 1 | `F8` | yes | yes | TX: "En Rx Transport"; RX: Clock = Auto(USB/MIDI) | `TimingClock` |
| Start | 1 | `FA` | yes | yes | same as Timing Clock | `Start` |
| Continue | 1 | `FB` | **no** | yes | RX: Clock = Auto | `Continue` |
| Stop | 1 | `FC` | yes | yes | same as Timing Clock | `Stop` |
| Active Sensing | 1 | `FE` | yes | yes | TX: "En Rx Transport"; RX: none | `ActiveSensing` |

Not supported by the device or by the crate: polyphonic key pressure (`An`), MTC quarter frame
(`F1`), Song Select (`F3`), Tune Request (`F6`), System Reset (`FF`). SysEx (`F0 … F7`) is
covered by [SysEx framing](sysex-framing.md), [device identity](device-identity.md),
[tuning](tuning.md) and [poly chain](poly-chain.md).

### Control changes that are not ordinary parameters

All of these share status `Bn`. The second byte decides what the message means.

| CC | Meaning | Spec value | Rust `parse_midi_bytes` result | Rust encode |
|---|---|---|---|---|
| 0 | Bank Select MSB | `00` | `ControlChange` | `BankSelect` emits `Bn 00 msb` |
| 32 (`0x20`) | Bank Select LSB | `00..04` | `ControlChange` | `BankSelect` emits `Bn 20 lsb` |
| 63 (`0x3F`) | LSB prefix for 10-bit CC values | `00..7F` | `ControlChange` | see [parameter encodings](parameter-encodings.md) |
| 6, 38, 98, 99 | Data Entry and NRPN address | — | `ControlChange` | see [NRPN map](nrpn-map.md) |
| 64 (`0x40`) | Damper / Hold1 | TX 0 or 127, RX 0..127 | `Damper { value: U7 }` | `Bn 40 vv` |
| 120 (`0x78`) | All Sound Off | `00` | `AllSoundOff` (value byte **ignored**) | `Bn 78 00` |
| 121 (`0x79`) | Reset All Controllers | not in spec | `ControlChange` | — |
| 122 (`0x7A`) | Local Sw | `00..7F` | `LocalControl`: `00` → Off, anything else → On (**not** U7-checked) | Off → `00`, On → `7F` |
| 123 (`0x7B`) | All Note Off | `00` | `AllNotesOff` (value byte **ignored**) | `Bn 7B 00` |
| 124..127 | Omni/mono/poly mode | not in spec | `ControlChange` | — |

### Pitch bend (14-bit, signed in the API, unsigned on the wire)

```text
wire   = (mm << 7) | ll           ; ll = bits 6..0, mm = bits 13..7; LSB is sent first
signed = wire - 8192              ; centre 0x2000 → 0
encode: wire = signed + 8192 ; ll = wire & 0x7F ; mm = (wire >> 7) & 0x7F
```

| signed (`I14`) | wire | bytes (ch 0) |
|---|---|---|
| -8192 | `0x0000` | `E0 00 00` |
| 0 | `0x2000` | `E0 00 40` |
| +8191 | `0x3FFF` | `E0 7F 7F` |

The Rust decoder masks both data bytes with `& 0x7F`, so a high bit is silently dropped rather
than rejected [rust]. The `i16::try_from` error branch can never fire, because the masked wire
range 0..16383 maps exactly onto -8192..8191 (`channel.rs:181-207`).

### Song Position Pointer

- [rust]: `F2 ll mm` carries a generic 14-bit beat count `(mm << 7) | ll`, LSB first. The
  data bytes are masked with `& 0x7F`. The doc comment says "each beat is 6 MIDI clocks".
- [spec] (§2-2): the second byte is drawn as `0xxx xxxp` and the third as `0ppp pppp`.
  `pppp` is 0..255 and equals `(STEP 1..16) × Step Resolution × 16`. The `x` bits are ignored.
  So the device uses only 8 bits of position. The diagram also puts 1 bit in the
  first data byte and 7 bits in the second, which does not match standard LSB-first SPP. See
  Open questions.

### Bank select and program numbers

The device stores 500 programs: 5 banks × 100 [spec] (TABLE 1 favourites "0~499=Program
1~500"). The spec's receive/transmit form for program `p` (0..499) is:

```text
Bn 00 00                ; CC0  = 0
Bn 20 (p / 100)         ; CC32 = bank 0..4
Cn (p % 100)            ; Program Change 0..99
```

The crate has two encoders, and they **disagree** with each other:

| Encoder | CC 0 | CC 32 | PC | Matches spec? |
|---|---|---|---|---|
| `BankSelect` message type (`channel.rs:470-556`) | `msb` (caller-chosen) | `lsb` checked to be 0..4 | — | yes, if `msb = 0` |
| `RealtimeController::program_change` (`controller.rs:461-469`) | `p / 100` (bank) | `p % 100` (slot) | `p % 100` | **no** |

`ProgramNumber::bank()` is `p / 100` and `slot_in_bank()` is `p % 100`
(`sysex/program/mod.rs:54-62`). For more detail see [errata](errata.md) and the
[realtime controller](../design/realtime-controller.md).

### `parse_midi_bytes` algorithm (Rust behaviour)

```text
parse(bytes):
  if len == 0                      -> InvalidMessage("empty message")
  s = bytes[0]
  if s >= 0xF0:
    F2 -> SPP.parse      (len >= 3, else InvalidMessage; data masked & 0x7F)
    F8/FA/FB/FC/FE -> that real-time unit value (only bytes[0] is checked)
    else -> InvalidMessage("unrecognized status byte 0x..")   ; F0 F1 F3..F7 F9 FD FF
  switch s & 0xF0:
    80 -> NoteOff.parse            (len >= 3)
    90 -> if len >= 3 and bytes[2] == 0: NoteOff{ch, key, velocity 0}
          else NoteOn.parse         (len >= 3)
    B0 -> if len < 3 -> InvalidMessage("CC message requires 3 bytes")
          switch bytes[1]: 120 AllSoundOff | 122 LocalControl | 123 AllNotesOff
                           | 64 Damper | else ControlChange
    C0 -> ProgramChange.parse      (len >= 2)
    D0 -> ChannelPressure.parse    (len >= 2)
    E0 -> PitchBend.parse          (len >= 3; data masked & 0x7F)
    else (00..70) -> InvalidMessage("unrecognized status byte 0x..")
```

Per-type parsers follow these rules [rust]:

- **Length check is a minimum, not an exact length.** Any bytes after the expected length are
  ignored. For example, `F8 00 00` parses as Timing Clock.
- **Status check**: `bytes[0] & 0xF0` must equal the type's nibble. Real-time types instead
  compare the whole byte. Channel = `bytes[0] & 0x0F`. The channel conversion cannot fail.
- **Data bytes** (`key`, `velocity`, `controller`, `value`, `program`, `pressure`) go through
  `U7::new`. A byte ≥ `0x80` gives `Error::OutOfRange{type_name:"U7"}`, **not**
  `InvalidMessage`. Pitch Bend and SPP mask instead of rejecting. AllSoundOff and AllNotesOff
  never read the value byte. LocalControl compares it to 0 without range-checking it.
- **No running status.** A leading data byte is an error. Callers must pass a complete message
  that starts with its status byte.
- **No realtime interleaving.** A real-time byte embedded inside another message is not
  extracted.
- `BankSelect` is not a `MidiMessage` variant, because it spans two messages. Its own parser
  needs ≥ 6 bytes. It checks that `bytes[0] & 0xF0 == 0xB0`, `bytes[1] == 0`,
  `bytes[3] & 0xF0 == 0xB0` and `bytes[4] == 32`, then applies the LSB ≤ 4 check
  (`OutOfRange{type_name:"BankSelect LSB"}`). It does **not** check that the two channels
  match.

## Invariants

- An encoded channel or system message is 1, 2 or 3 bytes long. A `BankSelect` pair is
  exactly 6.
- `encode(m)[0] & 0x80 != 0`, and every following byte satisfies `b & 0x80 == 0`.
- For channel messages, `encode(m)[0] & 0x0F == m.channel` and `encode(m)[0] & 0xF0` is the
  type nibble.
- Pitch bend: `-8192 <= signed <= 8191` and `0 <= wire <= 16383`.
- `decode(encode(m)) == m` for every variant **except** `NoteOn` with velocity 0, which
  decodes as `NoteOff`.
- Real-time messages are exactly one byte: `0xF8`, `0xFA`, `0xFB`, `0xFC` or `0xFE`.
- Bank LSB sent to the device is ≤ 4. Program Change sent to the device is ≤ 99 [spec].

## Errata & gotchas

- **Bank select encoding** [spec] vs [rust]: the spec (§1-1, §2-1) gives CC0 = 0 and
  CC32 = bank 0..4. `RealtimeController::program_change` sends CC0 = bank and CC32 = slot
  0..99, as shown by the test vector `B0 00 02`, `B0 20 32`, `C0 32` for program 250
  (`controller.rs:1118-1130`). Under the spec, CC32 = 50 is out of range. The README says
  bank select "differs from the spec in practice" (`README.md:224-225`), but no [hw] capture
  or test in the repo backs the controller's form (`xd-midi-capabilities.md:53`). The crate's
  own `BankSelect` type follows the spec. Treat this as unresolved; see [errata](errata.md).
- **TX Note Off velocity** [spec]: the device sends `8n kk 40`. `RealtimeController::stop_note`
  sends `8n kk 00` [rust] (`controller.rs:443-447`). Both are legal.
- **TX real-time gate naming** [spec]: §1-2 says F8/FA/FC/FE are sent when "En Rx Transport"
  is On. That setting (TABLE 1 offset 18) has "Rx" in its name. The device never transmits
  `FB` Continue.
- **Active Sensing RX is ungated** [spec] (§2-3 `*7` covers F8/FA/FB/FC but not FE).
- **CV IN CC 118/119** [spec]: transmitted only when CV IN Mode = Modulation and PolyChain =
  Master, and **never over USB MIDI** (§1-1 `*1-5`).
- **Velocity-0 Note On** [spec, rust]: the device treats `9n kk 00` as Note Off. The Rust
  `NoteOn` struct still allows velocity 0 on encode (the doc says 1..=127 but nothing enforces
  it, `channel.rs:31-40`). The result reaches the device as Note Off.
- **High-bit data bytes**: rejected with `OutOfRange` for most types, silently masked for
  Pitch Bend and SPP, and ignored for AllSoundOff/AllNotesOff/LocalControl. The `FromMidiBytes`
  doc comment says it returns `InvalidMessage`, which is not always true (`channel.rs:16-25`).
- **Edition differences**: CC 7/11 (volume/expression) appear in the E8 owner's-manual chart
  but not in the rev 1.01 TXT receive table. The crate follows the TXT
  (`xd-midi-capabilities.md:54`). See [CC map](cc-map.md).

## Test vectors

All channel examples use ch 0 unless stated.

| Bytes | Parsed as / encoded from | Citation |
|---|---|---|
| `90 3C 64` | NoteOn key 60 vel 100 (round trip) | `crates/minilogue-xd/src/message/channel.rs:561-572` |
| `9F 00 7F` (first byte) | NoteOn ch 15 | `crates/minilogue-xd/src/message/channel.rs:574-583` |
| `83 48 40` | NoteOff ch 3 key 72 vel 64 | `crates/minilogue-xd/src/message/channel.rs:597-608` |
| `90 3C 00` | `parse_midi_bytes` → NoteOff key 60 vel 0 | `crates/minilogue-xd/src/message/mod.rs:189-199` |
| `C5 2A` | ProgramChange ch 5 prog 42 | `crates/minilogue-xd/src/message/channel.rs:622-632` |
| `E0 00 40` / `E0 00 00` / `E0 7F 7F` | PitchBend 0 / -8192 / +8191 | `crates/minilogue-xd/src/message/channel.rs:655-683` |
| `B2 4A 64` | ControlChange ch 2 cc 74 val 100 | `crates/minilogue-xd/src/message/channel.rs:708-719` |
| `D0 50` | ChannelPressure 80 | `crates/minilogue-xd/src/message/channel.rs:733-743` |
| `B0 78 00` | AllSoundOff | `crates/minilogue-xd/src/message/channel.rs:766-775` |
| `B4 7B 00` | AllNotesOff ch 4 | `crates/minilogue-xd/src/message/channel.rs:784-793` |
| `B0 7A 00` / `B0 7A 7F` / `B0 7A 40` | LocalControl Off / On / On | `crates/minilogue-xd/src/message/channel.rs:802-831` |
| `B0 40 7F` | Damper 127 | `crates/minilogue-xd/src/message/channel.rs:840-850` |
| `B0 00 00 B0 20 02` | BankSelect msb 0 lsb 2 | `crates/minilogue-xd/src/message/channel.rs:859-871` |
| `F2 00 00` / `F2 00 40` / `F2 7F 7F` | SPP 0 / 8192 / 16383 | `crates/minilogue-xd/src/message/common.rs:54-86` |
| `F8` `FA` `FB` `FC` `FE` | Clock, Start, Continue, Stop, Active Sensing | `crates/minilogue-xd/src/message/realtime.rs:114-152` |

Rejection vectors:

| Bytes | Reason | Citation |
|---|---|---|
| (empty) | empty message | `crates/minilogue-xd/src/message/mod.rs:292-295` |
| `00` | data byte as status | `crates/minilogue-xd/src/message/mod.rs:297-301` |
| `F1` | unsupported system status | `crates/minilogue-xd/src/message/mod.rs:303-306` |
| `B0 4A` | CC shorter than 3 | `crates/minilogue-xd/src/message/mod.rs:308-311` |
| `90 3C`, `80`, `C0`, `D0`, `E0 00`, `F2 00` | too short for type | `channel.rs:590-593, 615-618, 648-651, 701-704, 759-762`; `common.rs:93-96` |
| `80 3C 64` as NoteOn, `B0 2A` as ProgramChange, `F3 00 00` as SPP | wrong status | `channel.rs:585-588, 643-646`; `common.rs:88-91` |
| `B0 79 00` as AllSoundOff; `B0 00 00 B0 20 05` via `new` | wrong CC / LSB > 4 | `channel.rs:777-780, 883-891` |
| `F9`, `FB`, `FD`, `FF`, `FA` for neighbouring real-time types | wrong byte | `realtime.rs:114-152` |

## Rust design

- Each message is a small `Copy` struct with typed fields (`U4` channel, `U7` data, `I14`
  bend, `U14` SPP; see [typed values](../design/typed-values.md)). Two traits sit on top:
  `ToMidiBytes::to_midi_bytes(&self) -> Vec<u8>` and
  `FromMidiBytes::from_midi_bytes(&[u8]) -> Result<Self>` (`channel.rs:10-25`).
- `MidiMessage` is a `#[non_exhaustive]` `Copy` enum with 16 variants that wraps those
  structs. Its `to_midi_bytes` is a match that delegates to each variant
  (`mod.rs:20-78`).
- **Incidental Rust idioms**: returning `Vec<u8>` (a heap allocation for 1 to 6 bytes),
  `String` error payloads built with `format!`, `#[non_exhaustive]`, and one trait impl per
  struct.
- **Essential design**: a closed set of message kinds; typed, range-checked fields; and the
  specialised-CC dispatch on `bytes[1]`. Velocity-0 normalisation to NoteOff happens in the
  dispatcher, not in `NoteOn::from_midi_bytes`. Parsing is stateless.
- The `RealtimeController` builds most outgoing bytes by hand
  (`[0x90 | ch, note, vel]`, `controller.rs:438-469`) rather than through these types. Zig
  should use a single encoder.

## Zig / TigerStyle notes

- Model `Message` as a tagged `union(enum)` with fields of type `u4`, `u7`, `i14` and `u14`.
  Encode into a caller-provided `*[3]u8` and return the length as a `u2`. No allocation.
  Encode `BankSelect` with a separate function into `*[6]u8`, or better, as three separate
  3-, 3- and 2-byte messages.
- Keep a comptime table, indexed by `status >> 4` (`0x8..0xE`), that gives the expected
  message length (`3,3,3,3,2,2,3`; `0xA` is unsupported). Use it in place of seven per-type
  length checks. Keep a second table for `0xF0..0xFF`.
- Parse error set (operating errors, from wire input):
  `error{ MessageEmpty, MessageTruncated, StatusUnsupported, DataByteInvalid }`. Choose one
  behaviour for high-bit data bytes and apply it to **every** type: reject with
  `DataByteInvalid`. Do not mask.
- Decide explicitly how to treat trailing bytes. Recommended: require
  `bytes.len == expected_len` and assert it in the caller that frames messages.
- If the Zig port reads a raw DIN byte stream rather than OS-framed packets, put a small
  byte-at-a-time framer in front of `parse`. The framer holds the running-status byte,
  passes real-time bytes `0xF8..0xFF` straight through even mid-message, and enforces a
  3-byte maximum. It is bounded and has no recursion. USB-MIDI packets from the OS already
  arrive framed ([USB ports](usb-ports.md)).
- Assertions to add: `assert(channel <= 15)` is implied by `u4`; after encode,
  `assert(target[0] & 0x80 != 0)` and `assert(target[i] & 0x80 == 0)` for i ≥ 1; for program
  change sent to the device, `assert(program <= 99)`; for bank LSB, `assert(bank <= 4)`.
- Pitch bend: store `i14` and convert with `@as(u14, @intCast(@as(i15, value) + 8192))`.
  The conversion cannot overflow, so it needs no error path. (Sketch only; check the exact
  cast built-ins against the Zig version you use.)

```zig
// sketch
pub const Message = union(enum) {
    note_off: struct { channel: u4, key: u7, velocity: u7 },
    note_on: struct { channel: u4, key: u7, velocity: u7 },
    control_change: struct { channel: u4, controller: u7, value: u7 },
    program_change: struct { channel: u4, program: u7 },
    channel_pressure: struct { channel: u4, pressure: u7 },
    pitch_bend: struct { channel: u4, value: i14 },
    song_position: u14,
    timing_clock, start, @"continue", stop, active_sensing,
};
pub fn encode(message: Message, target: *[3]u8) u2 { ... }
pub fn parse(source: []const u8) ParseError!Message { ... }
```

Whether CC 64/120/122/123 get their own union tags or are recognised with helper predicates
on `control_change` is a design choice. Helper predicates keep `parse` total and simple.

## Open questions

- **Bank select (hardware)**: which form does the XD actually accept: CC0 = 0 / CC32 = bank /
  PC = slot [spec], or CC0 = bank / CC32 = slot [rust controller]? Capture on hardware. See
  [errata](errata.md).
- **SPP bit layout**: does the XD read SPP as standard `(mm << 7) | ll` (in 16th-note
  units), or as the §2-2 diagram literally shows? The [rust] code uses the standard form.
- **AllSoundOff / AllNotesOff value byte**: the spec requires `00`. The crate accepts any
  value. Does the device act on `Bn 78 7F`?
- **Suspected bug**: `BankSelect::from_midi_bytes` accepts `B0 00 00 B1 20 02`, where the
  second CC is on a different channel (`channel.rs:515-555`).
- **Suspected inconsistency**: data bytes are rejected for some types and masked for others,
  and the documented error kind does not match the real one (see Errata).

## Sources

- `crates/minilogue-xd/src/message/mod.rs:20-78` — `MidiMessage` enum and encode delegation.
- `crates/minilogue-xd/src/message/mod.rs:80-169` — `parse_midi_bytes`.
- `crates/minilogue-xd/src/message/mod.rs:171-327` — dispatcher tests.
- `crates/minilogue-xd/src/message/channel.rs:10-25` — traits.
- `crates/minilogue-xd/src/message/channel.rs:31-288` — voice messages.
- `crates/minilogue-xd/src/message/channel.rs:290-476` — mode/RX-only CCs.
- `crates/minilogue-xd/src/message/channel.rs:470-556` — `BankSelect`.
- `crates/minilogue-xd/src/message/channel.rs:558-907` — tests.
- `crates/minilogue-xd/src/message/common.rs:10-97` — Song Position Pointer and tests.
- `crates/minilogue-xd/src/message/realtime.rs:1-153` — real-time messages and tests.
- `crates/minilogue-xd/src/controller.rs:437-474` — hand-built note, bend, program change and
  CC 123 messages.
- `crates/minilogue-xd/src/controller.rs:1118-1138` — controller test vectors.
- `crates/minilogue-xd/src/sysex/program/mod.rs:30-63` — `ProgramNumber` bank/slot split.
- `README.md:221-227` — README errata claim.
- `docs/wayfinder/research/xd-midi-capabilities.md:44-56` — DIN timing, discrepancies.
- Korg MIDI Impl rev 1.01 §1-1 (TX channel messages, `*1-1`..`*1-5`), §1-2 (TX real-time),
  §2-1 (RX channel messages, `*5-1`..`*5-5`), §2-2 (SPP), §2-3 (RX real-time, `*7`),
  TABLE 1 offsets 16-24 (MIDI Ch, Clock Source, Rx/Tx gates).
