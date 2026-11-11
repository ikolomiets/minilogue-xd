---
title: Log
category: meta
sources: []
related:
  - index.md
  - AGENTS.md
updated: 2026-11-11
---

# Log

> This is an append-only, chronological record of wiki ingests, queries and lint passes. Each
> entry has a parseable header, so `grep "^## \[" docs/wiki/log.md | tail -5` lists the latest
> work. The format is defined in [AGENTS.md](AGENTS.md#log-format).

## [2026-10-11] ingest | Initial wiki build (schema, sources, protocol, design, zig)

- Wrote the schema ([AGENTS.md](AGENTS.md)) and the page plan. The Korg MIDI Implementation
  rev 1.01 was read from an external copy, which is cited by section and not stored in the repo.
- Six parallel agents compiled the pages below from `crates/minilogue-xd/src/**` (including the
  tests), `crates/minilogue-xd/examples/`, `crates/minilogue-xd/tests/`, `docs/design/`,
  `docs/dev/`, `docs/wayfinder/`, `docs/korg-user-guide/book.md`, and the community workbench
  implementations:
  - Foundation: [MIDI messages](protocol/midi-messages.md), [USB ports](protocol/usb-ports.md),
    [7-bit codec](protocol/korg-7bit-codec.md), [typed values](design/typed-values.md),
    [errors](design/errors.md), [transport](design/transport.md).
  - Parameters: [parameter encodings](protocol/parameter-encodings.md),
    [CC map](protocol/cc-map.md), [NRPN map](protocol/nrpn-map.md),
    [stepped parameters](design/stepped-params.md).
  - SysEx: [SysEx framing](protocol/sysex-framing.md),
    [device identity](protocol/device-identity.md), [poly chain](protocol/poly-chain.md),
    [tuning](protocol/tuning.md), [SysEx transactions](design/sysex-transactions.md).
  - Program data: [program blob](protocol/program-blob.md),
    [sequencer blob](protocol/sequencer-blob.md), [file formats](protocol/file-formats.md).
  - Global, modules and SMF: [global blob](protocol/global-blob.md),
    [user modules](protocol/user-modules.md),
    [standard MIDI file](protocol/standard-midi-file.md),
    [MIDI file export](design/midi-file-export.md).
  - Tests and docs: [testing strategy](design/testing-strategy.md),
    [examples](design/examples.md), [design docs](sources/design-docs.md),
    [Wayfinder DSL](sources/wayfinder-dsl.md).
- The coordinator wrote the [source pages](index.md#sources), [overview](overview.md),
  [architecture](design/architecture.md), and the Zig pages:
  [port overview](zig/port-overview.md), [module layout](zig/module-layout.md),
  [TigerStyle mapping](zig/tigerstyle-mapping.md),
  [dependency elimination](zig/dependency-elimination.md),
  [platform MIDI](zig/platform-midi.md) and [test plan](zig/test-plan.md).

## [2026-11-11] ingest | Coordinator pages: controller, builders, byte fidelity, errata, memory budget

- [Realtime controller](design/realtime-controller.md), from `controller.rs` and its tests: every
  setter's wire bytes. It found the bend-range NRPN mismatch (Rust `0x16`/`0x17` vs spec
  `0x20`/`0x21`) and the program-change bank encoding mismatch.
- [Builders](design/builders.md), from `builder/patch.rs` and `builder/sequence.rs`: defaults,
  clamping (NaN → 0), and the default sequencer header that is serialized in the wrong order.
- [Byte fidelity](design/byte-fidelity.md), from commit `1c16566` and `sysex/program/*`. A full scan
  confirmed that `to_bytes_with_base` writes all 156 synth bytes and all 868 sequencer bytes, and
  that only the upper 6 bits of the 34 ten-bit high bytes come from the raw base.
- [Errata](protocol/errata.md): the consolidated register (rows M, C, N, E, P, S, G, T, I, PC, U,
  F and X), linking each row to its canonical page.
- [Memory budget](zig/memory-budget.md): every comptime size, two build profiles (core,
  user_modules), static totals (≈ 22.4 KiB core) and the DIN timing budget. It found that the
  5 s default timeout cannot cover a large user-slot dump over DIN.
- [Glossary](glossary.md) and [index](index.md).
- [Rust crate](sources/rust-crate.md): extended the README and code drift table (version,
  user-module and preset claims, motion API, `CentOffset` example, licence files, examples
  list, the NRPN count in `CLAUDE.md`).
- Harmonized names and numbers across pages:
  - Input ring 16 KiB in [SysEx transactions](design/sysex-transactions.md), to match
    [transport](design/transport.md) and [platform MIDI](zig/platform-midi.md).
  - `frame_size_max` replaces `sysex_bytes_max`.
  - `program_count_max`, `decoded_len_max` and `smf_file_size_max` replace `max_*` names.

## [2026-11-11] lint | Whole wiki (47 pages)

- A script checked front matter fields, category vs folder, a single H1, `related:` paths,
  945 relative links (146 with anchors, resolved with GitHub slug rules), orphans and
  [index](index.md) coverage. Result: clean. A seeded broken link and anchor were caught.
- Fixed an anchor in [errata](protocol/errata.md) (`#readme-and-code-drift`) and a milestone
  reference for user modules (M9).
- Category `overview` → `meta` on [overview](overview.md).
- Resolved a design contradiction. NAK is `error.Nak` plus `Diagnostics.status`, as on
  [errors](design/errors.md); before this, [SysEx transactions](design/sysex-transactions.md)
  returned it as an `Outcome`. `TransactionError` now also lists `FramesUnrelatedExceeded`,
  `FrameOverflow` and `InputOverrun`.
- Unified the "bit 7 set" error tags: `WireByteInvalid` (codec and frames) and
  `DataByteInvalid` (messages and blob values) replace `DataByteHighBit` and
  `InvalidSevenBitByte`. The program-blob error set is mapped onto `BlobError || FrameError`.
- Confirmed that the Korg spec text is not stored in the repo.
- Not checked: duplicated facts across pages, and glossary coverage of every term used.
