---
title: Program file formats (.mnlgxdprog, .mnlgxdlib, .mnlgxdpreset)
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/program/file.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/device.rs
  - crates/minilogue-xd/src/lib.rs
  - crates/minilogue-xd/Cargo.toml
  - README.md
  - CLAUDE.md
related:
  - protocol/program-blob.md
  - protocol/sequencer-blob.md
  - protocol/user-modules.md
  - protocol/tuning.md
  - design/byte-fidelity.md
  - design/errors.md
  - zig/dependency-elimination.md
  - zig/memory-budget.md
updated: 2026-11-11
---

# Program file formats (.mnlgxdprog, .mnlgxdlib, .mnlgxdpreset)

> Korg's librarian files for the minilogue xd are ZIP archives. Each program is a
> `Prog_NNN.prog_bin` entry holding the raw 1024-byte [program blob](program-blob.md) — the
> same bytes as a SysEx dump, *not* 7-bit encoded. A `FileInformation.xml` manifest sits
> alongside. The Rust crate implements a deliberately minimal subset: read one program, read
> all `*.prog_bin` entries of a library, and write Deflated archives with only `prog_bin`
> entries plus a 5-line manifest. It does **not** write `.prog_info` metadata. It does **not**
> implement `.mnlgxdpreset`. The README claim that 448-byte preset blobs are supported is
> false. User oscillator/effect units (`.mnlgxdunit`) are covered in
> [user modules](user-modules.md).

## Facts

### Container family

| Extension | Purpose | Program entries | Rust support |
|-----------|---------|-----------------|--------------|
| `.mnlgxdprog` | one program | 1 × `Prog_NNN.prog_bin` | read + write (`Prog_000` only) |
| `.mnlgxdlib` | program library | N × `Prog_NNN.prog_bin` | read + write |
| `.mnlgxdpreset` | preset pack (library + `PresetInformation.xml`) | N × `Prog_NNN.prog_bin` | **none** |
| `.mnlgxdunit` | logue SDK user module | n/a | see [user modules](user-modules.md) |

The whole module is behind the `file-formats` Cargo feature (`program/mod.rs:8-9`,
`Cargo.toml:12-16`, `lib.rs:70`). That feature pulls in the `zip` crate version 8.5 with only
the `deflate` feature (`Cargo.toml:24`). No example and no other module calls these
functions. `examples/quintal.rs --export` writes a Standard MIDI File, not a program file
([standard MIDI file](standard-midi-file.md)).

### Archive entries (community-documented librarian layout)

| Entry name | Content | Written by Rust? |
|------------|---------|------------------|
| `Prog_NNN.prog_bin` | raw 1024-byte program blob; `NNN` = 3-digit zero-padded index | yes |
| `Prog_NNN.prog_info` | XML, root `xd_ProgramInformation`: `Programmer`, `Comment`, optional `Copyright` (logue-manager adds `ProgramName`) | **no** |
| `FileInformation.xml` | manifest, root `KorgMSLibrarian_Data` | yes (minimal) |
| `PresetInformation.xml` | preset metadata, root `xd_Preset`: `DataID`, `Name`, `Author`, `Version`, `NumOfProg`, `Date`, `Prefix`, `Copyright` | no |
| `FavoriteData.fav_data` | favourites table (XML, root `xd_Favorite` → `Bank` → `Data`…) | no |

Sources: loguetools `common.py` (`prog_info_template_xml`, `fileinfo_xml`, `presetinfo_xml`)
and `xd.py` (`favorite_template`) @ba0fc46; logue-manager `logue_lib.py:663-697` @a04b2ea
**[community]**. No Korg-produced archive is checked into this repo, so treat the exact
attribute set as **[community]**, not **[spec]**. The Korg MIDI Implementation does not cover
files.

#### FileInformation.xml — full community form

```xml
<?xml version="1.0" encoding="UTF-8"?>
<KorgMSLibrarian_Data>
  <Product>minilogue xd</Product>
  <Contents NumProgramData="2" NumPresetInformation="0" NumTuneScaleData="0"
            NumTuneOctData="0" NumFavoriteData="0">
    <ProgramData>
      <Information>Prog_000.prog_info</Information>
      <ProgramBinary>Prog_000.prog_bin</ProgramBinary>
    </ProgramData>
    <ProgramData>
      <Information>Prog_001.prog_info</Information>
      <ProgramBinary>Prog_001.prog_bin</ProgramBinary>
    </ProgramData>
  </Contents>
</KorgMSLibrarian_Data>
```

Variants:

- For a preset, `NumPresetInformation="1"` and the manifest contains
  `<PresetInformation><File>PresetInformation.xml</File></PresetInformation>`.
- A library with more than one program may set `NumFavoriteData="1"` with
  `<FavoriteData><File>FavoriteData.fav_data</File></FavoriteData>`.
- loguetools skips "Init Program" patches but keeps the original index in names, so `NNN`
  can have gaps. Readers must not assume dense numbering.

### What Rust writes

**`write_prog_file(writer, &ProgramData)`** (`file.rs:50-64`):

- creates one entry, `Prog_000.prog_bin`, compressed with **Deflated**;
- writes `data.to_bytes()` (1024 bytes, overlaid on the retained raw blob — see
  [byte fidelity](../design/byte-fidelity.md));
- writes no `FileInformation.xml` and no `.prog_info`.

**`write_lib_file(writer, &[ProgramData])`** (`file.rs:118-148`):

1. For `i` in `0..N`, write an entry `format!("Prog_{i:03}.prog_bin")`, Deflated, containing
   1024 bytes. `{i:03}` means at least 3 digits, so index 1000 becomes `Prog_1000`.
2. Then write `FileInformation.xml` (Deflated) with **exactly** these bytes. `N` is decimal
   and has no leading zeros. Every line ends in LF and there is no indentation.

```text
<?xml version="1.0" encoding="UTF-8"?>\n
<KorgMSLibrarian_Data>\n
<Product>minilogue xd</Product>\n
<Contents NumProgramData="N"/>\n
</KorgMSLibrarian_Data>\n
```

That is 149 bytes for `N` < 10 and 151 bytes for `N` = 500. `minilogue xd` comes from
`device::PRODUCT_NAME` (`device.rs:113-114`). There is no upper bound on `N` (the device
holds 500) and no `.prog_info` entry. Timestamps and permissions are whatever the `zip`
crate defaults to; the code does not set them.

### What Rust reads

**`read_prog_file(reader)`** (`file.rs:29-43`):

```text
archive = open_zip(reader)                       -> Error::Zip("failed to open archive…")
entry   = archive.by_name("Prog_000.prog_bin")   -> Error::Zip("missing Prog_000.prog_bin…")
buf     = read_to_end(entry)                     -> Error::Zip("failed to read prog_bin…"); no size cap
return ProgramData.from_bytes(buf)               ; <1024 -> PayloadTooShort; >1024 accepted, tail ignored
```

The entry name match is exact and case-sensitive, at the archive root. A single-program file
whose entry is named after its slot — logue-manager writes `Prog_{slot:03}.prog_bin`
(`logue_lib.py:682-697`) — **fails** with `Error::Zip`.

**`read_lib_file(reader) -> Vec<ProgramData>`** (`file.rs:77-108`):

```text
names = [e.name for e in archive if e.name.ends_with(".prog_bin")]   ; any directory depth
sort(names)                                                          ; byte-lexicographic
for name in names: programs.push(ProgramData.from_bytes(read(name))?) ; first failure aborts all
```

- All other entries are ignored, including `FileInformation.xml` and `.prog_info`. The
  manifest is never cross-checked.
- The original `NNN` (slot) is lost. The result is only an ordered list.
- There is no limit on the entry count or on the decompressed size.
- With 4-digit names the sort is wrong: `Prog_1000` sorts before `Prog_101`. This cannot
  happen below 1000 entries.

Errors map to `Error::Zip(String)` for archive problems and `Error::Io` for write failures
(both behind `cfg(feature = "file-formats")`, `error.rs:121-128`). Blob errors come from
[program blob](program-blob.md) parsing (`PayloadTooShort`, `InvalidMagic`, `OutOfRange`, …).
See [errors](../design/errors.md).

### Compression: Stored vs Deflated

| Tool | Writes | Reads |
|------|--------|-------|
| Rust crate | Deflated (`file.rs:52`, `file.rs:120`) | whatever `zip` supports with `deflate` (Stored, Deflated) |
| loguetools `collapse` | **Stored** (`ZipFile(output, "w")` default) | Stored, Deflated |
| logue-manager | Deflated (`ZIP_DEFLATED`) | Stored, Deflated |

Whether the Korg librarian accepts Stored entries is not proven. loguetools' Stored output is
in common use, which suggests it does. Korg-produced files are presumed Deflated, so **a reader
must inflate**. A writer can choose Stored.

### `.mnlgxdpreset`

**Not implemented in Rust.** `README.md:226-227` says ".mnlgxdpreset files from firmware v2.10+
use 448-byte program blobs … Both formats are supported", and `CLAUDE.md:97` repeats it. Both
statements are false for this codebase: no reader, writer or 448-byte path exists, and
`ProgramData::from_bytes` rejects anything under 1024 bytes. **[community]** loguetools
`patch_ident` lists blob sizes "xd, og, monologue, prologue, and kingkorg … 1024, 448, 448,
336, 320 bytes". 448 is the original minilogue/monologue size. A `.mnlgxdpreset` written by
loguetools contains ordinary 1024-byte xd `prog_bin`s plus `PresetInformation.xml`. A port
should support `.mnlgxdpreset` as "library + preset metadata", not as a different blob size.

### Size bounds

| Item | Bound |
|------|-------|
| `prog_bin` entry | exactly 1024 bytes (enforce on read; Rust accepts ≥ 1024) |
| Programs per library | 500 (device memory); Rust: unbounded |
| Entry names | `Prog_NNN.prog_bin` 17 B, `Prog_NNN.prog_info` 18 B, `FileInformation.xml` 19 B |
| Rust manifest | 149–151 bytes for N ≤ 500 |
| Stored archive, Rust-equivalent content, N = 500 | 500 × (30 + 17 + 1024) + 500 × (46 + 17) + (30 + 19 + 151) + (46 + 19) + 22 = **567,287 bytes** |

## Invariants

- Every `prog_bin` entry is exactly 1024 bytes and begins with `PROG`.
- In a written library, program `i` lives in `Prog_{i:03}.prog_bin`, `i < 500`, and the names
  sort in index order.
- `FileInformation.xml`'s `NumProgramData` equals the number of `prog_bin` entries written.
- Reading a written file returns programs equal to the inputs (Rust `PartialEq` ignores the raw
  blob; a Zig port should compare the 1024 bytes).
- A reader never trusts sizes declared in the archive more than its caller-provided buffer.
  Assert `uncompressed_size == 1024` before inflating into a `[1024]u8`.

## Errata & gotchas

| # | Item | Detail | Provenance |
|---|------|--------|------------|
| 1 | 448-byte preset claim | False for this code; 448 is og/monologue | **[rust]**, **[community]** |
| 2 | Minimal manifest | Rust omits `ProgramData` children, `.prog_info`, and the `NumPresetInformation`/`NumTune*`/`NumFavoriteData` attributes. Librarian acceptance is **unverified** | **[rust]** vs **[community]** |
| 3 | Single-program entry name | Rust reads only `Prog_000.prog_bin`. Other tools may name it by slot | **[community]** logue-manager |
| 4 | Silent truncation | `read_*` accept entries longer than 1024 bytes and drop the tail | **[rust]** |
| 5 | Whole-library failure | One bad program (for example, an unknown enum byte or `PREDSEQD`) fails `read_lib_file` entirely | **[rust]**; see [sequencer blob](sequencer-blob.md#magic-pred--sq-or-seqd) |
| 6 | Name sort beyond 999 | Fails with 4-digit names | **[rust]** edge case |

## Test vectors

Rust tests (`file.rs:150-263`) are all in-memory round trips:

| Test | Input | Expect | Lines |
|------|-------|--------|-------|
| prog round trip | default program, name "FileTest", cutoff 999 | read == written | `file.rs:156-168` |
| prog preserves fields | delay/reverb/mod FX on, BPM 2400, step 1 note 60 velocity 127 | read == written | `file.rs:170-186` |
| lib round trip | 3 programs "Prog 0".."Prog 2", cutoff 0/100/200 | 3 programs, same order, names and cutoffs | `file.rs:188-208` |
| lib empty | `[]` | `[]` (archive holds only `FileInformation.xml`) | `file.rs:210-219` |
| lib single | 1 program | equal | `file.rs:221-231` |
| missing entry | zip with only `other.txt` | `read_prog_file` errors | `file.rs:233-247` |
| invalid data | `Prog_000.prog_bin` = 100 zero bytes | errors (`PayloadTooShort`) | `file.rs:249-263` |

Exact manifest for `N = 3`, as written by Rust (149 bytes):
`<?xml version="1.0" encoding="UTF-8"?>\n<KorgMSLibrarian_Data>\n<Product>minilogue xd</Product>\n<Contents NumProgramData="3"/>\n</KorgMSLibrarian_Data>\n`.

There is no real Korg-produced archive in the repo. Obtain one (a librarian export) before
finalising the Zig writer's manifest ([test plan](../zig/test-plan.md)).

## Rust design

- There are four free functions, generic over `Read + Seek` or `Write + Seek`. They allocate
  `Vec`s and `String`s and delegate all format knowledge to `ProgramData`.
- File I/O is a thin shell around the blob codec. That separation is the essential design;
  the generic I/O traits and the `zip` crate are incidental.
- There is no shared manifest model. The manifest is a `format!` string on write and ignored
  on read.

## Zig / TigerStyle notes

- **Reading.** Zig's standard library includes ZIP archive support (`std.zip`) and DEFLATE
  decompression (`std.compress.flate`). Check the exact API names against the Zig version
  you pin; they changed between releases. If the API does not fit (for example, if it
  requires a directory to extract into), write a small central-directory walker over a
  caller-provided byte slice. Read the EOCD, iterate the central-directory records, and for
  each `*.prog_bin` entry inflate exactly 1024 bytes into a `[1024]u8`. Then check
  `crc32 == header.crc32` with `std.hash.Crc32`. Reject compression methods other than 0
  (Stored) and 8 (Deflated).
- **Writing.** Write Stored only. It needs no compressor and is a fixed byte layout. All
  integers are little-endian:

  | Record | Signature | Fixed bytes | Fields after the signature |
  |--------|-----------|-------------|----------------------------|
  | Local file header | `50 4B 03 04` | 30 + name | version needed u16 = 10 (or 20); flags u16 = 0; method u16 = 0; mod time u16; mod date u16; CRC-32 u32; compressed size u32 = 1024; uncompressed size u32 = 1024; name len u16; extra len u16 = 0; name |
  | Central directory header | `50 4B 01 02` | 46 + name | version made by u16; version needed u16; flags; method; time; date; CRC; csize; usize; name len; extra len = 0; comment len = 0; disk start u16 = 0; internal attr u16 = 0; external attr u32 = 0; local header offset u32; name |
  | End of central directory | `50 4B 05 06` | 22 | disk u16 = 0; CD disk u16 = 0; entries on disk u16; total entries u16; CD size u32; CD offset u32; comment len u16 = 0 |

  Use a fixed DOS timestamp (for example, date `0x0021` = 1980-01-01, time 0) so output is
  deterministic. That matters for golden-file tests.
- **What to write for librarian compatibility:** `Prog_NNN.prog_bin` and `Prog_NNN.prog_info`
  (`<xd_ProgramInformation><Programmer/><Comment/></xd_ProgramInformation>`) per program,
  then a full `FileInformation.xml` with all five `Num*` attributes and a `ProgramData`
  element per program. That is a strict superset of what Rust writes and matches both
  community writers.
- **Static allocation:** the caller passes the output buffer. Bound it with a comptime
  function of `program_count_max = 500` (about 570 KB for `prog_bin`-only archives; budget about
  800 KB with `prog_info` and a full manifest). See [memory budget](../zig/memory-budget.md).
  The reader returns programs into a caller-provided `[]Program` slice and fails with
  `error.TooManyPrograms` beyond its length.
- **Lenient read, strict write:** accept any entry depth, gaps in `NNN`, and both
  compression methods. Use the manifest only for cross-checks (warn, don't fail). Keep the
  `NNN` index alongside each program so slot information survives.
- Errors: `error{ ZipInvalid, ZipUnsupportedMethod, ZipCrcMismatch, ProgramEntryMissing,
  ProgramEntrySize, TooManyPrograms }` plus the blob errors from
  [program blob](program-blob.md).

```zig
// sketch
const prog_bin_size = 1024;
const program_count_max = 500;
const local_header_size = 30;
const central_header_size = 46;
const eocd_size = 22;
const prog_bin_name_len = "Prog_000.prog_bin".len; // 17

fn stored_lib_size(n: u16, manifest_len: u32) u32 {
    assert(n <= program_count_max);
    const count: u32 = n; // widen before multiplying: u16 * 1134 overflows
    const per = local_header_size + central_header_size + 2 * prog_bin_name_len + prog_bin_size;
    const name = "FileInformation.xml".len;
    return count * per + local_header_size + central_header_size + 2 * name + manifest_len +
        eocd_size;
}
comptime { assert(stored_lib_size(500, 151) == 567_287); }
```

## Open questions

- Does Korg's Sound Librarian accept files without `.prog_info` entries, with the minimal
  manifest, or with Stored compression? None of these is verified for the Rust output.
- What exact `FileInformation.xml` does Korg's own librarian write (attribute order,
  indentation, BOM, CRLF)? Readers are presumably tolerant, but golden tests need a sample.
- Do factory `.mnlgxdlib`/`.mnlgxdpreset` files contain programs with `PREDSEQD` sequencer
  magic (firmware 1.xx data)? If so, the Rust reader rejects them.
- Are `TuneScaleData`/`TuneOctData` entries (user scales and octaves, see
  [tuning](tuning.md)) ever present in xd libraries, and under what entry names?

## Sources

- `crates/minilogue-xd/src/sysex/program/file.rs:1-148` — readers and writers.
- `crates/minilogue-xd/src/sysex/program/file.rs:150-263` — tests.
- `crates/minilogue-xd/src/sysex/program/mod.rs:8-9` — feature gate.
- `crates/minilogue-xd/src/device.rs:113-114` — `PRODUCT_NAME`.
- `crates/minilogue-xd/Cargo.toml:12-16`, `:24` — features, `zip` dependency.
- `crates/minilogue-xd/src/lib.rs:70` — feature table.
- `crates/minilogue-xd/src/error.rs:121-128` — `Io`, `Zip` variants.
- `README.md:219-228`, `CLAUDE.md:96-97` — the incorrect preset claim.
- [community] gazzar/loguetools @ba0fc46: `loguetools/common.py` (`fileinfo_xml`,
  `prog_info_template_xml`, `presetinfo_xml`, `patch_ident`, `zipread_progbins`),
  `loguetools/collapse.py` (Stored writer), `loguetools/xd.py` (`favorite_template`).
- [community] irgendwatt/logue-manager @a04b2ea `logue_lib.py:663-735`
  (`create_program_zip`, `read_program_file`).
- [Community references](../sources/community-references.md).
