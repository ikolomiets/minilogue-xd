---
title: Design and development documents
category: source
sources:
  - docs/design/index.md
  - docs/design/06-final/0001-minilogue-xd-rust-library-project-plan.md
  - docs/design/05-active/0002-midi-file-export-implementation-plan.md
  - docs/dev/0001-phase-1-implementation-plan-minilogue-xd-foundation.md
  - docs/dev/0002-phase-2-implementation-plan-parameter-layer.md
  - docs/dev/0003-phase-3-implementation-plan-sysex-layer.md
  - docs/dev/0004-phase-4-implementation-plan-high-level-api.md
  - odm.toml
  - README.md
  - CLAUDE.md
  - crates/minilogue-xd/Cargo.toml
related:
  - overview.md
  - design/architecture.md
  - design/testing-strategy.md
  - design/midi-file-export.md
  - protocol/errata.md
  - protocol/nrpn-map.md
  - protocol/program-blob.md
  - protocol/tuning.md
  - protocol/user-modules.md
  - protocol/file-formats.md
  - sources/rust-crate.md
  - sources/korg-midi-implementation.md
  - sources/community-references.md
  - zig/port-overview.md
updated: 2026-10-11
---

# Design and development documents

> `docs/design/` holds two numbered design documents, managed by the ODM tool:
>
> - **0001**, the four-phase, 24-milestone project plan (state *Final*);
> - **0002**, the Standard MIDI File export plan (state *Active*, though already implemented).
>
> `docs/dev/` holds four per-phase implementation plans. All were written in April 2026, before
> or during implementation. The code then diverged in documented ways:
>
> - Hardware testing reversed several plan assumptions, such as the endianness of 10-bit blob
>   fields.
> - About a dozen planned features were never built, including `.mnlgxdpreset`/`.mnlgxdunit`,
>   `no_std`, `MIDI_COVERAGE.md`, motion-sequence builders, and the user-module transactions.
>
> Read these documents for intent and rationale. Read the code and [hw] evidence for facts.

## Facts

### Inventory

| Path | Lines | Front matter | Author | State | Dates | Summary |
|------|------:|--------------|--------|-------|-------|---------|
| `docs/design/index.md` | 20 | none | generated | — | — | "Oxur Design Document Index", automatically generated, "Do not edit manually". Lists documents by number and by state |
| `docs/design/06-final/0001-minilogue-xd-rust-library-project-plan.md` | 816 | YAML `number: 1`, `state: Final`, `version: 1.0`, `tags: [change-me]` | "Claude Code" | Final | created 2026-04-05, updated 2026-04-06 | Project plan: 4 layers, 24 milestones, cross-cutting rules |
| `docs/design/05-active/0002-midi-file-export-implementation-plan.md` | 135 | YAML `number: 2`, `state: Active` | Duncan McGreggor | Active | 2026-04-06 | SMF format-0 writer `MidiFileBuilder` + `export_midi` example |
| `docs/dev/0001-phase-1-implementation-plan-minilogue-xd-foundation.md` | 224 | none | — | — | — | Milestones 1.1–1.6: scaffold, codec, channel and system messages, transport |
| `docs/dev/0002-phase-2-implementation-plan-parameter-layer.md` | 234 | none | — | — | — | Milestones 2.1–2.4: stepped enums, 10-bit encoding, CC map, NRPN map |
| `docs/dev/0003-phase-3-implementation-plan-sysex-layer.md` | 255 | none | — | — | — | M3.1–M3.8: frames, global blob, program blob, files, tuning, user modules, poly chain |
| `docs/dev/0004-phase-4-implementation-plan-high-level-api.md` | 205 | none | — | — | — | M4.1–M4.6: controller, transactions, builders, integration tests, polish |

### ODM document workflow

[rust] `odm.toml:1-14` configures the ODM design-document tool. The index title, "Oxur Design
Document Index", suggests the name expands to *Oxur Design-doc Manager*; that expansion is
inferred.

| Key | Value | Meaning |
|-----|-------|---------|
| `docs_directory` | `./docs/design` | Numbered design docs live here, in state subdirectories |
| `dev_directory` | `./docs/dev` | Implementation plans; no state directories and no front matter |
| `preserve_dustbin_structure` | `true` | Removed docs keep their state path in a "dustbin" |
| `auto_stage_git` | `true` | The tool `git add`s the files it moves or edits |

- **Comment mismatch.** The file's header comment says "ODM Configuration for the ECL/Fabryk
  Workspace" (`odm.toml:1`). It was copied from another project.
- **One directory per state.** A document's state is both its `state:` front-matter field and
  its directory, using numbered names that sort in lifecycle order. Three are observed:
  `02-under-review`, `05-active`, and `06-final`. Other numbers are presumably used by the tool
  but are not present.
- **Transitions are renames.** Commit `dc7fc3e` ("Updated docs.") did all of the following:
  - moved 0001 from `02-under-review/` to `06-final/` (git `R099`);
  - added 0002 under `05-active/`;
  - regenerated `index.md`.

  0001 was created in `02-under-review/` in commit `6cb0642`.
- **Stale path.** `CLAUDE.md:100-101` still points at the old
  `docs/design/02-under-review/…` path.

### 0001 — project plan

**Goal and sources.** Its stated goal is "100% coverage of the Korg Minilogue XD MIDI
Implementation (Revision 1.01)". It names four "workbench" reference implementations:
logue-sdk, gekart's `mnlgxd.py`, isnotinvain's minilogue-xd-util, and gazzar's loguetools
(`…0001…:17-19`). See [community references](community-references.md).

**Architecture** (`:22-40`). Four layers: Foundation → Parameter → SysEx → High-level API. Each
milestone is "scoped to be executable by Claude Code in a single context window" (`:43`). The
as-built layering is on [architecture](../design/architecture.md).

| Phase | Milestones (line) | Key planned deliverables |
|-------|-------------------|--------------------------|
| 1 Foundation | 1.1 scaffold (`:53`), 1.2 codec (`:68`), 1.3 channel TX (`:87`), 1.4 channel RX (`:109`), 1.5 realtime/common (`:129`), 1.6 transport (`:146`) | Codec acceptance: "Round-trip property holds for 1000 random inputs" (`:83`). Transport with a callback `listen` API (`:154`) |
| 2 Parameters | 2.1 enums (`:174`), 2.2 10-bit (`:219`), 2.3 CC (`:240`), 2.4 NRPN (`:264`) | Typed enums with TX/RX tables; 10-bit = CC63 low 3 bits + CC high 7 bits |
| 3 SysEx | 3.1 frames (`:302`), 3.2 global (`:328`), 3.3 synth bytes 0–155 (`:369`), 3.4 sequencer 156–1023 (`:422`), 3.5 files (`:465`), 3.6 tuning (`:494`), 3.7 user modules (`:524`), 3.8 poly chain (`:558`) | `.mnlgxdpreset` with 448-byte blobs (`:485`); `CentOffset(f32)` (`:501`); `.mnlgxdunit` (`:551`) |
| 4 High-level | 4.1 controller (`:583`), 4.2 transactions (`:618`), 4.3 patch builder (`:645`), 4.4 sequence builder (`:681`), 4.5 integration tests (`:709`), 4.6 polish (`:737`) | `send_user_slot` (`:637`); `TransactionError` (`:639`); `MockMidiPair` (`:641`, `:722`); `step(…, GateTime)` (`:694`); `tests/integration/` (`:715`); `CHANGELOG.md` (`:744`); `MIDI_COVERAGE.md` (`:745`); `std`/`no_std` (`:750`) |

A summary table of all milestones is at `:760-791`.

**Cross-cutting rules** (`:793-816`):

- **The workbench is ground truth.** Validate against the Python implementations and real
  files, not only the spec (`:801-806`).
- **Known errata,** from KnobKraft Orm (`:808-812`):
  - bank/program-number encoding differs from the spec;
  - `.mnlgxdpreset` from firmware 2.10+ uses 448-byte blobs;
  - the `SEQD` → `SQ` header backward-compatibility path must be parsed.
- **Strict dependency order** between milestones (`:814`).
- **Test-first** for the codec and 10-bit encoding, "the most failure-prone components"
  (`:816`).

### 0002 — MIDI file export plan

| Item | Plan | As built (`crates/minilogue-xd/src/midi_file.rs`) |
|------|------|---------------------------------------------------|
| Dependencies | "No external dependencies needed" (`…0002…:21`) | None added [rust] |
| API (`:43-56`) | `new`, `channel`, `ticks_per_quarter`, `track_name`, `note`, `cc`, `program_change`, `pitch_bend`, `sysex`, `patch_ccs`, `build`, `write_to` | All present (`:94-185`, `:319`, `:858`, `:914`). Added: `ten_bit_cc`, `ten_bit_cc_f32`, `stepped_cc`, `on_off_cc` (`:220-263`); `set_cutoff`, `set_resonance`, `set_lfo_rate`, `set_lfo_int`, `set_delay_dry_wet`, `set_reverb_dry_wet` (`:278-303`); `legend` (`:846`) |
| Track layout (`:76-87`) | Track name, then tempo, then time signature | Tempo, then time signature, then track name, as observed in `examples/berlin_school.mid` |
| Note off | `8n kk 40` | `8n kk 40` (`midi_file.rs:138`) |
| Example | `export_midi` with no `required-features` (`:97-101`) | As planned (`Cargo.toml:49-50`) |
| Tests | "~25" (`:112-123`) | 28 `#[test]` |
| State | Active | Implemented in `ec1222b` (after `dc7fc3e` filed the plan); state never advanced to Final |

See [MIDI file export](../design/midi-file-export.md) and
[standard MIDI file](../protocol/standard-midi-file.md).

### Dev plans — notable decisions and rationale

| Plan | Decision / claim | Line | Outcome in code |
|------|------------------|------|-----------------|
| Phase 1 | "Authoritative MIDI spec" is `workbench/minilogue-xd-util/assets/minilogue_xd__MIDIImp.txt` (gitignored) | `dev/0001…:7` | Same document as Korg rev 1.01; see [Korg MIDI implementation](korg-midi-implementation.md) |
| Phase 1 | proptest with 1000 random codec round trips | `dev/0001…:84` | 256 (proptest default); see [testing strategy](../design/testing-strategy.md) |
| Phase 1 | `MidiInput::listen(callback)` | `dev/0001…:175` | Replaced by polling `receive(timeout)` (`transport.rs:16-21`) |
| Phase 1 | Verify with `--no-default-features --features std` | `dev/0001…:220` | `std = []` has no effect (`Cargo.toml:14`) |
| Phase 2 | TX/RX asymmetry: send one canonical value, accept a band | `dev/0002…:28-58` | Implemented ([stepped params](../design/stepped-params.md)) |
| Phase 2 | `TenBitParam`: CC63 carries bits 0–2, the CC bits 3–9; CC63 may precede different target CCs | `dev/0002…:95-102` | Implemented |
| Phase 2 | NRPN addresses numbered **sequentially**: names (0, 0)–(0, 11), voice mode (0, 12), … transpose (0, 59), aftertouch (0, 60), master volume (1, 0) | `dev/0002…:180-189` | Code is neither this nor the spec; see Errata |
| Phase 2 | NRPN 4-state FSM: Idle → HaveMsb → Addressed → HaveDataLsb → resolved on CC6 | `dev/0002…:197` | Implemented; reset on an unrelated CC |
| Phase 3 | The spec's 336-byte program dump is a documentation error; the blob is 1024 bytes, "confirmed by the workbench Python code" | `dev/0003…:7` | 1024 **[hw]** (`1c16566`) |
| Phase 3 | `zip = "2"`, optional | `dev/0003…:58` | `zip = "8.5"`, optional (`Cargo.toml:24`) |
| Phase 3 | `read_10bit` big-endian H:L | `dev/0003…:85` | **Little-endian**, fixed in `1c16566` **[hw]** |
| Phase 3 | Favorites endianness "Need to verify" | `dev/0003…:112` | See [global blob](../protocol/global-blob.md) |
| Phase 3 | Global dump: spec says 32 bytes, TABLE 1 is 63; "Likely 63 is correct" | `dev/0003…:116` | 63 |
| Phase 3 | Voice mode is stored 1–4 in the blob ("need offset") | `dev/0003…:133` | `raw − 1`; contested, see Errata |
| Phase 3 | `PRED` (156–159), `SQ` (160–161), backward compatibility for `SEQD`; BPM u16 LE 100–3000 | `dev/0003…:148-149` | `PREDSQ` magic required; no `SEQD` path |
| Phase 3 | `.mnlgxdpreset` "likely synth params only" | `dev/0003…:179` | Not implemented |
| Phase 3 | `equal_temperament`, `just_major`, `pythagorean` | `dev/0003…:197` | Only `equal_temperament` (`sysex/tuning.rs:130`, `:195`) |
| Phase 3 | "~800+ tests expected" | `dev/0003…:250` | — |
| Phase 4 | Float → 10-bit via `(v * 1023.0).round()`; 31 high-res setter methods | `dev/0004…:37-43` | Implemented; out of range is an error |
| Phase 4 | Transactions borrow output + input, 5 s default timeout, `with_timeout` | `dev/0004…:64-74` | Implemented (`sysex/transaction.rs:48`) |
| Phase 4 | `tests/cc_coverage.rs` (~5 tests) and others | `dev/0004…:133` | 4 integration files, 27 tests |
| Phase 4 | "Grand total across all phases: 1127 tests" | `dev/0004…:193-205` | ~1,133 unit + 27 integration + 12 doctests at runtime |

### Planned versus implemented

| Planned (source) | Status in code [rust] |
|------------------|------------------------|
| Callback `listen` input (0001 `:154`; dev/0001 `:175`) | Polling `MidiInput::receive(timeout)` |
| `MockMidiPair` (0001 `:641`, `:722`) | Separate `MockOutput` + `MockMidiInput` (`transport.rs:48-133`) |
| `TransactionError` enum (0001 `:639`) | Variants live in the crate error type; see [errors](../design/errors.md) |
| User-module transactions such as `send_user_slot` (0001 `:637`) | Missing. Only frame builders and parsers exist: requests `0x17`–`0x1E` and reply parsers for info, slot status, and slot data (`sysex/user_module.rs:375-529`). There is no API-version reply (`0x47`) parser and no builder to *send* USER SLOT DATA (`0x4A`). See [user modules](../protocol/user-modules.md) |
| `.mnlgxdpreset` (448-byte) and `.mnlgxdunit` (0001 `:485`, `:551`) | Missing; only `.mnlgxdprog`/`.mnlgxdlib` read/write (`sysex/program/file.rs:29`, `:50`, `:77`, `:118`). See [file formats](../protocol/file-formats.md) |
| `load_prog_file(path)` / `load_lib_file` | Reader/writer based: `read_prog_file(reader)` etc. |
| `CentOffset(f32)` (0001 `:501`) | `struct CentOffset { semitone: u8, fraction: u16 }` (`sysex/tuning.rs:33-38`). See [tuning](../protocol/tuning.md) |
| `just_major`, `pythagorean` (dev/0003 `:197`) | Missing |
| `SequenceBuilder::step(…, GateTime)`, motion slots/steps (0001 `:694`) | `step(index, note, velocity)`, `step_poly`, and setters only (`builder/sequence.rs:38-134`); no motion API |
| `SEQD` → `SQ` compatibility (0001 `:812`; dev/0003 `:148`) | Missing; the `PREDSQ` magic is required |
| `tests/integration/`, fixtures (0001 `:715`) | Flat `tests/*.rs`; no fixture files |
| `CHANGELOG.md`, `MIDI_COVERAGE.md` (0001 `:744-745`) | Neither file exists |
| `std` feature enabling `no_std` (0001 `:750`) | `std = []`; no `#![no_std]` anywhere; `thiserror`, `Vec`, and `String` are used unconditionally |
| 1000 proptest cases | 256 |
| `crc32fast` optional (with user modules) | Non-optional (`Cargo.toml:21`); `bitflags` declared but unused |

## Invariants

These rules from the plans still hold and are worth carrying into the port.

- Each layer depends only on layers below it (0001 `:814`).
- Codec and parameter-encoding code is written test-first (0001 `:816`).
- When the spec, files, and hardware disagree, hardware and real files win (0001 `:801-812`;
  `CLAUDE.md:85-86`).

## Errata & gotchas

### Spec versus code (cross-checked for this page)

Tags: **[spec]** = Korg MIDI Impl rev 1.01; **[rust]** = code; **[hw]** = hardware-confirmed
commit. Canonical entries live on [errata](../protocol/errata.md).

| Topic | [spec] | [rust] | Evidence / verdict |
|-------|--------|--------|--------------------|
| Program dump size | MIDI EXCLUSIVE FORMAT (4), (5): "384Bytes (7bit) -> 336Bytes (8bit)"; TABLE 2 runs to offset 1023 | 1024 bytes → 1,171 packed | **[hw]** 1024 (`1c16566`, `sysex_roundtrip_diag`). The diagram is stale |
| Global dump size | Format (6): "37Bytes (7bit) -> 32Bytes (8bit)"; TABLE 1 is 63 bytes | 63 | Plan chose 63 (dev/0003 `:116`); no [hw] commit cited |
| Program number in request vs dump | Format (2): LSB bits 7~0 + MSB bit 8. Format (5): LSB bits 6~0 + MSB bit 7 | 7-bit split for both: `n & 0x7F`, `n >> 7` (`sysex/program/mod.rs:225-228`) | Code follows (5), which is 7-bit safe; commit `6102d42` |
| Bank select | §1-1/§2-1: CC0 = 0, CC32 = bank 0~4, then program change 0~99 | `program_change` sends CC0 = bank, CC32 = slot, PC = slot (`controller.rs:460-469`) | Conflict; `README.md:224` calls it an erratum, but no capture exists. `BankSelect` itself validates LSB ≤ 4 (`message/channel.rs:485-490`), which is inconsistent with the controller |
| NRPN addresses | *3 table: names `0x01`–`0x0C`, voice mode `0x10`, selects `0x11`–`0x13`, …, bend `0x20`/`0x21`, CV `0x28`–`0x2C`, LFO `0x34`–`0x36` | names `0x00`–`0x0B`, voice mode `0x0C`, selects `0x0D`–`0x0F`, …, bend `0x16`/`0x17`, CV `0x1C`–`0x20`, LFO `0x33`–`0x35` (`param/nrpn.rs:136-189`). Agrees from `0x30`–`0x32`, `0x3F`, `0x40`–`0x4D`, `0x50`, `0x7F` | Neither spec nor plan (plan was fully sequential). No [hw] evidence. See [NRPN map](../protocol/nrpn-map.md) |
| Voice-mode value | *3-5 (NRPN): 0 ARP LATCH … 4 POLY. TABLE 2 byte 21 *note P3: range 1~4, "1 ARP, 2 CHORD, 2 UNISON, 3 POLY" (typo) | `raw − 1` against Poly 0 … ArpLatch 4. The code comment says 3 = Poly | Three-way conflict. `examples/quintal.rs:650` claims raw 4 = ArpLatch "as saved by hardware" |
| Mod-FX type value | TABLE 2 byte 89 *note P12: 1 CHORUS, 2 ENSEMBLE, 3 PHASER, 4 FLANGER, 5 USER | raw 1 = Chorus | Agrees with the spec. `examples/quintal.rs:701` claims raw 1 = Ensemble "as saved by hardware" |
| CC59 voice-mode depth | §1-1 TX: 7-bit; §2-1 RX: *5-4 (10-bit with CC63) | 10-bit | Code follows RX |
| CC93/94 | Delay on/off 93, reverb on/off 94 (§1-1, §2-1) | 93/94 | Agree. The E8 owner's-manual edition lists 94/95 |
| `SEQD` header | *note S1: old 4-byte `SEQD` replaced by `SQ` + 2 bytes | Not handled | Planned, not built |
| Status codes | NOTE 2: ACK `0x23`; NAK `0x24`, `0x26`–`0x2F` | Same | Agree |

### README / CLAUDE.md versus code

| Doc claim | Location | Reality [rust] |
|-----------|----------|----------------|
| `minilogue-xd = "0.1"` | `README.md:26`, `:33` | Crate is `0.2.0` (`Cargo.toml:3`) |
| `xd.play_note(60, 100)` | `README.md:67` | Takes `U7` values |
| `PatchBuilder::new().name("Acid Bass").… .delay(DelaySubType::Tape, …).build()?` | `README.md:84-90` | `name` returns `Result<Self>` (`builder/patch.rs:56`); `delay` has a different signature (`:180`); `build` is infallible (`:221`) |
| `minilogue_xd::load_lib_file("factory.mnlgxdlib")` | `README.md:97` | No such function; use `read_lib_file(reader)` |
| `SequenceBuilder` example | `README.md:109-121` | Different methods (`builder/sequence.rs:38-134`) |
| `scale.set_note(64, CentOffset(-14.0))` | `README.md:133-135` | `CentOffset` is a struct; no `set_note` |
| `.mnlgxdpreset`, `.mnlgxdunit` supported; "Both formats are supported" (448/1024) | `README.md:156`, `:226-227` | Not implemented |
| `std` off → `no_std` | `README.md:203`; `src/lib.rs:71` | Not real |
| `MIDI_COVERAGE.md` documents the mapping | `README.md:269` | File does not exist |
| `LICENSE-MIT`, `LICENSE-APACHE` | `README.md:279-280` | Only `LICENSE` exists |
| Plan path `02-under-review/…` | `CLAUDE.md:100-101` | Moved to `06-final/` |
| Planned module list | `CLAUDE.md` architecture section | Omits `device`, `connection`, `midi_file` |

## Rust design

The plans explain *why* the crate looks the way it does:

- The layered modules came from the phase split.
- Typed enums with TX/RX tables came from Phase 2's asymmetry analysis.
- `raw_blob` preservation and little-endian helpers came from hardware fixes that contradicted
  Phase 3.

Where plan and code differ, the code reflects hardware evidence or simplification. The plan
does not.

## Zig / TigerStyle notes

- Do not port from the plans. Port from the wiki's protocol pages, which record the as-built
  and [hw] behaviour.
- Treat the "planned, not built" items as an explicit backlog with an evidence requirement:
  `.mnlgxdpreset` 448-byte blobs, `SEQD` compatibility, user-module send, and motion-sequence
  builders. The port milestones on [port overview](../zig/port-overview.md) should name them
  as optional.
- Replace `MIDI_COVERAGE.md` with a comptime table that maps every spec row to its encoder and
  decoder, plus a test that iterates the table. Coverage then becomes a build-time fact, not a
  document claim.
- The "1000 random inputs" acceptance becomes exhaustive tests where the domain is small, and
  seeded loops with a fixed iteration bound where it is not. See
  [testing strategy](../design/testing-strategy.md).
- There is no `std` feature to mirror. In Zig, keep the core free of I/O and allocation by
  construction. Keep platform MIDI in a separate module ([module layout](../zig/module-layout.md)).

## Open questions

- Is the voice-mode byte-21 mapping `raw − 1 → {Poly, Unison, Chord, Arp, ArpLatch}`, the
  comment's `1 Arp, 2 Chord, 3 Poly, 4 ArpLatch`, or something else? This needs a panel capture
  for each mode.
- Bank select: does the XD accept `CC0 = bank` (code) or `CC32 = bank` (spec)? This needs a
  capture of the XD's own transmitted program change with the panel's bank set to 2 or more.
- NRPN addresses: which side is right for addresses `0x00`–`0x2C`? Wayfinder issue 0013 owns
  the evidence policy ([wayfinder DSL](wayfinder-dsl.md)).
- Should 0002 be moved to `06-final` now that it is implemented?
- Is the 448-byte `.mnlgxdpreset` claim (KnobKraft) still relevant to current firmware?

## Sources

- `docs/design/index.md:1-20`.
- `docs/design/06-final/0001-minilogue-xd-rust-library-project-plan.md:1-816` — line numbers cited inline.
- `docs/design/05-active/0002-midi-file-export-implementation-plan.md:1-135`.
- `docs/dev/0001-phase-1-implementation-plan-minilogue-xd-foundation.md:5-7`, `:84`, `:175`, `:220`.
- `docs/dev/0002-phase-2-implementation-plan-parameter-layer.md:28-102`, `:180-197`.
- `docs/dev/0003-phase-3-implementation-plan-sysex-layer.md:7`, `:58`, `:85`, `:112-149`, `:179`, `:197`, `:250`.
- `docs/dev/0004-phase-4-implementation-plan-high-level-api.md:37-74`, `:133`, `:193-205`.
- `odm.toml:1-14`; `CLAUDE.md:85-86`, `:100-101`; `README.md` lines cited inline.
- `crates/minilogue-xd/Cargo.toml:1-28`; source files cited inline.
- Korg MIDI Impl rev 1.01 §1-1, §2-1, *3 NRPN table, *3-5, MIDI EXCLUSIVE FORMAT (2), (4)–(6),
  NOTE 2, TABLE 1, TABLE 2 *note P3, *note P12, *note S1.
- Git: `6cb0642`, `dc7fc3e`, `ec1222b`, `1c16566`, `6102d42`, `7e09f4a`.
