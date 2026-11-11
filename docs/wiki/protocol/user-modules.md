---
title: logue SDK user modules (TABLES 5–7, functions 0x17–0x1E / 0x47–0x4A)
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/user_module.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/Cargo.toml
  - crates/minilogue-xd/src/lib.rs
  - README.md
  - docs/index.html
  - docs/dev/0003-phase-3-implementation-plan-sysex-layer.md
  - https://github.com/korginc/logue-sdk
  - https://github.com/irgendwatt/logue-manager
related:
  - protocol/sysex-framing.md
  - protocol/korg-7bit-codec.md
  - protocol/errata.md
  - protocol/file-formats.md
  - protocol/device-identity.md
  - protocol/usb-ports.md
  - design/sysex-transactions.md
  - design/errors.md
  - zig/memory-budget.md
  - zig/dependency-elimination.md
  - sources/community-references.md
  - sources/korg-midi-implementation.md
updated: 2026-10-11
---

# logue SDK user modules (TABLES 5–7, functions 0x17–0x1E / 0x47–0x4A)

> The XD runs user-loadable logue SDK programs in four module types: oscillator, mod FX, delay
> FX and reverb FX. Each type has a fixed number of slots. Korg SysEx functions `0x17`–`0x1E`
> query, download, clear and swap slots, and `0x4A` also uploads one. The Rust crate covers part
> of this: request builders, 9/32/8+N-byte blob types, IEEE CRC-32 checking, and reply parsers.
> It has **no upload builder, no API-version reply parser, no transaction methods and no
> `.mnlgxdunit` support**, despite what the README says. Community captures also show that real
> replies carry raw module/slot bytes before the 7-bit data. The Rust reply parsers do not
> expect those bytes, and the Rust TABLE 6 layout disagrees with both the spec and the logue
> SDK header. Treat this page's [community] layouts as the better guess for a re-implementation.

## Module IDs and slots

| Module | ID byte | Slots (slot IDs) | Rust variant (display) | Max load size, logue SDK xd linker script **[community]** |
|---|---:|---|---|---:|
| Mod FX | `0x01` | 16 (0–15) | `ModFx` ("Mod FX") | 6 KiB (6144) |
| Delay FX | `0x02` | 8 (0–7) | `DelayFx` ("Delay FX") | 12 KiB (12288) |
| Reverb FX | `0x03` | 8 (0–7) | `ReverbFx` ("Reverb FX") | 12 KiB (12288) |
| Oscillator | `0x04` | 16 (0–15) | `Osc` ("Oscillator") | 32 KiB (32768) |

The IDs and slot ranges are from §3(12)–(17) **[spec]**. Rust rejects any other module byte with
`InvalidModuleId(b)`. That includes `0` and `5`, although TABLE 6 lists MODULE ID range 1~5
**[spec]**. In logue SDK `userprg.h`, ID `0` is "global" and there is no ID `5` for the xd
**[community]**. Platform IDs from the same header: prologue `1`, **minilogue xd `2`**, NTS-1
`3` **[community]**. §3(18) agrees for the xd (platform `2`) **[spec]**.

## Messages

Header `F0 42 3g 00 01 51` and the 7-bit codec are on [SysEx framing](sysex-framing.md) and
[Korg 7-bit codec](korg-7bit-codec.md). In requests, `mm` (module) and `ss` (slot) are **raw**
7-bit data bytes, not codec groups. Every value is < 0x80, so no MSB byte is needed.

### Host → device

| Func | Name | Wire bytes | Len | Device reply **[spec]** | Rust builder |
|---|---|---|---:|---|---|
| `0x17` | USER API VERSION REQUEST | `F0 42 3g 00 01 51 17 F7` | 8 | `0x47`, or `0x24` if busy | `build_api_version_request` |
| `0x18` | USER MODULE INFO REQUEST | `F0 42 3g 00 01 51 18 mm F7` | 9 | `0x48`, or `0x24` | `build_module_info_request` |
| `0x19` | USER SLOT STATUS REQUEST | `F0 42 3g 00 01 51 19 mm ss F7` | 10 | `0x49`, or `0x24` | `build_slot_status_request` ✔slot |
| `0x1A` | USER SLOT DATA REQUEST | `F0 42 3g 00 01 51 1A mm ss F7` | 10 | `0x4A`, or `0x24` | `build_slot_data_request` ✔slot |
| `0x1B` | CLEAR USER SLOT | `F0 42 3g 00 01 51 1B mm ss F7` | 10 | `0x23` ok; `0x2C` bad module; `0x2D` bad slot; `0x2F` internal | `build_clear_slot` ✔slot |
| `0x1D` | CLEAR USER MODULE (all slots) | `F0 42 3g 00 01 51 1D mm F7` | 9 | `0x23`; `0x2C`; `0x2F` | `build_clear_module` |
| `0x1E` | SWAP USER DATA | `F0 42 3g 00 01 51 1E mm sa sb F7` | 11 | `0x23`; `0x2C`; `0x2D`; `0x2F` | `build_swap_slots` ✔both slots |
| `0x4A` | USER SLOT DATA (upload) | `F0 42 3g 00 01 51 4A mm ss <7-bit TABLE 7> F7` **[community]** | var | `0x23`, or `0x24` **[spec]**; probably also `0x27`–`0x2B`, `0x2E` | **not implemented** |

✔slot = Rust checks `slot < max_slots(module)` and otherwise returns
`InvalidSlotIndex { slot, max }`. `0x1C` is not a user-module function (it is PROGRAM DATA DUMP
REQUEST). The NOTE 2 status codes reserved for user data are listed below. The spec does not say
which message triggers each one. By their names they are upload outcomes (inferred):

| Status | Name **[spec]** | Likely cause (inferred) |
|---|---|---|
| `0x27` | USER DATA SIZE ERROR | TABLE 7 size field inconsistent or too large |
| `0x28` | USER DATA CRC ERROR | CRC-32 mismatch |
| `0x29` | USER TARGET ERROR | header platform/module ≠ this device/module |
| `0x2A` | USER API ERROR | header API version unsupported |
| `0x2B` | USER LOAD SIZE ERROR | load size > slot RAM |
| `0x2C` | USER MODULE ERROR | module ID invalid |
| `0x2D` | USER SLOT ERROR | slot ID invalid |
| `0x2E` | USER FORMAT ERROR | malformed payload |
| `0x2F` | USER INTERNAL ERROR | unspecified failure |

### Device → host

| Func | Name | Spec body **[spec]** | Observed body **[community]** | Rust parser |
|---|---|---|---|---|
| `0x47` | USER API VERSION | 4 **raw** bytes: platform (xd = 2), major, minor, patch (each 0–99) | xd: `F0 42 3B 00 01 51 47 02 01 01 00 F7` → platform 2, API 1.1.0 | **none** |
| `0x48` | USER MODULE INFO | 7-bit, "11 → 9 bytes", TABLE 5 | NTS-1: `48 mm 00` + 7-bit data that decodes to **12** bytes | `parse_module_info_reply` (decodes everything after func) |
| `0x49` | USER SLOT STATUS | 7-bit, "37 → 32 bytes", TABLE 6 | NTS-1: `49 mm ss 00` + 7-bit TABLE 6; empty slot = `49 mm ss F7` | `parse_slot_status_reply` (decodes everything after func) |
| `0x4A` | USER SLOT DATA | 7-bit, variable, TABLE 7 | `4A mm ss` + 7-bit TABLE 7 | `parse_slot_data_reply` (decodes everything after func) |

The spec diagrams for `0x48`–`0x4A` show only "Data1, Data2, …" after the function byte. They
do not show the raw module/slot prefix that the community captures contain. Section §3(20)
(`0x49`) is marked transmit-only but its prose says the device also receives and saves it. That
prose looks copied from §3(21).

## TABLE 5 — user module info

| Off | Spec **[spec]** | Rust `UserModuleInfo` **[rust]** | NTS-1 capture **[community]** |
|---|---|---|---|
| 0–3 | MAX SLOT SIZE, 0~2^32−1 | `max_slot_size` u32 LE | u32 LE max payload size |
| 4–7 | MAX PROGRAM SIZE, 0~2^32−1 | `max_program_size` u32 LE | u32 LE max load size |
| 8 (spec: "8~9") | AVAILABLE SLOT COUNT, 0~255 | `available_slot_count` u8; `BLOB_SIZE = 9` | u32 LE slot count (bytes 8–11) |

The spec says "8~9" for the slot count, yet the §3(19) diagram converts 11 → 9 bytes. Rust picks
9 and reads only byte 8. Bytes past 9 are ignored, so a 12-byte decode still yields the right
low byte of the count. NTS-1 example values **[community]**:

| Module | max slot size | max program size | slots |
|---|---:|---:|---:|
| modfx | 8180 | 6144 | 16 |
| delfx / revfx | 16368 | 12288 | 8 |
| osc | 36848 | 32768 | 16 |

Each "slot size" exceeds its "program size" by 2036 or 4080 bytes. The xd values are not
captured.

## TABLE 6 — user slot status (32 bytes)

Three sources disagree. The [community] column is the first 32 bytes of the packed logue SDK
`user_prg_header_t`, and it matches decoded NTS-1 replies.

| Off | Spec **[spec]** | logue SDK / capture **[community]** | Rust `UserSlotStatus` **[rust]** |
|---|---|---|---|
| 0 | PLATFORM ID (range "1") | `target` u16 LE low byte = **module** | `platform_id` u8 |
| 1 | MODULE ID 1~5 | `target` high byte = **platform** | `module_id` (1–4 else error) |
| 2 | API VERSION MAJOR (2~3) | `api` u32 LE byte 0 = patch | `api_version.0` (major) |
| 3 | 〃 | `api` byte 1 = minor | `api_version.1` (minor) |
| 4 | API VERSION MINOR | `api` byte 2 = major | `api_version.2` (patch) |
| 5 | API VERSION PATCH | `api` byte 3 = 0 | `developer_id` u32 LE, bytes 5–8 |
| 6–9 | DEVELOPER ID | `dev_id` u32 LE | (5–8) |
| 9 | 〃 | 〃 | `program_id` u32 LE, bytes 9–12 |
| 10–13 | PROGRAM ID | `prg_id` u32 LE | (9–12) |
| 13 | 〃 | 〃 | `program_version.0` (major) |
| 14 | PROGRAM VERSION MAJOR (14~15) | `version` u32 LE byte 0 = patch | `program_version.1` (minor) |
| 15 | 〃 | `version` byte 1 = minor | `program_version.2` (patch) |
| 16 | PROGRAM VERSION MINOR | `version` byte 2 = major | name byte 0 |
| 17 | PROGRAM VERSION PATCH | `version` byte 3 = 0 | name byte 1 |
| 18–31 | PROGRAM NAME, ASCII (14 bytes) | `name[14]`, NUL-terminated (13 chars max) | name bytes 2–15 (16-byte field) |

Reading: the spec and the SDK agree on **field boundaries**: 2-byte target, then 4-byte api,
dev_id, prg_id and version, then a 14-byte name. The spec lists each multi-byte field from its
most significant byte down. The SDK stores them little-endian, as `target = platform << 8 |
module` and `api = major << 16 | minor << 8 | patch`. Rust uses 1-byte versions and a 16-byte
name, so every field from offset 5 on is **one byte early**. Rust round-trips its own bytes, but
it would misread a real device reply. Suspected bug (see Open questions).

Rust name handling: it reads bytes 16..32 up to the first `0x00` (or all 16) and converts them
with UTF-8-lossy. To write, it copies up to 16 **bytes** of the UTF-8 string and zero-fills the
rest. A 16-byte name has no terminator, and a multi-byte character can be split at byte 16.

## TABLE 7 — user slot data (variable)

| Off | Field | Encoding |
|---|---|---|
| 0–3 | PAYLOAD SIZE | u32 LE = number of bytes that follow the 8-byte header |
| 4–7 | PAYLOAD CRC32 | u32 LE, CRC-32/IEEE over exactly bytes `8 .. 8+size` |
| 8 … 8+size−1 | PAYLOAD DATA | opaque to TABLE 7 |

This level is consistent across the spec, Rust and the community captures. CRC verification
over `decoded[8 .. 8+size]` is confirmed on NTS-1 hardware **[community]**.

**Inner payload layout for an upload** (logue-manager `build_upload_payload`, logue SDK
`user_prg_header_t`) **[community]**:

| Payload off | Bytes | Content |
|---|---:|---|
| 0 | 32 | TABLE 6 bytes (target, api, dev_id, prg_id, version, name[14]) |
| 32 | 988 | rest of the 1024-byte `user_prg_header_t` (SDK: `num_param` u32, param descriptors, padding). logue-manager zero-fills it. |
| 1020 | 4 | `load_size` u32 LE = length of the program binary |
| 1024 | load_size | program binary (`payload.bin` from a `.mnlgxdunit`) |
| 1024 + load_size | 132 | signature area (`USER_PRG_SIG_SIZE = 0x84`), zero-filled by logue-manager |

So `size = load_size + 1156` and the CRC covers the header, the binary and the signature area.
The SDK constants are `USER_PRG_HEADER_SIZE = 0x400` and `USER_PRG_SIG_SIZE = 0x84`. Rust treats
the whole payload as opaque `Vec<u8>` and builds none of this.

### Parse algorithm (Rust `UserSlotData::from_bytes`)

```text
parse(data):
  if len(data) < 8:              error PayloadTooShort{8, len}
  size = u32_le(data[0..4]); crc = u32_le(data[4..8])
  if len(data) < 8 + size:       error PayloadTooShort{8+size, len}
  payload = data[8 .. 8+size]                  # trailing bytes ignored
  if crc32_ieee(payload) != crc: error Crc32Mismatch{expected crc, actual computed}
```

`from_payload(p)` sets `size = len(p)` and `crc = crc32_ieee(p)`. `to_bytes` writes the header
and the payload back verbatim. It does not recompute the CRC, so a corrupted struct serializes
as-is.

## CRC-32 parameters

| Parameter | Value |
|---|---|
| Name | CRC-32/ISO-HDLC ("IEEE", zlib, PNG) |
| Polynomial | `0x04C11DB7` normal, `0xEDB88320` reflected |
| Init | `0xFFFFFFFF` |
| RefIn / RefOut | true / true (process LSB first) |
| XorOut | `0xFFFFFFFF` |
| Check (`"123456789"`) | `0xCBF43926` |
| Stored | u32 little-endian at TABLE 7 offset 4 |

```text
crc32(bytes):
  c = 0xFFFFFFFF
  for b in bytes: c = table[(c ^ b) & 0xFF] ^ (c >> 8)
  return c ^ 0xFFFFFFFF
table[i] = 8 rounds of: c = (c & 1) ? (c >> 1) ^ 0xEDB88320 : c >> 1, starting c = i
```

## Implementation status (Rust) — stated plainly

| Capability | Status | Evidence |
|---|---|---|
| Request builders 0x17, 0x18, 0x19, 0x1A, 0x1B, 0x1D, 0x1E | implemented | `user_module.rs:370-480` |
| Reply parsers 0x48, 0x49, 0x4A | implemented, but assume no raw prefix (see errata) | `user_module.rs:482-539` |
| 0x47 API-version reply parser | **missing** (function ID constant only) | `frame.rs:57-58` |
| Slot upload (host → device 0x4A frame with `mm ss`) | **missing** | no builder in `user_module.rs` |
| `SysexTransaction` methods for user modules | **missing** | `transaction.rs` has global/program/tuning only |
| `.mnlgxdunit` read/write | **missing**. No code anywhere; `file-formats` feature covers `.mnlgxdprog` / `.mnlgxdlib` | `lib.rs:68-71`; grep finds no `mnlgxdunit` in `crates/` |
| README / site claims "Query, upload, download, clear, swap" and `.mnlgxdunit` | **overclaim** | `README.md:155-156`; `docs/index.html:503`, `632` |

The plan for `.mnlgxdunit` (a zip, feature-gated) is in the phase-3 plan and was never built.
Per [community] tooling, the file is a zip holding `manifest.json` (name, module, api, version,
dev_id, prg_id) and `payload.bin`. See [file formats](file-formats.md).

## Invariants

- `module_id ∈ {1, 2, 3, 4}`, and `slot < slot_count(module)` where `slot_count` is 16, 8, 8, 16.
  Both are < 0x80 and sent raw.
- Request lengths are fixed: `0x17` 8, `0x18`/`0x1D` 9, `0x19`/`0x1A`/`0x1B` 10, `0x1E` 11.
  The last byte is always `F7`.
- `user_module_info_size == 9` (Rust) and `user_slot_status_size == 32`. Encoded lengths: 9 → 11,
  32 → 37.
- TABLE 7: `decoded_len >= 8 + payload_size`, and
  `crc32_ieee(decoded[8 .. 8 + payload_size]) == u32_le(decoded[4..8])`.
- Upload payload (if implemented per [community]):
  `payload_size == 1024 + load_size + 132`, `u32_le(payload[1020..1024]) == load_size`, and
  `load_size <= max_program_size` from TABLE 5.
- `crc32_ieee("123456789") == 0xCBF43926`. Assert it at comptime or in a unit test.
- Swap: `slot_a` and `slot_b` are each validated. Rust allows `slot_a == slot_b`; the spec does
  not forbid it.

## Errata & gotchas

Record these on [errata](errata.md).

1. **Raw prefix on replies** **[community]**. NTS-1 replies put raw bytes after the function ID:
   `48 mm 00`, `49 mm ss 00`, `4A mm ss`. The 7-bit data follows them. Rust's parsers 7-bit-decode
   everything after the function byte, so on real traffic they would produce garbage or
   `PayloadTooShort`. The transaction layer already has `raw_request_response` (a raw-prefix
   path) for stored program dumps. User-module replies need the same treatment. Unverified on
   the xd, except that the `0x47` reply is raw.
2. **Empty slot** **[community]**. A status reply for an empty slot carries no data (`49 mm ss
   F7`; logue-manager treats any message of 10 bytes or fewer as empty). In Rust this surfaces as
   an error, not as "empty".
3. **TABLE 6 layout** — see the comparison table above. Rust is off by one from offset 5 and
   has 1-byte versions and a 16-byte name. The spec lists multi-byte fields high byte first, but
   the SDK stores them little-endian.
4. **TABLE 5 size**: 9 per the diagram, "8~9" per the table, 12 decoded on NTS-1.
5. **Channel must match** **[community]**. The xd answered `0x17` only when `g` equalled its
   global MIDI channel (captured on channel 12, `3B`). Other channels timed out with no NAK.
   SysEx goes to the XD's "SOUND" USB port; see [USB ports](usb-ports.md).
6. **No size limit in the spec**. The 0x4A body is "Variable". Bound it with TABLE 5
   `max_slot_size` at run time and with a comptime buffer (Zig notes).
7. **crc32fast is a hard dependency**. The phase-3 plan made it optional under `file-formats`,
   but `Cargo.toml` makes it unconditional. A Zig port needs only the 256-entry table.
8. **Rust tests are self-consistent only**. Every parser test feeds bytes that Rust's own
   `build_sysex` produced. No capture-based vector exists in the repo.

## Test vectors

From Rust tests (exact):

| Vector | Bytes / result | Citation |
|---|---|---|
| API version request ch 0 | 8 bytes, byte 6 = `17`: `F0 42 30 00 01 51 17 F7` | `user_module.rs:826-831` |
| Module info request ModFx ch 0 | `F0 42 30 00 01 51 18 01 F7` | `user_module.rs:833-843` |
| Slot status request Osc slot 5 ch 3 | `F0 42 33 00 01 51 19 04 05 F7` | `user_module.rs:858-866` |
| Slot data request Reverb slot 3 | bytes 6–8 = `1A 03 03` | `user_module.rs:873-879` |
| Clear slot ModFx slot 10 | bytes 6–8 = `1B 01 0A` | `user_module.rs:886-892` |
| Clear module DelayFx ch 15 | `F0 42 3F 00 01 51 1D 02 F7` | `user_module.rs:899-907` |
| Swap Osc 3 ↔ 7 ch 0 | `F0 42 30 00 01 51 1E 04 03 07 F7` | `user_module.rs:909-918` |
| Invalid slots | DelayFx 8, ReverbFx 8, ModFx 16, swap DelayFx (8,0) and (0,8) → error | `user_module.rs:868-871`, `881-884`, `894-897`, `920-928` |
| Module info LE | `01 02 03 04 05 06 07 08 0A` → `0x04030201`, `0x08070605`, 10 | `user_module.rs:689-697` |
| Status name | bytes 16.. = `48 69 00 58` → `"Hi"`; 16 × `A..P` → `"ABCDEFGHIJKLMNOP"` | `user_module.rs:732-753` |
| Status bad module | byte 1 = 99 → error | `user_module.rs:725-730` |
| Slot data short | 7 bytes → error; size 100 with 2 payload bytes → error | `user_module.rs:786-797` |
| CRC-32 | `""` → `0x00000000`; `"123456789"` → `0xCBF43926`; `00` → `0xD202EF8D` | `user_module.rs:803-820` |

Derived with the documented algorithm (zlib CRC-32 and NOTE 1 codec):

```text
crc32(01 02 03 04)          = 0xB63CFBCD
crc32(DE AD BE EF CA FE)    = 0x2542C930
TABLE 7 for payload 01 02 03 04 (12 bytes):
  04 00 00 00 CD FB 3C B6 01 02 03 04
7-bit encoded (14 bytes):
  30 04 00 00 00 4D 7B 3C 01 36 01 02 03 04
Rust-style frame (no raw prefix), ch 0:
  F0 42 30 00 01 51 4A 30 04 00 00 00 4D 7B 3C 01 36 01 02 03 04 F7
Community-style upload frame to Osc slot 0 would instead start:
  F0 42 30 00 01 51 4A 04 00 30 04 …
```

Hardware/community captures (not in the repo; [community]):

```text
xd   request  F0 42 3B 00 01 51 17 F7
xd   reply    F0 42 3B 00 01 51 47 02 01 01 00 F7          platform 2, API 1.1.0
NTS-1 0x48 body after F0..51: 48 01 00 01 74 1F 00 00 00 18 00 00 00 10 00 00 00
     → raw 01 00, decoded F4 1F 00 00 | 00 18 00 00 | 10 00 00 00 = 8180, 6144, 16
```

## Rust design

- `UserModuleId` is a `#[non_exhaustive]` enum with explicit discriminants. `max_slots()` and
  `from_byte()`/`to_byte()` are `match`es.
- `UserModuleInfo`, `UserSlotStatus` and `UserSlotData` each have `from_bytes(&[u8]) ->
  Result<Self>` and `to_bytes() -> Vec<u8>`. The status name is a `String`. Slot data owns a
  `Vec<u8>`.
- Requests use a private `sysex_header()` and push raw bytes. They do not use
  `frame::build_sysex`, which would 7-bit encode `mm`/`ss`. Only `0x17` uses
  `frame::build_sysex_request`.
- Replies use `frame::parse_sysex` (decode everything) plus a function-ID check
  (`WrongFunctionId`). This is the asymmetry behind errata 1.
- CRC is `crc32fast::hash`. The LE helpers are hand-written shifts.
- Essential: the ID/slot table, the fixed request bytes, the TABLE 7 header and CRC contract.
  Incidental: `Vec`/`String`, `from_utf8_lossy`, `crc32fast`.

## Zig / TigerStyle notes

- Constants: `user_module_info_size = 9` (accept ≥ 9; store 12 if you follow the captures),
  `user_slot_status_size = 32`, `user_slot_data_header_size = 8`,
  `user_prg_header_size = 1024`, `user_prg_sig_size = 132`, `slot_count = [_]u8{ 16, 8, 8, 16 }`
  indexed by `module - 1`.
- Buffer budget (assumption, verify with TABLE 5 on the device). The largest xd program is
  osc, `load_size ≤ 32768`. Then the TABLE 7 total is `8 + 1024 + 32768 + 132 = 33932` bytes,
  encoded `33932 + ceil(33932/7) = 38780` bytes, frame `7 + 2 + 38780 + 1 = 38790` bytes.
  NTS-1 reports a max slot size of 36848. Budget a comptime **40 KiB decoded / 48 KiB wire**
  buffer. At run time, reject when `payload_size > max_slot_size` or `> buffer_size`. See
  [memory budget](../zig/memory-budget.md).
- Model replies as `raw_prefix_len` + encoded body: 0x47 → 4 raw bytes and no body; 0x48 → 2;
  0x49 → 3; 0x4A → 2 ([community]). Make the prefix lengths comptime constants in one table, so
  a hardware capture can correct them in one place.
- Empty-slot detection: a 0x49 with no encoded body is `.empty`, not an error.
- Names: `[14]u8` per the SDK. Expose `name_slice()` up to the NUL. Never allocate.
- Error set: `error{ InvalidModuleId, InvalidSlotIndex, PayloadTooShort, PayloadTooLarge,
  Crc32Mismatch, WrongFunctionId }` plus framing errors.
- CRC: Zig std has a CRC-32 in `std.hash` (`std.hash.Crc32`), but its name has moved between
  releases. A local comptime table is ~10 lines and keeps the port self-contained. See
  [dependency elimination](../zig/dependency-elimination.md).

```zig
// sketch
const crc32_table: [256]u32 = blk: {
    @setEvalBranchQuota(256 * 8 * 4);
    var table: [256]u32 = undefined;
    for (&table, 0..) |*entry, index| {
        var c: u32 = @intCast(index);
        for (0..8) |_| c = if (c & 1 != 0) (c >> 1) ^ 0xEDB88320 else c >> 1;
        entry.* = c;
    }
    break :blk table;
};

pub fn crc32_ieee(bytes: []const u8) u32 {
    var c: u32 = 0xFFFFFFFF;
    for (bytes) |b| c = crc32_table[@as(u8, @truncate(c ^ b))] ^ (c >> 8);
    return c ^ 0xFFFFFFFF;
}

comptime {
    std.debug.assert(crc32_ieee("123456789") == 0xCBF43926);
}
```

## Open questions

- Do xd replies (`0x48`/`0x49`/`0x4A`) carry the same raw prefixes as NTS-1? Capture one of each.
- What is the xd TABLE 5 size: 9, 10 or 12 bytes? What values does it report per module?
- Confirm the TABLE 6 layout on the xd. Expect the [community] layout, with byte 1 = `0x02`.
- Which NAK codes does the xd send for an oversized or CRC-bad upload?
- Does the xd require a valid signature area or `num_param`/param descriptors in the 1024-byte
  header, or only `load_size`? logue-manager zero-fills them.
- Does a swap of a slot with itself return ACK or `0x2D`?

## Sources

- `crates/minilogue-xd/src/sysex/user_module.rs:1-24` — module doc.
- `crates/minilogue-xd/src/sysex/user_module.rs:36-90` — `UserModuleId`.
- `crates/minilogue-xd/src/sysex/user_module.rs:92-145` — TABLE 5 type.
- `crates/minilogue-xd/src/sysex/user_module.rs:147-233` — TABLE 6 type.
- `crates/minilogue-xd/src/sysex/user_module.rs:235-323` — TABLE 7 type.
- `crates/minilogue-xd/src/sysex/user_module.rs:325-352` — CRC and slot validation.
- `crates/minilogue-xd/src/sysex/user_module.rs:354-480` — raw header and request builders.
- `crates/minilogue-xd/src/sysex/user_module.rs:482-559` — reply parsers and LE helpers.
- `crates/minilogue-xd/src/sysex/user_module.rs:561-1009` — tests.
- `crates/minilogue-xd/src/sysex/frame.rs:30-64` — user function IDs.
- `crates/minilogue-xd/src/sysex/frame.rs:90-152` — status codes `0x23`–`0x2F`.
- `crates/minilogue-xd/src/sysex/transaction.rs:164-188` — `raw_request_response` (raw-prefix
  pattern used for program dumps).
- `crates/minilogue-xd/Cargo.toml:18-24` — `crc32fast = "1"`, not optional.
- `crates/minilogue-xd/src/lib.rs:68-71` — `file-formats` covers prog/lib only.
- `README.md:155-156`, `docs/index.html:503`, `docs/index.html:632` — overclaims.
- `docs/dev/0003-phase-3-implementation-plan-sysex-layer.md:56-62`, `203-213` — crc32 gating and
  `.mnlgxdunit` plan.
- Korg MIDI Impl rev 1.01 §1-4, §2-5, §3(11)–(21), TABLE 5, TABLE 6, TABLE 7, NOTE 1, NOTE 2.
- [korginc/logue-sdk](https://github.com/korginc/logue-sdk) — `platform/minilogue-xd/inc/userprg.h`
  (IDs, `user_prg_header_t`, header/sig sizes) and the per-module linker scripts (RAM sizes).
- [irgendwatt/logue-manager](https://github.com/irgendwatt/logue-manager) — `logue_lib.py`
  (upload builder, reply prefixes), `doc/analyze/probe_mnlgxd_output.txt` (xd `0x47`),
  `doc/analyze/logue_output.txt` and `find_crc_output.txt` (NTS-1 replies, CRC coverage).
