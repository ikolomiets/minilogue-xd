---
title: Korg 7-bit SysEx codec (NOTE 1)
category: protocol
sources:
  - crates/minilogue-xd/src/codec.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/sysex/tuning.rs
  - docs/wayfinder/research/xd-midi-capabilities.md
  - https://cdn.korg.com/us/support/download/files/5227b0b2f7c1da5acd0911ac3e8c4f38.txt
related:
  - protocol/sysex-framing.md
  - protocol/program-blob.md
  - protocol/global-blob.md
  - protocol/user-modules.md
  - protocol/tuning.md
  - protocol/errata.md
  - design/errors.md
  - design/byte-fidelity.md
  - zig/memory-budget.md
updated: 2026-10-11
---

# Korg 7-bit SysEx codec (NOTE 1)

> SysEx data bytes must have bit 7 clear. Korg therefore packs 8-bit blobs into 7-bit-safe
> bytes: each group of up to 7 source bytes becomes **one "MSB byte" followed by the 7 bytes
> with bit 7 cleared**. In the MSB byte, bit *i* holds bit 7 of source byte *i*. A final
> partial group of *r* bytes becomes *r* + 1 wire bytes. Encoding cannot fail. Decoding fails
> only if a wire byte has bit 7 set. The 1024-byte program blob encodes to **1171** wire bytes.

## Facts

### Bit layout of one group [spec]

```text
source (8-bit):   s0     s1     s2     s3     s4     s5     s6        (bytes 7n+0 .. 7n+6)
wire   (7-bit):   M      s0&7F  s1&7F  s2&7F  s3&7F  s4&7F  s5&7F  s6&7F
M  = bit 7 always 0
     bit 6 = s6.bit7, bit 5 = s5.bit7, ..., bit 0 = s0.bit7
```

This matches the NOTE 1 diagram, which labels the MSB byte's bits 6..0 as "7n+6,5,4,3,2,1,0"
(Korg MIDI Impl rev 1.01 NOTE 1). The Rust code sets `msb |= 1 << i` when
`source[i] & 0x80 != 0` (`codec.rs:36-47`).

### Encode algorithm

```text
encode(source[0..n]) -> target:
  for each chunk of source in steps of 7 (last chunk length r = 1..7):
    m = 0
    for i in 0..len(chunk): if chunk[i] & 0x80: m |= (1 << i)
    emit m
    for i in 0..len(chunk): emit chunk[i] & 0x7F
```

### Decode algorithm (Rust behaviour)

```text
decode(wire[0..w]) -> target:
  if w == 0: return []
  for each byte b in wire: if b & 0x80: error Codec("wire byte at offset {pos} has high bit set (0x..)")
  pos = 0
  while pos < w:
    m = wire[pos]; pos += 1
    g = min(w - pos, 7)
    for i in 0..g: emit wire[pos + i] | ((m >> i) & 1) << 7
    pos += g
```

All bytes are validated **before** any output is produced. The error reports the **first**
offending offset (`codec.rs:69-103`).

### Length formulas

```text
encoded_len(n) = (n / 7) * 8 + (n % 7 == 0 ? 0 : n % 7 + 1)
decoded_len(w) = (w / 8) * 7 + (w % 8 == 0 ? 0 : w % 8 - 1)   ; w % 8 == 1 → contributes 0
```

`encoded_len(n) ≤ n + ceil(n / 7)`. `decoded_len(w) ≤ w`. Rust uses `w` as the capacity
hint for decode (`codec.rs:84-86`).

| 8-bit payload | Use | Encoded wire bytes | Note |
|---|---|---|---|
| 0 | request / status frames | 0 | |
| 1 | — | 2 | |
| 3 | test vector | 4 | |
| 7 | one full group | 8 | |
| 9 | user module info (TABLE 5) | 11 | spec diagram "11 → 9" ✓ |
| 32 | user slot status (TABLE 6) | 37 | spec diagram "37 → 32" ✓ |
| 36 | user octave (TABLE 4) | 42 | if NOTE 1 applies; see Errata |
| 63 | global blob (TABLE 1) | 72 | spec diagram says "37 → 32"; see Errata |
| 336 | spec program-dump diagram | 384 | spec diagram "384 → 336" |
| 384 | user scale (TABLE 3) | 439 | if NOTE 1 applies; see Errata |
| 448 | v2.10+ `.mnlgxdpreset` program blob | 512 | file-only; files store 8-bit data, no codec |
| 1024 | program blob (TABLE 2) | 1171 | `1024 = 146 × 7 + 2` → `146 × 8 + 3` |
| user slot data | TABLE 7 | variable | 8-byte header + payload; see [user modules](user-modules.md) |

On the wire, a SysEx frame adds its header and the closing `F7` around the encoded payload.
For example, a current-program dump is 7 header bytes + 1171 + `F7` = 1179 bytes. A stored
program dump adds 2 program-number bytes, giving 1181. The canonical frame sizes are on
[SysEx framing](sysex-framing.md).

### Which messages carry NOTE 1 data

| Function | Message | Spec says NOTE 1? | Rust encodes with codec? |
|---|---|---|---|
| `0x40` | Current program data dump | yes (TABLE 2) | yes (`program/mod.rs:196-202`) |
| `0x4C` | Program data dump (1 prog) | yes; the 2 program-number bytes come **before** the encoded data and are not encoded | yes (`program/mod.rs:241-262`) |
| `0x51` | Global data dump | yes (TABLE 1) | yes (`global.rs:328-333`) |
| `0x48` | User module info | yes (TABLE 5) | yes (decode) |
| `0x49` | User slot status | yes (TABLE 6) | yes (decode) |
| `0x4A` | User slot data | yes (TABLE 7) | yes (decode) |
| `0x44` / `0x45` | User scale / octave dump | **no**: diagrams cite NOTE 3/4 only; TABLE 3/4 values are all 0..127 | **yes** (`tuning.rs:216-253`) |
| `0x47` | User API version | no (4 raw 7-bit fields) | no reply parser exists |
| `0x60` / `0x61` | Poly chain note on/off | no (raw 7-bit fields) | no (see [poly chain](poly-chain.md)) |
| `0x23`..`0x2F` | Status ACK/NAK | no payload | — |

Generic frame building calls `encode_7bit` on the whole payload (`frame.rs:209-231`).
Generic frame parsing calls `decode_7bit` on everything between the function ID and `F7`
(`frame.rs:327-341`).

### Wire timing

At DIN MIDI speed (31250 bit/s, 10 bits per byte = 3125 bytes/s), 1171 encoded bytes take
about 375 ms before framing (`xd-midi-capabilities.md:44`). USB is not bound by this rate.

## Invariants

- `encode` output length `== encoded_len(source.len)`.
- Every encoded byte satisfies `b & 0x80 == 0`.
- The MSB byte of a partial group with `r` data bytes has `m >> r == 0`. Unused high bits are
  zero.
- `decode(encode(x)) == x` for every byte string `x`. This is a paired assertion.
- For canonical input (`w % 8 != 1` and unused MSB bits zero),
  `encode(decode(w)) == w`.
- `decoded_len(encoded_len(n)) == n` for all `n`.
- A fixed-size blob decodes to exactly its size, for example
  `decoded_len(1171) == 1024`. Assert this before handing the bytes to the blob parser.

## Errata & gotchas

- **Program dump size: 336 [spec diagram] vs 1024 [spec TABLE 2, rust, hw].** The diagrams
  for functions `0x40` and `0x4C` say "384 Bytes (7bit) → 336 Bytes (8bit)". TABLE 2 runs to
  offset 1023. The crate emits 1024 bytes, which encode to 1171. Commit `1c16566` reports
  "byte-identical SysEx writes to hardware" with the 1024-byte blob **[hw]**. The diagram is
  stale. See [errata](errata.md) and [program blob](program-blob.md).
- **Global dump size: 32 [spec diagram] vs 63 [spec TABLE 1, rust].** The `0x51` diagram says
  "37 → 32". TABLE 1 has offsets 0..62. The crate uses `BLOB_SIZE = 63` (72 wire bytes)
  (`global.rs:138`). No hardware capture is cited. See [global blob](global-blob.md).
- **Tuning dumps** [spec] vs [rust]: the spec does not apply NOTE 1 to `0x44`/`0x45`. The
  crate 7-bit-encodes them anyway (384 → 439, 36 → 42). Because every TABLE 3/4 byte is ≤ 127,
  the encoded form is the raw form with an all-zero MSB byte inserted before every 7 data
  bytes. The two forms are **not** wire-compatible. See [tuning](tuning.md) and
  [errata](errata.md).
- **Non-canonical input is accepted** [rust]. A trailing lone MSB byte (`w % 8 == 1`) decodes
  to nothing. MSB bits beyond a partial group's length are ignored. So two different wire
  strings can decode to the same bytes, and `encode(decode(w)) != w` is possible. A strict
  decoder rejects both cases.
- **Validation pass before decoding.** Rust scans every byte for bit 7 first, so a frame with
  an embedded real-time byte (for example `F8` from CoreMIDI interleaving) fails as a whole.
  The transaction layer strips bytes > `0xF7` before framing (see
  [SysEx transactions](../design/sysex-transactions.md)).

## Test vectors

| 8-bit source | Wire | Citation |
|---|---|---|
| `FF 80 00` | `03 7F 00 00` | `crates/minilogue-xd/src/codec.rs:210-221`; doc examples `:23-28`, `:62-67` |
| `FF FF FF FF FF FF FF` | `7F 7F 7F 7F 7F 7F 7F 7F` | `crates/minilogue-xd/src/codec.rs:223-233` |
| `00 00 00 00 00 00 00` | `00 00 00 00 00 00 00 00` | `crates/minilogue-xd/src/codec.rs:235-243` |
| (empty) | (empty), in both directions | `crates/minilogue-xd/src/codec.rs:180-188` |
| `01 02 03` | `00 01 02 03` | `crates/minilogue-xd/src/sysex/frame.rs:522-532` (comment) |
| 336 × `AB` | 384 bytes | `crates/minilogue-xd/src/codec.rs:126-131` |
| 32 × `00` | 37 bytes | `crates/minilogue-xd/src/codec.rs:133-139` |
| 9 × `00` | 11 bytes | `crates/minilogue-xd/src/codec.rs:141-147` |

Rejections: wire `00 80` returns `Error::Codec` with a message containing "high bit set"
(`codec.rs:152-163`). Eight bytes of `00` with `FF` at any position 0..7 are rejected
(`codec.rs:165-175`).

Generated checks: round trip for lengths 0..=21 with `source[i] = (i * 37) as u8`
(`codec.rs:112-121`); length formula for n = 0..=300 (`codec.rs:193-205`); proptest round
trip over random byte strings of length 0..256 (`codec.rs:248-255`).

Derived vector for a Zig test. It was computed from the algorithm above, not taken from a
Rust test, and it exercises a full group plus a partial group:
`80 01 02 03 04 05 86 87` → `41 00 01 02 03 04 05 06 01 07`.
- In the first group, source byte 0 (`0x80`) sets bit 0 and source byte 6 (`0x86`) sets
  bit 6, so `M = 0x41`.
- Source byte 7 (`0x87`) starts a partial group, so `M = 0x01` followed by `07`.

## Rust design

- There are two free functions:
  - `encode_7bit(&[u8]) -> Vec<u8>` is infallible.
  - `decode_7bit(&[u8]) -> Result<Vec<u8>>` returns `Error::Codec(String)`.
- Both allocate. Neither keeps state. Nothing else is involved: no traits and no streaming
  (`codec.rs:29-103`).
- Callers are `frame::build_sysex` and `frame::parse_sysex`, plus the stored-program dump
  builder and parser, which put the 2 program-number bytes outside the encoded region
  (`frame.rs:220-231, 327-341`; `program/mod.rs:244-309`).
- **Essential design**: the bit layout, the partial final group, and the all-or-nothing
  high-bit validation.
- **Incidental Rust idioms**: the `Vec` returns, the `String` error message, and the
  `chunks(7)` iterator.

## Zig / TigerStyle notes

- Make both functions write into caller-provided buffers and return a `u32` length. Neither
  needs an allocator.
- `encoded_len` and `decoded_len` are `fn(u32) u32`, usable at comptime, so every SysEx buffer
  size can be a comptime constant. Examples: `comptime assert(encoded_len(1024) == 1171)`,
  `comptime assert(encoded_len(63) == 72)`, `comptime assert(encoded_len(32) == 37)`.
- Preconditions:
  - `encode`: `assert(target.len >= encoded_len(source.len))`.
  - `decode`: `assert(target.len >= decoded_len(source.len))`.
- Postconditions:
  - `encode`: assert that every target byte is `< 0x80` and the returned length is exact.
  - `decode`: assert that the returned length equals `decoded_len`.
- The loops are bounded by `source.len` and the inner group loop by 7. Neither uses
  recursion.
- Error set: `error{ WireByteInvalid }` for bit 7 set, plus optionally
  `WireLengthInvalid` when `w % 8 == 1` and `MsbPaddingInvalid` for strict canonical
  decoding. Recommended: be strict, and assert canonicity on the encode side.
- For fixed blobs, decode straight into the blob's backing array, for example
  `*[1024]u8`. First check `decoded_len(wire.len) == 1024`, and fail with an operating
  error otherwise.

```zig
// sketch
pub fn encoded_len(source_len: u32) u32 {
    const remainder = source_len % 7;
    return (source_len / 7) * 8 + if (remainder == 0) 0 else remainder + 1;
}

pub fn encode(source: []const u8, target: []u8) u32 {
    assert(target.len >= encoded_len(@intCast(source.len)));
    var target_index: u32 = 0;
    var source_index: u32 = 0;
    while (source_index < source.len) : (source_index += 7) {
        const group = source[source_index..@min(source_index + 7, source.len)];
        var msb: u8 = 0;
        for (group, 0..) |byte, i| msb |= (byte >> 7) << @intCast(i);
        target[target_index] = msb;
        for (group, 0..) |byte, i| target[target_index + 1 + i] = byte & 0x7F;
        target_index += 1 + @as(u32, @intCast(group.len));
    }
    assert(target_index == encoded_len(@intCast(source.len)));
    return target_index;
}
```

## Open questions

- Does the device accept or emit **tuning dumps** (`0x44`/`0x45`) 7-bit-encoded, as the
  crate does, or raw, as the spec implies? Hardware capture needed.
- Does the device ever emit a non-canonical trailing MSB byte? If not, a strict Zig decoder
  is safe.
- Global dump on the wire: is it 63 bytes (72 wire bytes) or 32 bytes (37 wire bytes) on
  current firmware? See [global blob](global-blob.md).

## Sources

- `crates/minilogue-xd/src/codec.rs:1-11` — module description.
- `crates/minilogue-xd/src/codec.rs:15-50` — `encode_7bit`.
- `crates/minilogue-xd/src/codec.rs:52-103` — `decode_7bit`.
- `crates/minilogue-xd/src/codec.rs:105-256` — tests.
- `crates/minilogue-xd/src/sysex/frame.rs:205-231, 327-341, 515-532` — frame build/parse use
  the codec.
- `crates/minilogue-xd/src/sysex/program/mod.rs:196-202, 241-309` — program dumps.
- `crates/minilogue-xd/src/sysex/global.rs:138, 328-333` — 63-byte global blob.
- `crates/minilogue-xd/src/sysex/tuning.rs:216-253` — tuning dumps 7-bit-encoded.
- `docs/wayfinder/research/xd-midi-capabilities.md:44, 56` — DIN timing, dump-size discrepancy.
- Git commit `1c16566` — [hw] byte-identical program SysEx writes.
- Korg MIDI Impl rev 1.01 NOTE 1 (bit layout), §(4)/(5) program dump diagrams (336),
  §(6) global dump diagram (32), §(9-1)/(10-1) tuning dumps (NOTE 3/4, no NOTE 1),
  §(19)/(20)/(21) user module data (NOTE 1, TABLE 5/6/7), TABLE 1, TABLE 2, TABLE 3, TABLE 4.
