---
title: Errata — consolidated spec, hardware and Rust disagreements
category: protocol
sources:
  - Korg minilogue xd MIDI Implementation rev 1.01 (TXT)
  - Korg minilogue xd Owner's Manual E8 (MIDI implementation chart)
  - crates/minilogue-xd/src/
  - docs/wayfinder/research/xd-midi-capabilities.md
  - README.md
  - git commits 1c16566, d465474, 6102d42, bc6c452, 213020c, 5e4b4f0
related:
  - protocol/midi-messages.md
  - protocol/cc-map.md
  - protocol/nrpn-map.md
  - protocol/parameter-encodings.md
  - protocol/program-blob.md
  - protocol/sequencer-blob.md
  - protocol/global-blob.md
  - protocol/tuning.md
  - protocol/user-modules.md
  - protocol/sysex-framing.md
  - protocol/device-identity.md
  - protocol/poly-chain.md
  - protocol/file-formats.md
  - protocol/standard-midi-file.md
  - protocol/usb-ports.md
  - design/stepped-params.md
  - design/realtime-controller.md
  - design/byte-fidelity.md
  - design/builders.md
  - design/sysex-transactions.md
  - sources/korg-midi-implementation.md
  - sources/rust-crate.md
updated: 2026-11-11
---

# Errata — consolidated spec, hardware and Rust disagreements

> This page is the register of every known disagreement between the Korg MIDI
> Implementation (rev 1.01), the device, community tools and the Rust crate. There is one
> row per issue. Detail and test vectors stay on the linked page; this page records the
> **decision the Zig port takes**. Read it before implementing any table. When a new
> disagreement is found, add a row here and a row in the subject page's "Errata & gotchas".

## Facts

### Precedence rule

Evidence ranks as follows. **[hw]** (a capture or a hardware-verified commit) beats
**[spec]**. The spec beats **[community]**, except where the spec contradicts itself; then
community consensus across independent tools decides. **[rust]** alone never decides: the
crate is a source of bugs as well as evidence. Rows with no deciding evidence are marked
**disputed**. The port implements them behind an explicit option and refuses them in strict
mode (see [realtime controller](../design/realtime-controller.md#zig--tigerstyle-notes)).

Port decision keywords:

- **spec**: follow the TXT.
- **hw**: follow the device.
- **preserve**: keep raw bytes and don't interpret them.
- **fix**: a Rust bug; do not copy it.
- **disputed**: behind an option, off by default.
- **n/a**: documentation only.

### Channel messages, CC and NRPN

| # | Topic | Spec says | Rust does | Evidence | Port | Page |
|---|---|---|---|---|---|---|
| M1 | Bank select | CC0 = 0, CC32 = bank 0~4, PC 0~99 (§1-1 `*1-1`, §2-1) | controller: CC0 = bank, CC32 = slot (`controller.rs:464-468`). The `BankSelect` type follows the spec | none; README claims an erratum without a capture | **spec**; Rust form as an option | [MIDI messages](midi-messages.md#bank-select-and-program-numbers) |
| M2 | Note Off TX velocity | device sends `8n kk 40` | `stop_note` sends `8n kk 00` | — | **spec** (send `40`) | [realtime controller](../design/realtime-controller.md#errata--gotchas) |
| M3 | Velocity-0 Note On | treated as Note Off on RX | `NoteOn` allows vel 0 | [spec] | **spec**: reject 0 in `NoteOn`; provide `NoteOff` | [MIDI messages](midi-messages.md#errata--gotchas) |
| M4 | Pitch bend −8192 | full 14-bit range | float mapping never produces wire 0 | [rust] | integer API (`u14`) | [parameter encodings](parameter-encodings.md#float--integer-mappings-controller-api) |
| M5 | Realtime TX gating | F8/FA/FC/FE sent when "En Rx Transport" is On (TABLE 1 off 18); FB never sent; FE RX ungated | not modelled | [spec] | **spec** (doc) | [MIDI messages](midi-messages.md#errata--gotchas) |
| M6 | SPP byte order | §2-2 ambiguous | standard 14-bit, LSB first | MIDI 1.0 | standard | [MIDI messages](midi-messages.md) |
| C1 | CC59 resolution | TX table: no `*1-4`; RX table: `*5-4` (10-bit) | 10-bit both ways | none | **disputed**: RX 10-bit; TX 7-bit unless the option is set | [CC map](cc-map.md#errata--gotchas) |
| C2 | Delay/reverb on-off CC | TXT rev 1.01: 93/94 | 93/94 | E8 chart says 94/95 | **spec** (TXT) | [CC map](cc-map.md#errata--gotchas) |
| C3 | CC7/CC11 | absent from TXT | not modelled | E8 lists them | **disputed** (RX only) | [CC map](cc-map.md#errata--gotchas) |
| C4 | Sync/Ring RX polarity | `*6-08/09` "0~63, 64~127 = ON, OFF" contradicts TX `*2-08/09` | 0–63 Off, 64–127 On | spec self-inconsistent | Rust reading (Off low) | [stepped params](../design/stepped-params.md#errata--gotchas) |
| C5 | Mod FX type RX bands | `*6-12` has 102 in two bands | 103–127 = User | spec typo | Rust reading | [CC map](cc-map.md#errata--gotchas) |
| C6 | Noise select RX bands | `*6-19` lists 8 bands for 4 values | 4 × 32-wide | spec copy-paste | Rust reading | [CC map](cc-map.md#errata--gotchas) |
| C7 | CC96 USER TX list | ends "…112, 128" | 127 | 128 is not 7-bit | 127 | [CC map](cc-map.md#errata--gotchas) |
| C8 | CC96 sub-type values | TX sends **band values** (0, 16, … 127; Ensemble 0, 64, 127) | `midi_file::patch_ccs` sends the raw index (`midi_file.rs:509-519`) | [spec] | **fix** | [MIDI file export](../design/midi-file-export.md#errata--gotchas) |
| C9 | CC63 placement | low 3 bits sent **first** (`*1-4`) | CC63 first | [spec] | spec; groups are indivisible | [parameter encodings](parameter-encodings.md#errata--gotchas) |
| C10 | NRPN data LSB | CC63 (`*3-1…3`), not CC38 | CC38 resets the NRPN FSM | [spec] | spec | [NRPN map](nrpn-map.md#errata--gotchas) |
| C11 | CV IN 118/119 | TX only in Modulation mode + PolyChain Master, never over USB | unconditional | [spec] | spec (doc) | [CC map](cc-map.md#errata--gotchas) |
| N1 | **NRPN addresses** (32 of 51) | names 0x01–0x0C; voice mode 0x10; noise 0x11; bend 0x20/0x21; CV 0x28–0x2C; … | shifted by +1, +4, +10, +12 or +2 by group (`param/nrpn.rs:136-189`) | no [hw] | **spec**; verify per address on hw | [NRPN map](nrpn-map.md) |
| N2 | NRPN 0x51 after-touch assign | not in `*3` (only blob byte 155) | invented address | [rust] | **fix** (drop) | [NRPN map](nrpn-map.md#errata--gotchas) |
| N3 | NRPN semantic ranges | bend ≤ 12, level ≤ 120, name idx < 12 | unchecked except in controller setters | [spec] | validate at encode | [NRPN map](nrpn-map.md#errata--gotchas) |
| N4 | 8-bit high part > 25 | 0~200 | masks to 5 bits; 32..127 aliases | [rust] | reject `high > 25` | [parameter encodings](parameter-encodings.md#errata--gotchas) |

### Enumerations

| # | Topic | Spec says | Rust does | Evidence | Port | Page |
|---|---|---|---|---|---|---|
| E1 | **Voice mode** order | `*3-5` NRPN 0–4 ARP LATCH…POLY; TABLE 2 P3 1~4 garbled ("2" twice) | enum Poly 0…ArpLatch 4; blob raw = prog + 1. The default `voice_mode_type = 3` is commented "Poly" but decodes as Chord | community: 1 ARP, 2 CHORD, 3 UNISON, 4 POLY; `quintal.rs:650` "raw 4 = ArpLatch as saved by hardware" | **disputed**; blob byte kept raw, typed view behind an option | [stepped params](../design/stepped-params.md#voicemodetype) |
| E2 | Mod FX type raw 1 | P12: 1 = CHORUS, 2 = ENSEMBLE | 1 → Chorus | `quintal.rs:701`: device saved raw 1 after setting Ensemble over CC | **disputed** | [program blob](program-blob.md#errata--gotchas) |
| E3 | Micro tuning | NRPN 0–22 + 112–123; blob 0–22 + 128–139 | dense 0–38, different names from 11 up | [spec] | **spec** | [stepped params](../design/stepped-params.md#microtuning-39-variants) |
| E4 | Assign targets (P19) | GATE TIME … DELAY DEPTH | labels wrong at every index | [spec] | **spec** | [stepped params](../design/stepped-params.md#errata--gotchas) |
| E5 | User param type | 0~2 | stored raw | factory patches contain 3 [hw] (`param/enums.rs:477-480`) | **preserve** | [program blob](program-blob.md#errata--gotchas) |
| E6 | ARP rate labels (S4) | 0 = 64th … 10 = 4th | `ArpRate` labels unrelated | loguetools: S4 off by one on hw | **disputed** (raw u8) | [sequencer blob](sequencer-blob.md#arp-and-enumerations) |
| E7 | 1-based blob enums | byte 21 is 1~4, byte 89 is 1~5 | enum program values 0-based; ±1 at the boundary | [spec] | per-field table | [stepped params](../design/stepped-params.md#errata--gotchas) |

### Program and sequencer blobs

| # | Topic | Spec says | Rust does | Evidence | Port | Page |
|---|---|---|---|---|---|---|
| P1 | 10-bit byte order | Bit column `H:0~7`, `L:0~7` | little-endian | **[hw]** `1c16566`; reverb 870 = `66 03` | **hw** | [program blob](program-blob.md#field-encodings) |
| P2 | Upper 6 bits of 10-bit high byte | unmentioned | preserved via raw base | **[hw]** `1c16566` | **preserve** | [byte fidelity](../design/byte-fidelity.md) |
| P3 | Byte 148 bits 7..4 | Reserved | `user_param_type_flags` | **[hw]** `1c16566` | **preserve** | [byte fidelity](../design/byte-fidelity.md) |
| P4 | Name padding | P1 charset lacks NUL | accepts NUL; builder pads with spaces | **[hw]** pads with NUL | **hw** | [byte fidelity](../design/byte-fidelity.md#errata--gotchas) |
| P5 | Program dump size | diagrams: 336 B (384 wire) | 1024 B (1171 wire; frames 1179/1181) | **[hw]** `1c16566`, `d465474` | **hw** | [program blob](program-blob.md#errata--gotchas) |
| P6 | Program-number bits | request "LSB 7~0, MSB 8" vs dump "LSB 6~0, MSB 7" | `lsb = n & 0x7F`, `msb = n >> 7` (`6102d42`) | logue-manager agrees | Rust form | [program blob](program-blob.md#errata--gotchas) |
| P7 | SYNC/RING blob polarity | bytes 34/35: 0 = ON | follows spec | isnotinvain, gekart, loguetools: 1 = ON | **disputed** | [program blob](program-blob.md#errata--gotchas) |
| P8 | Field typos | PORTAMENTO "0,1"; EG TARGET "H:0~7"; REVERB SUB 0~19 vs 0~17; DRY WET "0~1024" | sensible readings | [spec] typos | Rust readings | [program blob](program-blob.md#errata--gotchas) |
| S1 | **Sequencer header order** | magic, active steps, BPM, len, res, swing, gate | rotated: BPM, len, res, swing, gate, active | **[hw]** capture + community | **spec/hw**; Rust is wrong | [sequencer blob](sequencer-blob.md#header--what-rust-actually-does-bug) |
| S2 | `SEQD` magic | S1: old programs carry `SEQD`, converted on load | rejects anything but `PREDSQ` | **[hw]** `0x4C` capture shows `PREDSEQD` | accept both; **preserve** | [sequencer blob](sequencer-blob.md#magic-pred--sq-or-seqd) |
| S3 | BPM high byte | only bits 3..0 (`H:0~3`) | full u16 | [spec] | mask on read; **preserve** the rest | [sequencer blob](sequencer-blob.md#errata--gotchas) |
| S4 | Step 16 range text | "426~1021" | — | actually 970~1021 | n/a | [sequencer blob](sequencer-blob.md#errata--gotchas) |
| S5 | Swing encoding | −75~+75, no encoding given | +75 offset | community; capture 75 ≈ 0 | +75 offset | [sequencer blob](sequencer-blob.md#errata--gotchas) |
| S6 | Motion slot u16 | high = param ID; low bits 0/1 = on/smooth | single `parameter_id` | [spec], [hw] | split fields | [sequencer blob](sequencer-blob.md#motion-slot-parameter-s2) |
| S7 | Stored vs edit ARP bytes | — | — | same program: `00 00` stored, `37 05` in edit buffer [hw] | **preserve** | [sequencer blob](sequencer-blob.md#errata--gotchas) |

### Global, tuning, identity, poly chain, user modules

| # | Topic | Spec says | Rust does | Evidence | Port | Page |
|---|---|---|---|---|---|---|
| G1 | Global dump size | diagram 32 B (37 wire) | 63 B (72 wire; frame 80) | TABLE 1 spans 63 | TABLE 1 | [global blob](global-blob.md#errata--gotchas) |
| G2 | Tune / transpose | signed values | offsets 50 / 12 | [spec] | **disputed** until hw | [global blob](global-blob.md#errata--gotchas) |
| G3 | Oscilloscope flag | 1 = Disable | inverted | [spec] | **spec** | [global blob](global-blob.md#errata--gotchas) |
| G4 | Favourites | 0~499 in one byte | — | cannot fit | **disputed** (preserve) | [global blob](global-blob.md#errata--gotchas) |
| T1 | MTS bulk checksum | XOR of bytes 1..405 including `7E` | XOR from byte 2, always off by `0x7E` (`tuning.rs:314-319`) | MMA MTS | **fix** | [tuning](tuning.md#errata--gotchas) |
| T2 | MTS single-note change | realtime `7F` | `7E` (`tuning.rs:399, 435`) | MMA MTS | **fix** | [tuning](tuning.md#errata--gotchas) |
| T3 | Tuning dumps 0x44/0x45 | carry scale number `ss` (7F = edit) + raw data | omit `ss`, 7-bit encode | [spec] | **spec** | [tuning](tuning.md#errata--gotchas) |
| T4 | Tuning message typos | header, codes 0x18/0x19/0x48/0x49, "Func=1C", labels | — | [spec] | n/a | [tuning](tuning.md#errata--gotchas) |
| T5 | TABLE 4 octave byte | signed (0–23, 116–127 = −12…) | unsigned | [spec] | **spec** | [tuning](tuning.md#errata--gotchas) |
| T6 | MTS RX condition | only while the USER SCALE/OCTAVE edit page is shown | — | user guide | n/a (doc) | [tuning](tuning.md) |
| I1 | Search reply | channel/echo bytes | swapped; echo not masked | [spec] | **spec** | [device identity](device-identity.md#errata--gotchas) |
| I2 | Identity request | can address 7F | can't send 7F; IDs unchecked | [spec] | **spec** | [device identity](device-identity.md#errata--gotchas) |
| PC1 | Poly-chain pitch | 16-bit (`0000 00hh`) | allows 21 bits (`poly_chain.rs:96`) | [spec] | **spec** | [poly chain](poly-chain.md#errata--gotchas) |
| U1 | User-module replies | start with a raw prefix (`48 mm 00 …`) | not handled; would 7-bit decode | [spec] + community | **spec** | [user modules](user-modules.md#errata--gotchas) |
| U2 | TABLE 6 layout | — | off by one from offset 5 | [spec] | **spec** | [user modules](user-modules.md#errata--gotchas) |
| U3 | Module ID 5 | valid | rejected | [spec] | **spec** | [user modules](user-modules.md#errata--gotchas) |
| U4 | TABLE 5 size | 8~9 | 9 | NTS-1 captures: 12 | accept ≥ 8, **preserve** | [user modules](user-modules.md#errata--gotchas) |
| U5 | API version reply 0x47 | raw, not 7-bit | no parser | [spec] | **spec** | [user modules](user-modules.md) |

### Framing, transactions, transport

| # | Topic | Spec / reality | Rust does | Evidence | Port | Page |
|---|---|---|---|---|---|---|
| F1 | Status codes | NOTE 2: 0x23 ACK; 0x24, 0x26–0x2F NAK; 0x25 unassigned | matches | [spec] | spec | [SysEx framing](sysex-framing.md) |
| F2 | Malformed program write | — | — | **[hw]** answers `0x26` DATA FORMAT ERROR | expect a NAK, not a timeout | [SysEx framing](sysex-framing.md#errata--gotchas) |
| F3 | Stream assembly | F8–FF may interleave; F0 restarts a frame | classifies chunks by first byte; mid-frame F0 appended (regression `5e4b4f0`); no size cap | MIDI 1.0 | **fix**: byte-wise assembler | [SysEx transactions](../design/sysex-transactions.md) |
| F4 | Reply validation | header/channel/function must match | any complete frame accepted; ACK to a request → `UnexpectedResponse` | [rust] | **fix** | [SysEx transactions](../design/sysex-transactions.md) |
| F5 | USB port roles | no Korg doc names ports | send SOUND, listen KBD/KNOB | **[hw]** `bc6c452`, `213020c` (macOS) | hw | [USB ports](usb-ports.md#errata--gotchas) |

### Files

| # | Topic | Claim | Rust does | Evidence | Port | Page |
|---|---|---|---|---|---|---|
| X1 | 448-byte presets | README: `.mnlgxdpreset` 448-byte blobs | not implemented | 448 is og/monologue [community] | n/a until a real file exists | [file formats](file-formats.md#errata--gotchas) |
| X2 | `.mnlgxdunit` / user-module upload | README `:155-156` | not implemented | — | M9 adds slot transactions; `.mnlgxdunit` out of scope | [user modules](user-modules.md) |
| X3 | Minimal `.mnlgxdlib` manifest | librarian writes `ProgramData`, `.prog_info`, `Num*` attributes | omits them | acceptance unverified | write the full manifest | [file formats](file-formats.md#errata--gotchas) |
| X4 | Oversized entries | — | tail silently dropped | [rust] | **fix**: reject | [file formats](file-formats.md#errata--gotchas) |
| X5 | SMF SysEx | F7 terminator | not appended | SMF 1.0 | **fix** | [standard MIDI file](standard-midi-file.md#errata--gotchas) |

## Invariants

1. Every row names exactly one port decision. **Disputed** rows are never silent defaults.
2. Every **preserve** row has a [byte fidelity](../design/byte-fidelity.md) test: the bytes
   survive `parse → serialize`.
3. Every **fix** row has a regression vector in the Zig suite that the Rust crate would
   fail ([test plan](../zig/test-plan.md)).
4. A row moves from **disputed** to **hw** only with a cited capture: a commit, a file in
   `workbench/` or a community dump.

## Errata & gotchas

- `CLAUDE.md` says "29 NRPN parameters". The spec has 61 addresses; Rust has 51 variants
  over 62 addresses.
- README drift (version 0.1 vs crate 0.2.0, the `no_std` claim, preset/unit support) is
  tracked on [rust crate](../sources/rust-crate.md#readme-and-code-drift). It is not
  a protocol erratum.
- The wayfinder research warns "do not apply a universal offset" to NRPN addresses
  (`xd-midi-capabilities.md:52`). Some groups agree with the spec. Port per address.

## Test vectors

Each row's vectors live on its linked page. The highest-value **hardware** captures are:

| Capture | What it settles | Page |
|---|---|---|
| 10-bit pair `66 03` = 870 at bytes 107–108 | P1 | [program blob](program-blob.md#field-encodings) |
| 1181-byte `0x4C` replies (`d465474`) | P5 | [SysEx framing](sysex-framing.md) |
| `FF FF D8 04 10 01 4B 48` at abs 162–169 | S1 | [sequencer blob](sequencer-blob.md#header--what-rust-actually-does-bug) |
| `PREDSEQD` in a stored-program reply | S2 | [sequencer blob](sequencer-blob.md#magic-pred--sq-or-seqd) |

## Rust design

The crate records errata in three places: README "Known documentation errata", doc
comments and commit messages. It has no single register, and several README claims are
stale. Rust code is evidence of **behaviour**, not of the device.

## Zig / TigerStyle notes

- Encode the precedence as data. Each comptime parameter-table row carries
  `evidence: enum { spec, hw, community, disputed }`. A `strict` build option refuses to
  encode `disputed` rows (`error.DisputedMapping`).
- Keep this page and the table in sync. A test can parse this page's row IDs (M1, N1, …)
  and assert that each ID appears in the Zig source as a comment or table tag.

## Open questions

Hardware captures needed, ranked by impact:

1. N1 NRPN addresses
2. M1 bank select
3. E1 voice mode
4. P7 SYNC/RING polarity
5. E2 Mod FX type
6. C1 CC59
7. G2 global tune/transpose
8. E6 ARP rate

## Sources

- Korg minilogue xd MIDI Implementation rev 1.01: §1-1, §1-2, §2-1, §2-2, `*1`–`*7` notes,
  TABLEs 1–7, NOTE 1, NOTE 2, notes P1–P24, S1–S4. See
  [Korg MIDI implementation](../sources/korg-midi-implementation.md).
- Owner's Manual E8, MIDI implementation chart (p.68).
- `docs/wayfinder/research/xd-midi-capabilities.md:48-56`.
- Commits `1c16566`, `d465474`, `6102d42`, `bc6c452`, `213020c`, `5e4b4f0`.
- The subject pages linked in each row, which hold the file and line citations.
