---
title: Dependency elimination
category: zig
sources:
  - crates/minilogue-xd/Cargo.toml
  - crates/minilogue-xd/src/sysex/program/file.rs
  - crates/minilogue-xd/src/sysex/user_module.rs
  - crates/minilogue-xd/src/transport.rs
  - /opt/homebrew/Cellar/zig/0.16.0_1/lib/zig/std (Zig 0.16 standard library, local install)
related:
  - sources/rust-crate.md
  - zig/platform-midi.md
  - zig/port-overview.md
  - protocol/file-formats.md
  - protocol/user-modules.md
updated: 2026-10-11
---

# Dependency elimination

> Every Rust dependency can be replaced by the Zig standard library, the OS's own MIDI API, or
> under 300 lines of local code. The Zig port therefore needs **zero third-party packages**.
> Under TigerStyle, "zero dependencies" means no `build.zig.zon` dependencies; the Zig std and
> system frameworks/libraries are allowed. The only non-trivial replacements are the MIDI
> backends ([platform MIDI](platform-midi.md)) and a small zip writer.

## Facts

| Rust crate | Used for | Used where | Zig replacement | Effort |
|------------|----------|------------|-----------------|--------|
| `thiserror` 2 | `#[derive(Error)]`, `Display` for `Error`/`SysexError` | `error.rs` | Native error sets (`error{Timeout, Nak, ...}`), plus an optional `Diagnostics` out-struct for the payload (status byte, offset, expected/actual). See [errors](../design/errors.md). | trivial |
| `bitflags` 2 | **nothing**: declared, never imported | — | — | none |
| `crc32fast` 1 | CRC-32 of user-slot data | `sysex/user_module.rs:331` | `std.hash.Crc32` (alias of `Crc32IsoHdlc`, the same zlib/PKZIP polynomial `0xEDB88320` reflected). Or a 256-entry comptime table, about 20 lines. | trivial |
| `music-comp-mt` 0.5 | Note/pitch/chord types; re-exported as `minilogue_xd::theory` | examples only (`test_drive`, `berlin_school`, `exit`, `quintal`, `export_midi`) | Not part of the library. Port the examples with a small comptime note-name table (`C`, `C#`, … × octave → MIDI number) and literal chord intervals. | small |
| `midir` 0.10 (`midi-io`) | Port enumeration, connect, send, callback input | `transport.rs:136-330`, `device.rs:141-260` | Direct OS bindings: CoreMIDI (macOS), ALSA rawmidi through `/dev/snd` ioctls (Linux), WinMM (Windows). See [platform MIDI](platform-midi.md). | **large** |
| `zip` 8.5 (`file-formats`, deflate) | Read/write `.mnlgxdprog` / `.mnlgxdlib` | `sysex/program/file.rs:12-140` | Read: a small central-directory reader plus `std.compress.flate.Decompress` (raw container). Write: a small local-header/central-directory writer plus `std.compress.flate.Compress` or method 0 (stored). | medium |
| `proptest` 1 (dev) | Property tests for codec and 10-bit encoding | `codec.rs`, `param/encoding.rs` | A seeded `std.Random.DefaultPrng` loop with a fixed iteration count (deterministic, so failures reproduce). Optionally use `std.testing.fuzz` for coverage-guided runs. See [test plan](test-plan.md). | small |
| `std::sync::mpsc` | Callback thread → caller handoff for input | `transport.rs:266-318` | A fixed-capacity SPSC ring buffer of bytes (`[N]u8` + atomic head/tail). See [platform MIDI](platform-midi.md#input-handoff). | small |
| `std::time::{Instant, Duration}` | Transaction deadline | `sysex/transaction.rs:103-140` | Inject time: the core receives `now_ns: u64` from the caller. Only `io/` reads a monotonic clock (`std.Io.Clock` / `Io.Timestamp` in 0.16). | trivial |
| `Vec<u8>`, `String`, `VecDeque` | Every message and payload | everywhere | Fixed arrays sized by comptime maxima (see [memory budget](memory-budget.md)); slices into caller buffers. | pervasive but mechanical |

### What the Zig 0.16 std actually provides (verified locally)

- `std.zip`: **read only**. It has `EndRecord.findBuffer`/`findFile`, an `Iterator` over a
  `*File.Reader`, and `Entry.extract` *into a directory*. Methods handled: `store` (0) and
  `deflate` (8). There is no writer and no "extract into memory" helper. For in-memory
  operation, parse the end record and central directory yourself (a few dozen lines, all
  fixed-size little-endian structs), then stream each entry through
  `std.compress.flate.Decompress`.
- `std.compress.flate`: `Decompress.zig` and a full `Compress.zig` (about 2.6k lines; modes
  include `Raw` (stored blocks), `Huffman`, and full LZ77+Huffman). Use `Container.raw` for
  zip entries, since zip stores raw deflate streams without zlib/gzip headers.
- `std.hash.Crc32`: needed both for user-module CRC and for zip local/central headers.
- `std.testing.fuzz` and `std.testing.Smith` exist in 0.16 for coverage-guided fuzzing.

APIs churn between Zig releases (0.16 → 0.17 changed `Io`). Pin one Zig version in
`build.zig.zon` (`minimum_zig_version`) and keep std usage behind thin local wrappers.

## Invariants

- No `build.zig.zon` `.dependencies`. CI should fail if any appear.
- Every replacement keeps byte-exact behavior. The CRC must equal `crc32fast::hash` for the same
  input; `0xCBF43926` for `"123456789"` is the standard check value. Zip files written by the
  port must be readable by the Rust crate, and the reverse.
- Only `io/` modules may import OS bindings. Protocol modules import nothing but `std` (and
  ideally only `std.debug.assert`, `std.mem`, `std.math`).

## Errata & gotchas

- **Deflate vs stored.** Rust writes entries with `CompressionMethod::Deflated`
  (`file.rs:52`, `file.rs:120`). Nobody has confirmed whether the Korg Librarian and the XD
  firmware accept **stored** (method 0) entries. Until that is tested on hardware, write
  deflate with `std.compress.flate.Compress` to match the Rust output. Reading must accept both.
- **Zip details that bite.** Use little-endian everywhere. Include the data-descriptor flag
  (bit 3) only if you stream. Zip64 is never needed: files are a few hundred KB at most. Use
  DOS date/time fields; a fixed constant is fine and keeps output deterministic, which helps
  golden tests. The CRC-32 covers *uncompressed* data.
- **`music-comp-mt` is re-exported from the library** (`lib.rs`: `pub use music_comp_mt as
  theory`), so it looks like a core dependency. It isn't one; leave it out of the Zig library.
- Linking `CoreMIDI.framework`, `libwinmm`, or (optionally) `libasound` is a *system*
  dependency, not a package dependency. The Linux rawmidi route avoids even libasound.

## Zig / TigerStyle notes

- Size zip buffers statically. A `.mnlgxdlib` holds at most 500 programs × 1024 bytes plus
  headers and `FileInformation.xml`, about 520 KB uncompressed. Stream entry by entry with a
  fixed `[1024]u8` entry buffer and a fixed central-directory scratch area instead of loading
  whole archives. See [memory budget](memory-budget.md).
- The CRC table can be a `comptime`-generated `[256]u32`. Assert its known check value at
  comptime.
- For property tests, follow TigerBeetle's approach: a seeded PRNG, the seed printed on
  failure, and bounded iteration counts.

## Open questions

- Does the Korg Librarian accept stored (method 0) zip entries? (Hardware or librarian test.)
- Does the XD accept `.mnlgxdlib` files whose `FileInformation.xml` differs in whitespace or
  attribute order from Korg's? See [file formats](../protocol/file-formats.md).

## Sources

- `crates/minilogue-xd/Cargo.toml` — `[dependencies]`, `[dev-dependencies]`, features.
- `crates/minilogue-xd/src/sysex/program/file.rs:12-140` — zip read/write, `Deflated` method.
- `crates/minilogue-xd/src/sysex/user_module.rs:331` — `crc32fast::hash`.
- `crates/minilogue-xd/src/transport.rs:225-320` — midir input + `mpsc` handoff.
- Zig 0.16 std: `std/zip.zig` (read-only; `CompressionMethod{store=0, deflate=8}`),
  `std/compress/flate/Compress.zig`, `std/compress/flate/Decompress.zig`.
