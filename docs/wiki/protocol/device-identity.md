---
title: Device identity — Universal Identity and Korg Search Device
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/identity.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - docs/korg-user-guide/book.md
related:
  - protocol/sysex-framing.md
  - protocol/usb-ports.md
  - protocol/errata.md
  - design/sysex-transactions.md
  - design/typed-values.md
  - design/transport.md
updated: 2026-10-11
---

# Device identity — Universal Identity and Korg Search Device

> The minilogue xd answers two discovery messages that do **not** use the Korg exclusive header
> (see [SysEx framing](sysex-framing.md)):
> - The MIDI-standard **Universal Non-Realtime Identity Request** `F0 7E nn 06 01 F7`.
> - Korg's own **Search Device Request** `F0 42 50 00 dd F7`.
>
> Both replies are 15 bytes. They carry the family ID `51 01`, member ID `00 00` and four firmware
> version bytes, and both report the global channel.
>
> The Rust crate builds and parses both messages, but it disagrees with Korg spec rev 1.01 in
> four places:
> - It cannot address "any channel" (`7F`).
> - It labels the version bytes wrongly.
> - It swaps the channel and echo-ID bytes in the search reply.
> - It does not check the manufacturer in the identity reply.
>
> None of these messages has been exercised on hardware.

## Identity Request (host → device)

| Offset | Value | Meaning |
|-------:|-------|---------|
| 0 | `0xF0` | SysEx start |
| 1 | `0x7E` | Universal Non-Realtime |
| 2 | `nn` | Target: `0x00`–`0x0F` = the device's global channel; `0x7F` = any channel [spec] |
| 3 | `0x06` | General Information |
| 4 | `0x01` | Identity Request |
| 5 | `0xF7` | EOX |

Length: 6 bytes. Rust: `build_identity_request(channel: U4)`
(`crates/minilogue-xd/src/sysex/identity.rs:20-22`). `U4` makes `0x7F` unrepresentable; see
[Errata](#errata--gotchas).

The owner's-manual MIDI chart says "In addition to Korg exclusive messages, Inquiry is supported"
(`docs/korg-user-guide/book.md:2719`).

## Identity Reply (device → host)

| Offset | Value | Spec meaning | Rust field |
|-------:|-------|--------------|------------|
| 0 | `0xF0` | SysEx start | checked |
| 1 | `0x7E` | Universal Non-Realtime | checked |
| 2 | `0g` | Global channel `g` (0–15) | `channel: U4` (fails if `> 15`) |
| 3 | `0x06` | General Information | checked |
| 4 | `0x02` | Identity Reply | checked |
| 5 | `0x42` | Manufacturer ID (Korg) | `manufacturer_id` — **not checked** |
| 6 | `0x51` | Family ID, LSB | `family_id = b6 \| b7 << 8` → `0x0151` |
| 7 | `0x01` | Family ID, MSB | |
| 8 | `0x00` | Member ID, LSB | `member_id = b8 \| b9 << 8` → `0x0000` |
| 9 | `0x00` | Member ID, MSB | |
| 10 | `v0` | **Minor version, LSB** | `version[0]` (Rust doc says "major") |
| 11 | `v1` | **Minor version, MSB** | `version[1]` (Rust doc says "minor") |
| 12 | `v2` | **Major version, LSB** | `version[2]` (Rust doc says "micro") |
| 13 | `v3` | **Major version, MSB** | `version[3]` (Rust doc says "build") |
| 14 | `0xF7` | EOX | checked (as the *last* byte) |

The spec says it is "transmitted whenever an INQUIRY MESSAGE REQUEST is received" (§1-3).

### Version decoding

**[spec]** The bytes are in the order minor-LSB, minor-MSB, major-LSB, major-MSB. Every byte is
7-bit. The spec does not say whether "LSB/MSB" means 7-bit halves or 8-bit halves.

- **Recommended:** `minor = v0 | (v1 << 7)`, `major = v2 | (v3 << 7)`. MIDI data bytes carry 7 bits.
  Every realistic version number is below 128, so `v1 = v3 = 0`, and both interpretations agree.
- Firmware "2.10" would then be expected as `0A 00 02 00`. This is derived from the spec and has
  **not** been captured from hardware.
- **Keep the 4 raw bytes too.** The field naming is unverified.

The Rust code combines family and member IDs with `<< 8`. This yields the conventional
`0x0151`, which matches how Korg documents the family. With `<< 7`, the 7-bit interpretation would
give `0xD1`. For identity matching, compare the **raw bytes** `51 01 00 00`, not a combined
integer.

## Search Device Request (host → device)

| Offset | Value | Meaning |
|-------:|-------|---------|
| 0 | `0xF0` | SysEx start |
| 1 | `0x42` | Korg |
| 2 | `0x50` | Search Device |
| 3 | `0x00` | Request |
| 4 | `dd` | Echo-back ID (0–127), copied into the reply |
| 5 | `0xF7` | EOX |

Length: 6 bytes. Rust: `build_search_device(echo_id: u8)`
(`identity.rs:116-118`). **`echo_id` is not masked or validated.** Passing `>= 0x80` emits an
illegal data byte, which would terminate the SysEx on a real MIDI parser.

## Search Device Reply (device → host)

| Offset | Spec value | Spec meaning | Rust reads it as |
|-------:|------------|--------------|------------------|
| 0 | `0xF0` | SysEx start | checked |
| 1 | `0x42` | Korg | checked |
| 2 | `0x50` | Search Device | checked |
| 3 | `0x01` | Reply | checked |
| 4 | `0g` | **Global channel** (0–15) | **`echo_id`** |
| 5 | `dd` | **Echo-back ID** | **`channel: U4`** (fails if `> 15`) |
| 6 | `0x51` | Family ID, LSB | `family_id` (LE, `<< 8`) |
| 7 | `0x01` | Family ID, MSB | |
| 8 | `0x00` | Member ID, LSB | `member_id` |
| 9 | `0x00` | Member ID, MSB | |
| 10–13 | `v0 v1 v2 v3` | Minor LSB, minor MSB, major LSB, major MSB | `version[0..4]` |
| 14 | `0xF7` | EOX | checked (as the last byte) |

**[spec] wins** for a re-implementation: channel at offset 4, echo at offset 5 (§1-5).

The Rust order has no hardware evidence behind it. With the Rust parser, a spec-shaped reply
behaves as follows:

- Echo ID `> 15`: fails with `InvalidHeader("channel byte out of range")`.
- Echo ID `<= 15`: channel and echo are silently swapped.

## Validation order (Rust)

`parse_identity_reply` (`identity.rs:50-104`):

1. `len >= 15`, else `PayloadTooShort { expected: 15, actual }`.
2. `[0] == F0`.
3. `[len-1] == F7`.
4. `[1] == 7E`.
5. `[3] == 06 && [4] == 02`.
6. `U4::new([2])`.

Steps 2–6 fail with `InvalidHeader`. Longer inputs are accepted; bytes 14 … len−2 are ignored.

`parse_search_device_reply` (`identity.rs:146-208`):

1. `len >= 15`.
2. `[0] == F0`.
3. `[len-1] == F7`.
4. `[1] == 42`.
5. `[2] == 50`.
6. `[3] == 01`.
7. echo = `[4]`; `U4::new([5])`.

Neither parser checks the family or member ID. Neither rejects data bytes `>= 0x80`.

## Use in transactions

`SysexTransaction::query_identity`:

1. Sends `build_identity_request(self.channel)`.
2. Receives one SysEx frame via `receive_sysex`.
3. Parses it with `parse_identity_reply`. It bypasses `parse_sysex`, because the reply is not a
   Korg-header frame (`transaction.rs:353-358`).

There is no transaction helper for Search Device. Consequences:

- If the host's channel is wrong, the device ignores the request and the call times out after the
  transaction timeout (default 5 s). This is because `0x7F` cannot be sent.
- Any other SysEx frame that arrives first, such as a dump, fails parsing. The call does not skip
  it.

See [SysEx transactions](../design/sysex-transactions.md).

## Invariants

- Identity request: `len == 6`, `[1] == 0x7E`, `[3..5] == {0x06, 0x01}`, and `[2] <= 0x0F or [2] == 0x7F`.
- Identity reply: `len == 15`, `[0] == 0xF0`, `[14] == 0xF7`, `[1] == 0x7E`,
  `[3..5] == {0x06, 0x02}`, `[2] <= 0x0F`.
- For the minilogue xd: `[5] == 0x42`, `[6..10] == {0x51, 0x01, 0x00, 0x00}`.
- Search request: `len == 6`, `[1..4] == {0x42, 0x50, 0x00}`, `[4] < 0x80`.
- Search reply: `len == 15`, `[1..4] == {0x42, 0x50, 0x01}`, `[4] <= 0x0F` (channel),
  `[5] == echo sent` (the echo ID).
- Every byte strictly between `F0` and `F7` is `< 0x80`.

## Errata & gotchas

| # | Issue | Provenance | Resolution for the port |
|---|-------|------------|-------------------------|
| 1 | Version bytes are documented in Rust as `[major, minor, micro, build]` (`identity.rs:38`, `134`). The spec says minor-LSB, minor-MSB, major-LSB, major-MSB. | [spec] vs [rust] doc bug | Follow the spec; keep the raw bytes. |
| 2 | The search reply has channel at 4 and echo at 5 in the spec; Rust reads echo from 4 and channel from 5 (`identity.rs:191-195`). | [spec] vs [rust] | Follow the spec. The Rust tests encode the swapped order (`identity.rs:329-345`). |
| 3 | The identity request cannot be sent to `7F` (any channel) because the argument is `U4` (`identity.rs:17-22`). | [rust] limitation | Accept `u7` and allow `0x7F`, or offer an options struct `{ target: .any \| .channel(u4) }`. Use `.any` for discovery. |
| 4 | `parse_identity_reply` accepts any manufacturer and any family. | [rust] | Check `42 51 01` to avoid accepting another device's reply on a shared bus. |
| 5 | `build_search_device` does not mask `echo_id`. | [rust] | Take `echo_id: u7`. |
| 6 | Both parsers accept frames longer than 15 bytes. | [rust] | Require `len == 15`. |
| 7 | The reply's channel `g` is the global channel, which also forms byte 2 (`0x30 \| g`) of every Korg exclusive frame ([SysEx framing](sysex-framing.md)). | [spec] | Use the reply to learn `g` before sending Korg exclusive requests. |

## Test vectors

| Case | Bytes | Expected | Citation |
|------|-------|----------|----------|
| Identity request, ch 0 | `F0 7E 00 06 01 F7` | — | `identity.rs:223-226` |
| Identity request, ch 15 | `F0 7E 0F 06 01 F7` | — | `identity.rs:229-232` |
| Identity reply | `F0 7E 00 06 02 42 51 01 00 00 02 01 00 00 F7` | ch 0, mfr `0x42`, family `0x0151`, member 0, version raw `02 01 00 00`. Per spec this is minor 130 (7-bit halves) or 258 (8-bit halves) and major 0. It is a synthetic vector. | `identity.rs:239-254` |
| Identity reply, ch 5 | `F0 7E 05 06 02 42 51 01 00 00 01 00 00 00 F7` | ch 5 | `identity.rs:257-264` |
| Too short | `F0 7E 00 06 02 42 F7` | error | `identity.rs:267-270` |
| `[0] = 00` / last byte `00` | 15-byte variants | error | `identity.rs:273-288` |
| Sub-ID `06 01` (request, not reply) | `F0 7E 00 06 01 42 51 01 00 00 01 00 00 00 F7` | error | `identity.rs:291-297` |
| `7F` instead of `7E` | `F0 7F 00 06 02 …` | error | `identity.rs:300-306` |
| Search request, echo 0 | `F0 42 50 00 00 F7` | — | `identity.rs:313-316` |
| Search request, echo `0x42` | `F0 42 50 00 42 F7` | — | `identity.rs:319-322` |
| Search reply (**Rust order**) | `F0 42 50 01 7F 00 51 01 00 00 02 01 00 00 F7` | Rust: echo `0x7F`, ch 0. **Per spec this is ch `0x7F` (invalid), echo 0** | `identity.rs:329-345` |
| Search reply, Rust ch 10 | `F0 42 50 01 00 0A 51 01 00 00 01 00 00 00 F7` | Rust: ch 10 (spec: ch 0, echo 10) | `identity.rs:348-355` |
| Search reply, byte 5 = `0x10` | `F0 42 50 01 00 10 51 01 …` | Rust error (spec: valid, echo 16) | `identity.rs:445-451` |
| Search reply, bad bytes | `[1]=43`, `[2]=51`, `[3]=00`, too short `F0 42 50 01 00 F7` | error | `identity.rs:358-406` |
| Transaction identity | queue the reply above → `manufacturer_id 0x42`, `family_id 0x0151` | — | `transaction.rs:492-512` |

**Spec-correct vectors for a port.** These are derived from the spec, not captured from hardware.
They assume global channel 0, echo `0x55`, firmware 2.10:

- Identity reply: `F0 7E 00 06 02 42 51 01 00 00 0A 00 02 00 F7`.
- Search request: `F0 42 50 00 55 F7`.
- Search reply: `F0 42 50 01 00 55 51 01 00 00 0A 00 02 00 F7`.

## Rust design

- Free functions return `Vec<u8>` (6 bytes) and owned structs `IdentityReply` and
  `SearchDeviceReply` with `family_id: u16`, `member_id: u16` and `version: [u8; 4]`
  (`identity.rs:29-40`, `125-136`). The struct shapes are fine. The `Vec` is incidental.
- The errors reuse `SysexError::PayloadTooShort` and `InvalidHeader(String)`.
- Channel typing with `U4` is a good idea for the reply but a mistake for the request target
  (see [typed values](../design/typed-values.md)).

## Zig / TigerStyle notes

- Both requests are 6-byte constants plus one variable byte. Build them into `*[6]u8`; do not
  allocate.
- Parse into `struct { channel: u4, family: [2]u7, member: [2]u7, version: [4]u7 }`. Use
  `comptime assert(identity_reply_len == 15)`.
- `const IdentityTarget = union(enum) { any, channel: u4 };`. Encode `.any` as `0x7F`.
- Error set: `error{ ReplyLengthInvalid, StartByteMissing, EndByteMissing, NotIdentityReply,
  ManufacturerMismatch, FamilyMismatch, ChannelInvalid, EchoMismatch }`.
- **Discovery flow** (bounded):
  1. Send the identity request to `0x7F`.
  2. Poll the assembler (see [SysEx transactions](../design/sysex-transactions.md)) until
     `deadline_ns`.
  3. Accept the first frame that parses as an identity reply with `42 51 01 00 00`.
  4. Discard other frames, up to a bounded count.
  5. Store `g`.
- When matching a search reply, assert that `echo_id_received == echo_id_sent`.

## Open questions

- Capture real identity and search replies from hardware. This would confirm the version-byte
  meaning (e.g. firmware 2.10), the family and member bytes, and the search-reply byte order.
- Does the device answer an identity request on any port other than KBD/KNOB (see
  [USB ports](usb-ports.md))? Does it answer on 5-pin MIDI OUT?
- Does the device answer `nn` ≠ global channel at all (spec: no; only `g` or `7F`)?

## Sources

- Korg MIDI Impl rev 1.01 §1-3 (Device Inquiry Reply), §1-5 (Search Device Reply), §2-4 (Device
  Inquiry Request), §2-6 (Search Device Request).
- `crates/minilogue-xd/src/sysex/identity.rs:14-43` — request builder and reply struct;
  `50-104` — identity parser; `110-139` — search builder and struct; `146-208` — search parser;
  `223-451` — tests.
- `crates/minilogue-xd/src/sysex/transaction.rs:353-358` — `query_identity`; `492-512` — test.
- `docs/korg-user-guide/book.md:2719` — "Inquiry is supported".
