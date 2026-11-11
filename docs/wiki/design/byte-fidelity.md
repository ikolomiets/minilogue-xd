---
title: Byte fidelity — round-tripping device blobs without loss
category: design
sources:
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/sysex/program/synth.rs
  - crates/minilogue-xd/src/sysex/program/sequencer.rs
  - crates/minilogue-xd/src/sysex/program/file.rs
  - crates/minilogue-xd/src/sysex/helpers.rs
  - crates/minilogue-xd/src/sysex/global.rs
  - crates/minilogue-xd/examples/sysex_roundtrip_diag.rs
  - crates/minilogue-xd/examples/quintal.rs
  - git commit 1c16566
related:
  - protocol/program-blob.md
  - protocol/sequencer-blob.md
  - protocol/global-blob.md
  - protocol/file-formats.md
  - protocol/korg-7bit-codec.md
  - protocol/sysex-framing.md
  - design/builders.md
  - design/testing-strategy.md
  - protocol/errata.md
  - zig/tigerstyle-mapping.md
  - zig/port-overview.md
  - zig/test-plan.md
updated: 2026-11-11
---

# Byte fidelity — round-tripping device blobs without loss

> The device stores data the spec does not document. It uses the upper 6 bits of every
> 10-bit high byte, the "reserved" nibble of byte 148, NUL name padding, and enum values
> outside the documented range. A library that parses a blob into typed fields and writes
> it back from zeros corrupts that data. The device then answers `0x26` DATA FORMAT ERROR
> or plays a different sound. The Rust crate learned this on hardware (commit `1c16566`).
> It now keeps the raw 1024-byte blob and overlays the typed fields onto it. **The Zig port
> should invert the design:** the blob *is* the storage (`[1024]u8`), and typed accessors
> read and write bits in place. Fidelity then holds by construction, and the only
> remaining invariant is "a setter touches only its own bits".

## Facts

### What the device puts in bits the spec does not describe

| Location | Spec | Device | Provenance |
|---|---|---|---|
| High byte of each 10-bit pair, bits 7..2 (34 pairs in bytes 19–154) | not mentioned | carries data; must be preserved | **[hw]** `1c16566` bug 2 |
| 10-bit byte order | Bit column reads `H:0~7`, `L:0~7` | **little-endian**: low byte first | **[hw]** `1c16566` bug 1; reverb 870 = `66 03` (`quintal.rs:716`) |
| Byte 148 bits 7..4 | Reserved | non-zero on hardware | **[hw]** `1c16566` bug 4 |
| Name bytes 4–15 | P1 charset, which has no NUL | right-padded with `0x00` | **[hw]** `1c16566` bug 5 |
| Sequencer bytes 156–161 | `PREDSQ` magic | required. A missing magic produces `0x26` | **[hw]** `1c16566` bug 3 |
| Sequencer magic from internal memory | S1: `SEQD` converted to `SQ` on load | `0x4C` replies can carry `PREDSEQD` | **[hw]** community capture ([sequencer blob](../protocol/sequencer-blob.md)) |
| User-param type bytes 148/149 | values 0~2 | factory patches contain 3 | **[hw]** `param/enums.rs:477-480` |
| BPM high byte | `H:0~3` (bits 7..4 not BPM) | unknown; preserve | [spec] |
| Motion slot `u16` low byte | bits 0/1 = on/smooth | present | [spec], [hw] capture |

The commit message reports the outcome: "Verified: byte-identical SysEx writes to hardware,
correct patch name display, and matching sound compared to CC-saved reference patches"
(`git show 1c16566`). The tool behind that check is the `sysex_roundtrip_diag` example. It
reads the edit buffer, parses it, re-serializes it and diffs byte by byte. With `--compare`,
it diffs two stored slots (`sysex_roundtrip_diag.rs:1-16`, `:18-63`, `:139`).

### How Rust achieves fidelity today

1. **Retain the raw blob.** `ProgramData { synth, sequencer, raw_blob:
   Option<Box<[u8; 1024]>> }` (`mod.rs:86-97`). `from_bytes` copies the first 1024 input
   bytes into `raw_blob` (`mod.rs:131-153`). `new()` leaves it `None` (`mod.rs:113-119`).
2. **Overlay the typed fields.** `to_bytes` calls `SynthParams::to_bytes_with_base(&raw[..156])`
   and `SequencerParams::to_bytes_with_base(&raw[156..])` when a raw blob is present. With no
   raw blob, it calls the zero-based `to_bytes()` (`mod.rs:159-180`).
3. **Masking writers.** `write_10bit` writes the whole low byte but only bits 1..0 of the
   high byte: `hi = (hi & 0xFC) | (v >> 8)` (`helpers.rs:31-35`). `read_10bit` masks with
   `0x03FF` (`helpers.rs:16-20`).
4. **Model the leftovers as fields.** Byte 148's upper nibble lives in
   `user_param_type_flags` (`synth.rs:335-336`, read `:502`, written `:631-633`). The name
   validator accepts `0x00` (`synth.rs:29-31`).

### What actually survives an overlay

`SynthParams::to_bytes_with_base` writes **all 156 synth bytes**. The base contributes only
the upper 6 bits of the 34 ten-bit high bytes (offsets 20, 25, 27, 31, 33, 37, 43–63 odd,
67–79 odd, 84, 86, 96, 98, 102, 104, 108, 110, 152, 154). This was checked by scanning
`synth.rs:520-646` while writing this page. `SequencerParams::to_bytes_with_base` writes
all 868 sequencer bytes: the header 0–33, 16 steps × 52 bytes = 34–865, and the ARP bytes
866–867 (`sequencer.rs:253-299`). Everything else survives only because it was parsed into
a field that stores it **verbatim**: raw `u8` fields, the 7-byte motion slots and the name
array.

| Class of data | Survives `from_bytes → to_bytes`? | Why |
|---|---|---|
| Upper 6 bits of 10-bit high bytes | yes | base + `write_10bit` masking |
| Byte 148 bits 7..4 | yes | modelled field `user_param_type_flags` |
| Raw `u8` fields (`voice_mode_type`, `mod_fx_type`, `micro_tuning`, …) | yes | stored verbatim |
| Motion data, step notes/velocities/gates | yes | stored verbatim |
| Boolean bytes holding anything other than 0/1 | **no**: collapsed to 0/1 | `u8::from(bool)` overwrites the byte |
| Enum bytes with undocumented values | **parse fails** | `from_program_value(..)?` rejects them, so the whole blob is unreadable (`synth.rs:402-463`) |
| `PREDSEQD` sequencer | **parse fails** | `MAGIC` must equal `PREDSQ` (`sequencer.rs:175-180`) |
| Bytes beyond 1024 in a file entry | dropped silently | `from_bytes` checks only `len < SIZE` ([file formats](../protocol/file-formats.md)) |
| Global blob (63 bytes) | only if every boolean byte is 0/1 | no raw copy is kept ([global blob](../protocol/global-blob.md#round-trip-fidelity)) |

## Invariants

These are the properties the port must hold, and test.

1. **Identity.** For every blob `b` the device can produce, `serialize(parse(b)) == b`,
   byte for byte. That covers all 1024 bytes, including reserved bits and undocumented
   values.
2. **Locality.** Setting field `F` changes only the bits in `F`'s mask at `F`'s offsets.
   Each of the other 8,192 bits stays as it was.
3. **Wire identity.** `decode7(encode7(b)) == b` for any length. A 1024-byte blob always
   encodes to 1171 bytes ([7-bit codec](../protocol/korg-7bit-codec.md)).
4. **No partial acceptance.** A parse either keeps every byte or fails with a specific
   error. It never drops a tail or rewrites a value without reporting it.
5. **Freshly built blobs are valid.** A blob made from scratch (not from a device) must
   have the magic numbers, the sequencer header in spec order and NUL name padding. Bits
   the spec doesn't describe are zero (see [builders](builders.md)).

## Errata & gotchas

| # | Item | Detail | Provenance |
|---|---|---|---|
| 1 | `PartialEq` ignores `raw_blob` | Two `ProgramData` can compare equal yet serialize to different bytes (`mod.rs:99-103`). Tests that assert `data == recovered` do not prove fidelity. | [rust] |
| 2 | `ProgramData::new` / builders have no base | Output has zero upper bits. That is fine for fresh patches. But editing a device patch through `new(parsed.synth, ...)` loses the device's bits. | [rust] |
| 3 | Strict enum decode blocks fidelity | One unknown byte, for example a future firmware value, makes the whole program unreadable. With `.mnlgxdlib` files the entire library fails ([file formats](../protocol/file-formats.md)). | [rust] |
| 4 | Booleans collapse | `key_trig`, `lfo_key_sync`, the FX on bits and others read `!= 0` and write 0/1. Inverted SYNC/RING use `== 0`. A device value of 2 would come back as 1 (or 0 for the inverted pair). | [rust] |
| 5 | Sequencer header rotation | Rust's header layout is rotated against the spec and the hardware. `parse → serialize` is still the identity because the bug is symmetric. But every field accessor reads the wrong byte, and every fresh blob is malformed ([sequencer blob](../protocol/sequencer-blob.md)). | [hw] + [spec] vs [rust] |
| 6 | `ProgramName::from_string` pads with spaces | Hardware pads with NUL. `as_str` trims spaces but not NULs, so a device name keeps `\0` characters (`synth.rs:67-84`). | [rust] vs [hw] |
| 7 | Silent name truncation | `from_string` keeps the first 12 bytes without an error. | [rust] |
| 8 | No ProgramData-level fidelity test | Upper-bit preservation is tested only at the helper level (`helpers.rs:137-144`) and for byte 148 (`synth.rs:1245-1255`). No test feeds a real device blob through `ProgramData`, and the repo has no `.syx`/`.prog_bin` fixtures. | [rust]; see [testing strategy](testing-strategy.md) |
| 9 | Global blob has no raw copy | It is fully modelled, but any boolean byte other than 0/1 is normalized. | [rust] |

## Test vectors

| Case | Input | Expected | Source |
|---|---|---|---|
| `write_10bit` keeps upper bits | `[00 A4]`, write 0x1FF | `[FF A5]` | `helpers.rs:137-144` |
| `write_10bit` zero | `[FF FF]`, write 0 | `[00 FC]` | `helpers.rs:113-118` |
| `write_10bit` max into zeros | `[00 00]`, write 1023 | `[FF 03]` | `helpers.rs:120-126` |
| Input masked to 10 bits | `[00 00]`, write 0xFFFF | `[FF 03]` | `helpers.rs:128-134` |
| Hardware LE read | `[99 02]` | 665 (Replicant xd cutoff) | `helpers.rs:156-161` [hw] |
| Hardware LE read | bytes 107–108 = `66 03` | reverb time 870 | `quintal.rs:716` [hw] |
| Byte 148 flags | `0x01 \| 0x02<<2 \| 0xF0` | p5 = 1, p6 = 2, flags `0xF0`, re-emitted unchanged | `synth.rs:1245-1255` |
| Exhaustive 10-bit | all v in 0..=1023 | `read(write(v)) == v` | `helpers.rs:147-152` |

Tests the Zig port adds (see [test plan](../zig/test-plan.md)):

- **Real-blob identity.** Use every program from available `.mnlgxdlib` / `.mnlgxdprog`
  files and community `.syx` captures: `serialize(parse(b)) == b`.
- **Random-blob identity.** Take a PRNG-filled blob, patch in valid magic numbers and run
  the identity check. Invariant 1 should hold even for garbage in the undocumented fields.
- **Locality.** For each setter, on a random blob: set a random value, XOR with the
  original, and assert the diff ⊆ the field's comptime mask.

## Rust design

- The design grew in two steps. First came "typed struct ↔ bytes" (clean, but lossy). Then
  the raw-base overlay was retrofitted after hardware testing. The result keeps both
  representations, so they can disagree: equality, `new()` and builders all use only the
  typed side.
- `to_bytes_with_base` on `SynthParams` returns `[u8; 156]` and takes `&[u8; 156]`. On
  `SequencerParams` it returns `Vec<u8>`, takes `&[u8]` and **panics** on a short base
  (`sequencer.rs:253-258`).
- About 300 lines of symmetric `from_bytes` / `to_bytes_with_base` code must stay in sync
  by hand. The sequencer header rotation is an example of a symmetric bug that the round
  trip tests cannot catch.

## Zig / TigerStyle notes

- **Store bytes, expose views.**

  ```zig
  pub const Program = struct {
      bytes: [1024]u8,

      pub fn cutoff(p: *const Program) u10 {
          return read_u10_le(p.bytes[60..62]);
      }
      pub fn set_cutoff(p: *Program, v: u10) void {
          write_u10_le(p.bytes[60..62], v); // touches bits 9..0 only
      }
  };

  fn write_u10_le(b: *[2]u8, v: u10) void {
      b[0] = @truncate(v);
      b[1] = (b[1] & 0xFC) | @as(u8, @intCast(v >> 8));
  }
  ```

  Generate the accessors from one comptime field table: name, offset, kind and bit mask.
  The same table drives the locality test, the [memory budget](../zig/memory-budget.md)
  asserts and the docs.
- **Enums as raw bytes plus an optional typed view.** For example,
  `fn vco1_wave(p) ?VcoWave` returns `null` for unknown values. `fn vco1_wave_raw(p) u8`
  always works. Setters take the enum. Parsing never fails on a value, only on structure:
  length and magic.
- **Accept both sequencer magics.** Expose `fn seq_magic(p) enum { sq, seqd }`. Whether to
  normalize to `SQ` on write is the caller's choice; preserve by default.
- **Booleans.** The getter returns `b != 0`. The setter writes 0/1, which is the only lossy
  operation, and only for that field. Note this in each setter's doc.
- **Validation is a separate pass.** `fn validate(p) ValidationReport` lists out-of-range
  values without changing anything. Writers may refuse to send an invalid blob in strict
  mode.
- **Fresh blobs** come from a comptime `init_program` template. The template has the
  `PROG` and `PREDSQ` magic numbers, the spec-order header, NUL-padded names and documented
  defaults. Assert the template's own invariants at comptime.
- **Global blob** gets the same treatment: `Global = struct { bytes: [63]u8 }`
  ([global blob](../protocol/global-blob.md)).
- **No allocation.** `Program` is 1 KiB and lives in a caller-owned slot. A library of 500
  programs is a fixed `[500]Program` (see [memory budget](../zig/memory-budget.md)).

## Open questions

- What do the upper 6 bits of the 10-bit high bytes encode? Knob-catch flags or motion
  state are guesses. Diffing two hardware saves that differ only in knob movement would
  tell.
- Does the device accept an all-zero upper-bit blob for *every* parameter, or only for
  most? The 1c16566 tests used the raw base for device patches, and builders produce
  zero-based blobs.
- Should a Zig writer normalize `PREDSEQD` to `PREDSQ`? The spec says the device converts
  on load.

## Sources

- `git show 1c16566`: "Fix SysEx program dump round-trip for byte-perfect hardware fidelity"
  (five bugs, plus raw-blob preservation).
- `crates/minilogue-xd/src/sysex/program/mod.rs:74-182` (`ProgramData`, `from_bytes`,
  `to_bytes`).
- `crates/minilogue-xd/src/sysex/program/synth.rs:29-31`, `:335-336`, `:372-513`,
  `:520-656`.
- `crates/minilogue-xd/src/sysex/program/sequencer.rs:150-310`.
- `crates/minilogue-xd/src/sysex/helpers.rs:1-75` and tests `:77-259`.
- `crates/minilogue-xd/examples/sysex_roundtrip_diag.rs`, `crates/minilogue-xd/examples/quintal.rs:716`.
- Korg MIDI Implementation rev 1.01 TABLE 2 and notes P1, S1.
