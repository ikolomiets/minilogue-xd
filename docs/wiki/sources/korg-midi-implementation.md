---
title: Source — Korg minilogue xd MIDI Implementation rev 1.01
category: source
sources:
  - https://cdn.korg.com/us/support/download/files/5227b0b2f7c1da5acd0911ac3e8c4f38.txt
  - https://www.korg.com/us/support/download/manual/0/811/4440/
  - docs/wayfinder/research/xd-midi-capabilities.md
related:
  - protocol/errata.md
  - protocol/midi-messages.md
  - protocol/sysex-framing.md
  - sources/korg-user-guide.md
updated: 2026-10-11
---

# Source — Korg minilogue xd MIDI Implementation rev 1.01

> Korg's official MIDI Implementation for the minilogue xd, **Revision 1.01 (2020-02-10)**. It is a
> plain-text document of about 2,200 lines with ASCII tables. It is the **[spec]** source for every
> protocol page in this wiki. It is external and copyrighted, so it is **not stored in the repo**.
> Pages summarize its facts in their own tables and cite it by section. Where it disagrees with
> real hardware or files, **[hw]** wins; see [errata](../protocol/errata.md).

## Where to get it

- Text: `https://cdn.korg.com/us/support/download/files/5227b0b2f7c1da5acd0911ac3e8c4f38.txt`
- Official download page: `https://www.korg.com/us/support/download/manual/0/811/4440/`
- The owner's manual edition E8 also has a MIDI implementation chart (pp. 68). It disagrees
  with the TXT in places (see [errata](../protocol/errata.md)). An earlier edition of the manual
  is in-repo; see [Korg user guide](korg-user-guide.md).

Cite as `Korg MIDI Impl rev 1.01 §<section>` / `TABLE n` / `NOTE n` / `*n` (footnote).

## Document structure → wiki pages

| Spec section | Content | Wiki page |
|--------------|---------|-----------|
| §1-1 Channel messages (transmitted), footnotes `*1-1`…`*1-5`, `*2-01`…`*2-24` | Notes, CCs the XD sends, representative values for stepped CCs (e.g. `0,64,127`) | [MIDI messages](../protocol/midi-messages.md), [CC map](../protocol/cc-map.md), [stepped params](../design/stepped-params.md) |
| `*1-4` / `*5-4` | 10-bit CC values: lower 3 bits first via CC#63 | [parameter encodings](../protocol/parameter-encodings.md) |
| `*3`, `*3-1`…`*3-6` | NRPN table; 10/8/14-bit sub-encodings; voice-mode and micro-tuning value tables | [NRPN map](../protocol/nrpn-map.md) |
| §1-2 / §2-3 System realtime | Clock, Start, Continue, Stop (and Active Sensing on receive); `*7` clock source condition | [MIDI messages](../protocol/midi-messages.md) |
| §1-3 / §2-4 Universal SysEx | Identity request/reply | [device identity](../protocol/device-identity.md) |
| §1-4 / §2-5 System exclusive | Korg exclusive header, function list | [SysEx framing](../protocol/sysex-framing.md) |
| §1-5 / §2-6 Search device | Korg search device request/reply | [device identity](../protocol/device-identity.md) |
| §2-1 Channel messages (recognized), `*5-n`, `*6-01`…`*6-19` | Receive ranges for stepped CCs (e.g. `0~42,43~85,86~127`) | [CC map](../protocol/cc-map.md), [stepped params](../design/stepped-params.md) |
| §2-2 System common | Song Position Pointer | [MIDI messages](../protocol/midi-messages.md) |
| §2-7 Bulk tuning dump, §2-8 Single note tuning change | MIDI Tuning Standard | [tuning](../protocol/tuning.md) |
| "MIDI EXCLUSIVE FORMAT" (1)–(24) | Every Korg exclusive message, byte by byte, with R/T direction | [SysEx framing](../protocol/sysex-framing.md) and the per-message pages |
| NOTE 1 | 7-bit data format conversion (7 data bytes → 8 MIDI bytes) | [7-bit codec](../protocol/korg-7bit-codec.md) |
| NOTE 2 | Status codes `0x23` (ACK) and `0x24`–`0x2F` (NAKs) | [SysEx framing](../protocol/sysex-framing.md) |
| NOTE 3 / NOTE 4 | User scale 384 bytes / user octave 36 bytes | [tuning](../protocol/tuning.md) |
| TABLE 1 | Global parameters | [global blob](../protocol/global-blob.md) |
| TABLE 2 (+ notes P1…P*n*, `[POLY]`/`[UNISON]`/`[CHORD]`/`[ARP]`, `[BPM SYNC OFF/ON]`) | Program parameters: synth and sequencer | [program blob](../protocol/program-blob.md), [sequencer blob](../protocol/sequencer-blob.md) |
| TABLE 3 / TABLE 4 | User scale / user octave data | [tuning](../protocol/tuning.md) |
| TABLE 5 / 6 / 7 | User module info / slot status / slot data | [user modules](../protocol/user-modules.md) |

## Reliability

The document is the best single reference, but the community and this crate found places where
it is wrong, ambiguous, or out of date. Known classes of problems:

- **Addresses and encodings that differ from shipped behavior.** Bank select and program-number
  encoding are the main ones, plus some NRPN addresses where the code and the spec disagree.
- **Size diagrams that disagree with tables.** Dump diagrams and TABLE 2 do not agree on the
  program size, and preset files use 448-byte blobs.
- **Edition conflicts.** The TXT and the owner's manual E8 chart disagree on some CC numbers.
- **Byte-order details the text leaves implicit**, such as 10-bit fields in blobs being
  little-endian with flag bits sharing the high byte.

Each item, with evidence and its resolution status, is on [errata](../protocol/errata.md).

## Sources

- Korg MIDI Impl rev 1.01 (TXT), header line: "KORG minilogue xd Implementation Revision 1.01
  (2020.2.10)".
- `docs/wayfinder/research/xd-midi-capabilities.md:6-12` — reference list and edition notes.
- `docs/wayfinder/research/xd-midi-capabilities.md:41-49` — documented discrepancies.
