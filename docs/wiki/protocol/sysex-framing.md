---
title: Korg SysEx framing, function IDs and status codes
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/mod.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/sysex/global.rs
  - crates/minilogue-xd/src/sysex/tuning.rs
  - crates/minilogue-xd/src/sysex/user_module.rs
  - crates/minilogue-xd/src/sysex/poly_chain.rs
  - crates/minilogue-xd/src/codec.rs
  - crates/minilogue-xd/src/error.rs
  - docs/wayfinder/research/xd-midi-capabilities.md
  - docs/korg-user-guide/book.md
related:
  - protocol/korg-7bit-codec.md
  - protocol/device-identity.md
  - protocol/poly-chain.md
  - protocol/tuning.md
  - protocol/program-blob.md
  - protocol/global-blob.md
  - protocol/user-modules.md
  - protocol/usb-ports.md
  - protocol/errata.md
  - design/sysex-transactions.md
  - design/errors.md
  - zig/memory-budget.md
updated: 2026-11-11
---

# Korg SysEx framing, function IDs and status codes

> Every minilogue xd-specific System Exclusive message uses one frame:
> `F0 42 3g 00 01 51 <function ID> <payload…> F7`, where `g` is the device's MIDI global channel.
> The function ID byte says what the message is: a request, a data dump, a poly-chain note, or a
> status (ACK/NAK) reply. This page owns the header layout, the complete function-ID table (the
> direction, payload, encoding, reply and wire length of each ID) and the status codes. It also
> owns the generic build and parse rules. The payload's encoding varies by function ID:
> - **Data dumps** are packed with the Korg 7-bit codec ([NOTE 1](korg-7bit-codec.md)).
> - **Request parameters, poly-chain notes and the API-version reply** are raw 7-bit bytes.
>
> A parser must know which kind it is handling. A generic "decode everything after byte 6" parser
> corrupts the raw-byte messages.

## Frame layout

| Offset | Value | Meaning |
|--------|-------|---------|
| 0 | `0xF0` | System Exclusive start |
| 1 | `0x42` | Korg manufacturer ID |
| 2 | `0x30 \| g` | bits 7..4 = `0011`; bits 3..0 = MIDI global channel `g` (0–15) [spec] |
| 3 | `0x00` | Device ID byte 1 |
| 4 | `0x01` | Device ID byte 2 |
| 5 | `0x51` | Device ID byte 3 (minilogue xd) |
| 6 | function ID | See [function-ID table](#function-id-table) |
| 7 … n−2 | payload | 0 or more bytes, every byte `< 0x80` |
| n−1 | `0xF7` | End of Exclusive (EOX) |

- The minimum frame is 8 bytes (header + function ID + EOX), with no payload. Requests without
  parameters and all status replies have this shape.
- `g` is the *global* MIDI channel from the device's global settings. It is not a per-message
  channel. Its stored value is in [global blob](global-blob.md).
  - The Rust examples always use `g = 0`.
  - Whether the device ignores frames whose `g` differs from its global channel is not verified
    (see [Open questions](#open-questions)).
- `0x51` is both the third device-ID byte (offset 5) and the GLOBAL DATA DUMP function ID
  (offset 6). This is a coincidence: always match by offset, never by searching for a value.
- Other SysEx messages share the wire but do **not** use this header:
  - Universal Identity (`F0 7E …`) and Korg Search Device (`F0 42 50 …`): see
    [device identity](device-identity.md).
  - MIDI Tuning Standard (`F0 7E/7F … 08 …`): see [tuning](tuning.md).

Constants: `KORG_ID = 0x42`, `DEVICE_ID = [00 01 51]`
(`crates/minilogue-xd/src/sysex/mod.rs:18-22`). Frame layout as implemented:
`crates/minilogue-xd/src/sysex/frame.rs:1-9`, `220-249`.

### Payload kinds

| Kind | Meaning | Function IDs |
|------|---------|--------------|
| none | Nothing between the function ID and `F7` | `0E 10 17`, status `23–2F`; Rust's `14 15` |
| raw | Every byte is a 7-bit value, used as-is | request parameters (`18 19 1A 1B 1C 1D 1E`), `47`, `60`, `61`; spec `44 45` (see below) |
| encoded | 8-bit blob packed per [NOTE 1](korg-7bit-codec.md) (8 wire bytes per 7 data bytes) | `40`, `51`, `48`, `49`, `4A`; `4C` after 2 raw program-number bytes |

Two kinds are mixed in one payload:

- `0x4C` has a raw program-number prefix followed by an encoded blob.
- In the spec, `0x44` and `0x45` start with a raw scale-number byte `ss`, followed by data.

**[spec]** The spec references NOTE 1 for exactly the "encoded" rows. It does not reference
NOTE 1 for `0x44`/`0x45`, and it gives them no "Conv. Size" line. Every TABLE 3/4 value is 0–127,
so an unencoded payload is plausible. The Rust code encodes them anyway; see [tuning](tuning.md).

## Function-ID table

Direction: **H→XD** means the host sends it and the device receives it ("R" in the spec). **XD→H**
means the device transmits it ("T"). **↔** means both ("R/T").

The *Rust frame* column is the total wire length in bytes, including `F0` and `F7`, as the Rust
builders emit or expect it. Encoded lengths are computed with the [codec](korg-7bit-codec.md)
length formula.

### Requests and commands (host → device)

| ID | Spec name | Payload after function ID | Device answers with | Rust builder | Rust frame |
|----|-----------|---------------------------|---------------------|--------------|-----------:|
| `0x0E` | GLOBAL DATA DUMP REQUEST | none | `0x51`, or `0x24` | `global::build_global_request` | 8 |
| `0x10` | CURRENT PROGRAM DATA DUMP REQUEST | none | `0x40`, or `0x24` | `program::build_current_program_request` | 8 |
| `0x14` | USER SCALE DATA DUMP REQUEST | spec: `ss` = scale 0–5 (bits 2..0); **Rust: none** | `0x44`, or `0x24` | `tuning::build_user_scale_request` | 8 (spec 9) |
| `0x15` | USER OCTAVE DATA DUMP REQUEST | spec: `ss` = octave 0–5 (bits 2..0); **Rust: none** | `0x45`, or `0x24` | `tuning::build_user_octave_request` | 8 (spec 9) |
| `0x17` | USER API VERSION REQUEST | none | `0x47`, or `0x24` if busy | `user_module::build_api_version_request` | 8 |
| `0x18` | USER MODULE INFO REQUEST | module ID (1 modfx, 2 delfx, 3 revfx, 4 osc) | `0x48`, or `0x24` if busy | `user_module::build_module_info_request` | 9 |
| `0x19` | USER SLOT STATUS REQUEST | module ID, slot | `0x49`, or `0x24` if busy | `user_module::build_slot_status_request` | 10 |
| `0x1A` | USER SLOT DATA REQUEST | module ID, slot | `0x4A`, or `0x24` if busy | `user_module::build_slot_data_request` | 10 |
| `0x1B` | CLEAR USER SLOT | module ID, slot | `0x23` ok; `0x2C` bad module; `0x2D` bad slot; `0x2F` internal | `user_module::build_clear_slot` | 10 |
| `0x1C` | PROGRAM DATA DUMP REQUEST (1 PROG) | `pp` (program LSB), `PP` (program MSB) | `0x4C`, or `0x24` | `program::build_program_request` | 10 |
| `0x1D` | CLEAR USER MODULE | module ID | `0x23` ok; `0x2C` bad module; `0x2F` internal | `user_module::build_clear_module` | 9 |
| `0x1E` | SWAP USER DATA | module ID, slot A, slot B | `0x23` ok; `0x2C`; `0x2D`; `0x2F` | `user_module::build_swap_slots` | 11 |

- Slot ranges: modfx and osc 0–15; delfx and revfx 0–7. Validation details are in
  [user modules](user-modules.md).
- The program-number split is: Rust `pp = n & 0x7F`, `PP = (n >> 7) & 0x7F`, with n in 0–499.
  The spec's annotations conflict with this (see [Errata](#errata--gotchas)). Program numbering is
  owned by [program blob](program-blob.md).

### Dumps and replies

| ID | Spec name | Dir | Payload after function ID | Kind | When / device action | Rust frame |
|----|-----------|-----|---------------------------|------|----------------------|-----------:|
| `0x40` | CURRENT PROGRAM DATA DUMP | ↔ | 1024-byte [program blob](program-blob.md) → 1171 wire bytes | encoded | Sent in reply to `0x10` or on the "Program Dump" utility. On receipt, written to the **edit buffer**, answered `0x23`/`0x24` | 1179 |
| `0x4C` | PROGRAM DATA DUMP (1 PROG) | ↔ | `pp PP` (raw), then 1024-byte blob → 1171 | raw + encoded | Sent in reply to `0x1C` or on "All Dump". On receipt, written to **internal memory** at the program number, answered `0x23`/`0x24` | 1181 |
| `0x51` | GLOBAL DATA DUMP | ↔ | [global blob](global-blob.md); Rust 63 bytes → 72 | encoded | Sent in reply to `0x0E` or on "All Dump". On receipt, saved, answered `0x23`/`0x24` | 80 |
| `0x44` | USER SCALE DATA DUMP | ↔ | spec: `ss` + 384 bytes (TABLE 3); **Rust: 384 → 439 encoded, no `ss`** | spec raw / Rust encoded | Sent in reply to `0x14` or on dump. On receipt: `ss` 0–5 is saved to internal memory; `ss = 0x7F` is saved to the edit buffer (current scale); answered `0x23`/`0x24` | 447 (spec 393) |
| `0x45` | USER OCTAVE DATA DUMP | ↔ | spec: `ss` + 36 bytes (TABLE 4); **Rust: 36 → 42 encoded, no `ss`** | spec raw / Rust encoded | Same as `0x44` | 50 (spec 45) |
| `0x47` | USER API VERSION | XD→H | platform ID (xd = 2), major, minor, patch (each 0–99) | raw | Sent in reply to `0x17` | 12 (no Rust parser) |
| `0x48` | USER MODULE INFO | XD→H | TABLE 5 | encoded | Sent in reply to `0x18`; see [user modules](user-modules.md) | Rust 9 → 11: 19 |
| `0x49` | USER SLOT STATUS | XD→H | TABLE 6, 32 bytes → 37 | encoded | Sent in reply to `0x19` | 45 |
| `0x4A` | USER SLOT DATA | ↔ | TABLE 7, variable size | encoded | Sent in reply to `0x1A`. On receipt, saved to a slot, answered with a status | variable |

### Poly chain and status

| ID | Spec name | Dir | Payload | Kind | When | Rust frame |
|----|-----------|-----|---------|------|------|-----------:|
| `0x60` | POLY CHAIN NOTE ON | ↔ | voice slot, note, velocity, pitch H, M, L | raw | Master transmits; Slave acts on it. See [poly chain](poly-chain.md) | 14 |
| `0x61` | POLY CHAIN NOTE OFF | ↔ | voice slot, mute | raw | Same | 10 |
| `0x23`–`0x2F` | STATUS (ACK/NAK) | XD→H | none | none | Sent after the device receives an exclusive message it acts on ("E" in §1-4) | 8 |

**Transmit conditions [spec]** (§1-4 *4):

| Condition | IDs |
|-----------|-----|
| R (on request) and D (on MIDI-dump page) | `40 4C 51 44 45` |
| R only | `47 48 49 4A` |
| P (when Poly Chain = Master) | `60 61` |
| E (after receiving an exclusive message) | status `23–2F` |

The *All Dump* and *Program Dump* utilities send the same frames unsolicited. A host must therefore
expect dump frames that it did not request.

## Status codes (NOTE 2)

Status frames are 8 bytes, `F0 42 3g 00 01 51 <status> F7`. The status byte sits in the
function-ID position.

| Byte | Spec name | Type | Rust variant | Rust `Display` | `is_ack` | `is_error` |
|------|-----------|------|--------------|----------------|:--------:|:----------:|
| `0x23` | DATA LOAD COMPLETED | ACK | `DataLoadCompleted` | `Data Load Completed` | ✓ | — |
| `0x24` | DATA LOAD ERROR | NAK | `DataLoadError` | `Data Load Error` | — | ✓ |
| `0x25` | (unassigned) | — | none; `from_byte` → `None` | — | — | — |
| `0x26` | DATA FORMAT ERROR | NAK | `DataFormatError` | `Data Format Error` | — | ✓ |
| `0x27` | USER DATA SIZE ERROR | NAK | `UserDataSizeError` | `User Data Size Error` | — | ✓ |
| `0x28` | USER DATA CRC ERROR | NAK | `UserDataCrcError` | `User Data CRC Error` | — | ✓ |
| `0x29` | USER TARGET ERROR | NAK | `UserTargetError` | `User Target Error` | — | ✓ |
| `0x2A` | USER API ERROR | NAK | `UserApiError` | `User API Error` | — | ✓ |
| `0x2B` | USER LOAD SIZE ERROR | NAK | `UserLoadSizeError` | `User Load Size Error` | — | ✓ |
| `0x2C` | USER MODULE ERROR | NAK | `UserModuleError` | `User Module Error` | — | ✓ |
| `0x2D` | USER SLOT ERROR | NAK | `UserSlotError` | `User Slot Error` | — | ✓ |
| `0x2E` | USER FORMAT ERROR | NAK | `UserFormatError` | `User Format Error` | — | ✓ |
| `0x2F` | USER INTERNAL ERROR | NAK | `UserInternalError` | `User Internal Error` | — | ✓ |

- The Rust table matches NOTE 2 exactly [spec][rust].
- `from_byte` returns `None` for every other byte, including `0x00`, `0x25`, `0x30` and `0xFF`.
- `is_error` is defined as `!is_ack`, so it is true for the 11 NAK codes.

**When the device sends them [spec]:**

- `0x24` means busy, a failed request, or a rejected dump.
- `0x2C`, `0x2D` and `0x2F` come from the user-slot commands.

**[hw]** Sending a malformed program blob produces `0x26` DATA FORMAT ERROR. Commit `1c16566`
fixed the byte fidelity of program writes; see [byte fidelity](../design/byte-fidelity.md).

Status codes in Rust: `crates/minilogue-xd/src/sysex/frame.rs:89-184`.

## Generic build and parse (Rust behavior)

### Builders

| Function | Output | Notes |
|----------|--------|-------|
| `build_sysex(channel, function_id, data_8bit)` | header + `encode_7bit(data_8bit)` + `F7` | Always encodes. With empty data it equals `build_sysex_request`. It must not be used for raw-payload messages. `frame.rs:220-231` |
| `build_sysex_request(channel, function_id)` | `F0 42 3g 00 01 51 <fid> F7` (always 8 bytes) | No parameter bytes. `frame.rs:238-249` |
| `build_status(channel, status)` | `build_sysex_request(channel, status.to_byte())` | Device-side message. Used only for tests and mocks. `frame.rs:256-258` |

`channel` is a `U4` (0–15), so `0x30 | channel` cannot overflow into the high nibble. Messages
with raw parameters are hand-assembled by their own modules:

- `0x1C` and `0x4C`: `program/mod.rs:225-262`
- `0x18`–`0x1E`: `user_module.rs:384-480`
- `0x60` and `0x61`: `poly_chain.rs:145-193`

### `parse_sysex(bytes) -> SysexFrame`

Checks run in this order. The first failure returns `SysexError::InvalidHeader(String)`, except
step 8, which returns `Error::Codec` (`frame.rs:269-341`).

1. `len >= 8`.
2. `bytes[0] == 0xF0`.
3. `bytes[len-1] == 0xF7`.
4. `bytes[1] == 0x42`.
5. `bytes[2] & 0xF0 == 0x30`. `channel = bytes[2] & 0x0F`.
6. `bytes[3..6] == 00 01 51`.
7. `function_id = bytes[6]`. This is **not** validated against the known IDs.
8. `data = decode_7bit(bytes[7 .. len-1])`. This fails if any byte has bit 7 set.

Result: `SysexFrame { channel: U4, function_id: u8, data: Vec<u8> }`, where `data` is the
**decoded** payload and is empty for requests and status frames (`frame.rs:194-202`).

Because step 8 always decodes, `parse_sysex` is only correct for the "none" and "encoded"
payload kinds. Raw-payload messages are parsed by their own modules: program dump, poly chain,
identity, MTS. If `parse_sysex` were applied to `0x47`, its 4 raw bytes would be decoded as one
7-bit group and yield 3 garbage bytes.

## Invariants

- `frame.len >= 8`, `frame[0] == 0xF0`, `frame[frame.len - 1] == 0xF7`.
- For every `i` in `1 .. frame.len - 1`: `frame[i] < 0x80`.
- `frame[1] == 0x42`, `frame[2] & 0xF0 == 0x30`, `frame[3..6] == {0x00, 0x01, 0x51}`.
- `channel <= 15`, and `frame[2] == 0x30 | channel`.
- A status frame has `len == 8` and `frame[6]` in `{0x23, 0x24, 0x26 … 0x2F}`.
- `status_is_ack(s) == (s == 0x23)`. For every recognized `s`,
  `status_is_error(s) == !status_is_ack(s)`.
- An encoded payload satisfies `wire_len == encoded_len(blob_len)`, and decoding it gives back
  exactly `blob_len` bytes. Pair this assert across build and parse.
- Each fixed-size function ID has a single wire length. For example, `0x40` → 1179 and
  `0x4C` → 1181 (see table). Assert the length before decoding.
- Request frames built for a request ID never contain a payload byte `>= 0x80`. Assert this on
  every parameter.

## Errata & gotchas

| # | Issue | Provenance | Resolution |
|---|-------|------------|------------|
| 1 | Sections (7)–(10-2) print the header as `F0,42,3g,00,01,44`. `0x44` is not the xd's ID. | [spec] typo | Use `00 01 51`. `0x44` appears to be another Korg device's ID (copy-paste) — unverified. |
| 2 | The binary columns contradict the hex: `14H` is shown as `0001 1000` (= `0x18`), `15H` as `0001 1001` (`0x19`), `44H` as `0100 1000` (`0x48`), `45H` as `0100 1001` (`0x49`). | [spec] typo | The hex column agrees with the function lists in §1-4/§2-5 and with the Rust code. Use the hex. |
| 3 | Section titles and text are mislabelled: (8)'s function row names `0x15` USER **SCALE** DATA DUMP REQUEST; (10-2) is titled USER **SCALE** for `0x45`; (9-1) says "Receive Func=1C" instead of `14`; (22)/(23) describe `60`/`61` as "USER SLOT DATA"; (20) `0x49` is marked T-only but its text says the device receives and saves it. | [spec] copy-paste | Ignore the labels; trust the function IDs. |
| 4 | Program dumps `0x40`/`0x4C` are annotated "384Bytes (7bit) → 336Bytes (8bit)", but TABLE 2 is 1024 bytes. | [spec] vs [hw] | **[hw] wins:** `0x4C` frames from a real device are 1181 bytes, i.e. 1024-byte blobs (commit `d465474`). Details: [program blob](program-blob.md), [errata](errata.md). |
| 5 | Global dump `0x51` is annotated "37Bytes (7bit) → 32Bytes (8bit)", but TABLE 1 spans offsets 0–62 (63 bytes). | [spec] internal conflict | Rust uses 63 bytes (frame 80). Not verified on hardware; see [global blob](global-blob.md). |
| 6 | Program number: request (2) annotates `pp` as "LSB bit 7~0" and `PP` as "MSB bit 8"; dump (5) says "LSB bit 6~0", "MSB bit 7". Neither 7+1 bits split reaches 499. | [spec] inconsistent | Rust uses a 7/7 split (`PP` = 0–3). Reads and writes work on hardware for the programs tried (commit `1c16566`). See [program blob](program-blob.md). |
| 7 | `0x14`/`0x15` take a scale-number byte `ss` in the spec. `0x44`/`0x45` carry `ss` (0–5, or `0x7F` = current/edit buffer) before the data. Rust omits `ss` everywhere and 7-bit-encodes the data. | [spec] vs [rust] | Unverified on hardware. See [tuning](tuning.md). |
| 8 | `0x51` occurs at offset 5 (device ID) and is also the global-dump function ID at offset 6. | coincidence | Match by offset. |
| 9 | `0x25` is unassigned in NOTE 2. | [spec] | Treat it as an unknown status; it is neither ACK nor NAK. |
| 10 | Replies come back on the **KBD/KNOB** USB port, not the SOUND port the request was sent to. | [hw] commit `bc6c452` | See [USB ports](usb-ports.md). |

## Test vectors

All from `crates/minilogue-xd/src/sysex/frame.rs` tests, except the encoded-length rows, which are
computed.

| Case | Bytes | Citation |
|------|-------|----------|
| Global request, ch 0 | `F0 42 30 00 01 51 0E F7` | `frame.rs:461-464` |
| Current-program request, ch 15 | `F0 42 3F 00 01 51 10 F7` | `frame.rs:467-470` |
| Any no-parameter request is 8 bytes; `[6]` = ID; `[7]` = `F7` | for IDs `14 15 17 18 19 1A 1B 1C 1D 1E`, ch 5 | `frame.rs:473-493` |
| ACK, ch 0 | `F0 42 30 00 01 51 23 F7` | `frame.rs:500-503` |
| NAK DataFormatError, ch 3 | `[6] == 0x26` | `frame.rs:506-509` |
| `build_sysex` with empty payload, fid `0x51` | `F0 42 30 00 01 51 51 F7` | `frame.rs:516-520` |
| `build_sysex(ch0, 0x51, [01 02 03])` | `F0 42 30 00 01 51 51 00 01 02 03 F7` | `frame.rs:523-532` (shape asserted; bytes per codec) |
| Round trip of payload `FF 80 00 7F 01 55 AA`, fid `0x40`, ch 3 | decodes back to the same 7 bytes | `frame.rs:548-555` |
| Round trip of payload `00..FF` (256 bytes), fid `0x4C` | decodes back unchanged | `frame.rs:558-563` |
| Status `0x28` round trip, ch 10 | `channel == 10`, `function_id == 0x28`, `data` empty | `frame.rs:566-572` |
| All channels 0–15 round trip | `[2] == 0x30 + c` | `frame.rs:575-581` |
| Too short (7 bytes) | `F0 42 30 00 01 51 F7` → error | `frame.rs:588-591` |
| Empty input | error | `frame.rs:594-596` |
| `[0] = 00`, `[7] = 00`, `[1] = 43`, `[2] = 40`, `[4] = FF` | each → error | `frame.rs:599-632` |
| Status table round trip, all 12 codes | see table above | `frame.rs:356-376` |
| Unknown status bytes | `00 25 30 FF` → `None` | `frame.rs:379-384` |

Frame lengths (computed): `0x40` → 1179, `0x4C` → 1181, `0x51` → 80, Rust `0x44` → 447, Rust
`0x45` → 50, `0x48` → 19, `0x49` → 45.

## Rust design

- **Function IDs** are bare `pub const …: u8` values (`frame.rs:23-77`), not an enum.
  `SysexFrame::function_id` is a raw `u8`, which keeps unknown IDs representable. The Zig port
  can still use an enum with a catch-all.
- **`SysexStatus`** is a `#[non_exhaustive]` enum. `from_byte` returns `Option` and `to_byte`
  maps back. `is_ack`, `is_error` and `Display` are the essential behavior.
- **Owned buffers:**
  - Every builder returns a fresh `Vec<u8>`.
  - `parse_sysex` borrows `&[u8]` and returns an owned, decoded `Vec<u8>`.
  - The allocation is incidental: lengths are fixed per function ID.
- **Error strings:**
  - `InvalidHeader(String)` formats a message per failure (`error.rs:8-9`). The *distinct cases*
    are essential; the strings are incidental.
  - `WrongFunctionId { expected, found }` is raised by typed parsers such as
    `parse_current_program_dump` (`program/mod.rs:210-220`), not by `parse_sysex`.
- **Raw-payload messages bypass `parse_sysex`.** `parse_program_dump` re-checks the header by
  hand, because it must read `pp PP` before decoding (`program/mod.rs:272-309`). Poly chain has
  its own `validate_header` (`poly_chain.rs:294-325`). This duplication is incidental. A Zig port
  should split *header validation* from *payload decoding* so that every message type shares one
  header check.
- **No `0x47` parser and no host→device `0x4A` builder** exist (`user_module.rs:375-539`).

## Zig / TigerStyle notes

- **Comptime descriptor table** (sketch):

  ```zig
  // sketch
  const PayloadKind = enum(u8) { none, raw, encoded, raw_then_encoded };
  const FunctionId = enum(u8) {
      global_request = 0x0E, current_program_request = 0x10,
      user_scale_request = 0x14, user_octave_request = 0x15,
      user_api_version_request = 0x17, user_module_info_request = 0x18,
      user_slot_status_request = 0x19, user_slot_data_request = 0x1A,
      user_slot_clear = 0x1B, program_request = 0x1C,
      user_module_clear = 0x1D, user_slot_swap = 0x1E,
      current_program_dump = 0x40, user_scale_dump = 0x44, user_octave_dump = 0x45,
      user_api_version = 0x47, user_module_info = 0x48, user_slot_status = 0x49,
      user_slot_data = 0x4A, program_dump = 0x4C, global_dump = 0x51,
      poly_chain_note_on = 0x60, poly_chain_note_off = 0x61,
      _,
  };
  const Status = enum(u8) {
      data_load_completed = 0x23, data_load_error = 0x24, data_format_error = 0x26,
      user_data_size_error = 0x27, user_data_crc_error = 0x28, user_target_error = 0x29,
      user_api_error = 0x2A, user_load_size_error = 0x2B, user_module_error = 0x2C,
      user_slot_error = 0x2D, user_format_error = 0x2E, user_internal_error = 0x2F,
  };
  ```

  Pair it with a `comptime` array of
  `{ id, kind, raw_prefix_len, blob_len, frame_len_max, reply_ids }`. Then
  `comptime assert(frame_len(.program_dump) == 1181)` and similar checks hold for every
  fixed-size ID.
- **Caller-provided buffers, no `Vec`:**
  - `frame_write(target: []u8, options: FrameOptions, payload: []const u8) u16` asserts
    `target.len >= frame_len` and returns the bytes written.
  - Parsing returns a *view*: `FrameView { channel: u4, function_id: u8, payload_wire: []const u8 }`.
    Decoding is a separate step into a caller buffer of the expected blob length.
  - This separation fixes the Rust "decode everything" pitfall.
- **Error set:** `FrameError` from [errors](../design/errors.md): `error{ FrameTooShort,
  StartByteMissing, EndByteMissing, ManufacturerMismatch, ChannelByteInvalid, DeviceIdMismatch,
  FunctionIdMismatch, FrameLengthMismatch } || CodecError`. A data byte with bit 7 set is
  `WireByteInvalid`. Each Rust `InvalidHeader` case becomes its own tag.
- **Integer widths:** `channel: u4`, function ID `u8` (wire values `< 0x80`), lengths `u16`
  (max frame 1181 < 65536). Parameter bytes are `u7`. Convert to `u8` only when writing to the
  wire.
- **Maximum frame sizes** for static buffers:

  | Frame | Max bytes |
  |-------|----------:|
  | Program dump `0x4C` (largest fixed-size frame) | 1181 |
  | Current program `0x40` | 1179 |
  | Rust user scale `0x44` | 447 |
  | MTS bulk ([tuning](tuning.md)) | 408 |
  | Global `0x51` | 80 |

  A 1184-byte (or 2048-byte, for margin) receive buffer covers everything except user slot data
  `0x4A`. Size that one from [user modules](user-modules.md) and the
  [memory budget](../zig/memory-budget.md).
- **Assertions:**
  - Before encode, assert the payload length matches the table.
  - After decode, assert the decoded length equals `blob_len`.
  - On every write, assert `frame[2] & 0xF0 == 0x30`.

## Open questions

- Does the device ignore Korg exclusive messages whose `g` differs from its global channel, or
  answer them anyway? This is not tested on hardware. The examples only use `g = 0`
  (`examples/read_patch.rs:25-43`).
- Does the device reply `0x24`/`0x26` to an unknown function ID, or stay silent? A transaction
  layer must time out in the silent case.
- Does `0x44`/`0x45` carry the `ss` byte, and is its payload raw or NOTE-1 encoded? See
  [tuning](tuning.md).
- Does the device send status `0x23` after *every* accepted exclusive message ("E"), including
  requests that it answers with a dump?

## Sources

- Korg MIDI Impl rev 1.01 §1-4 (transmitted function IDs, *4), §2-5 (received function IDs),
  §Exclusive format (1)–(24), NOTE 1, NOTE 2, NOTE 3, NOTE 4.
- `crates/minilogue-xd/src/sysex/mod.rs:18-22` — `KORG_ID`, `DEVICE_ID`.
- `crates/minilogue-xd/src/sysex/frame.rs:1-9` — frame format; `23-77` — function-ID constants;
  `89-184` — `SysexStatus`; `194-207` — `SysexFrame`, `MIN_FRAME_LEN`; `220-258` — builders;
  `269-341` — `parse_sysex`; `356-668` — tests.
- `crates/minilogue-xd/src/codec.rs:29-50`, `69-103` — encode and decode (high-bit rejection).
- `crates/minilogue-xd/src/sysex/program/mod.rs:192-309` — `0x10`/`0x40`/`0x1C`/`0x4C`
  builders and parsers.
- `crates/minilogue-xd/src/sysex/global.rs:138`, `322-357` — global blob size and frames.
- `crates/minilogue-xd/src/sysex/tuning.rs:212-271` — `0x14`/`0x15`/`0x44`/`0x45`.
- `crates/minilogue-xd/src/sysex/user_module.rs:375-539` — user-module requests and replies.
- `crates/minilogue-xd/src/sysex/poly_chain.rs:145-193` — `0x60`/`0x61` builders.
- `crates/minilogue-xd/src/error.rs:6-88` — `SysexError`.
- `docs/wayfinder/research/xd-midi-capabilities.md:48-58` — earlier list of spec/code
  discrepancies.
- Git commits `bc6c452`, `d465474`, `1c16566` — [hw] evidence (see
  [SysEx transactions](../design/sysex-transactions.md)).
