---
title: Source — Korg minilogue xd owner's manual (in-repo Markdown)
category: source
sources:
  - docs/korg-user-guide/book.md
  - docs/korg-user-guide/metadata.json
  - docs/korg-user-guide/images/
related:
  - sources/korg-midi-implementation.md
  - protocol/usb-ports.md
  - protocol/global-blob.md
  - protocol/program-blob.md
  - glossary.md
updated: 2026-10-11
---

# Source — Korg minilogue xd owner's manual (in-repo Markdown)

> The Korg **minilogue xd / minilogue xd module owner's manual** converted to Markdown
> (`docs/korg-user-guide/book.md`, about 2,700 lines). Its images are kept in `images/`, and each
> one has an AI-written caption next to it, so an LLM can read the figures as text. It explains
> what parameters *mean* musically: ranges in user units, voice modes, the sequencer, and the
> global settings. The MIDI spec lists only raw values.

## What it is good for

| Need | Where to look in `book.md` | Backs wiki page |
|------|----------------------------|-----------------|
| User-facing ranges (e.g. PITCH −1200…+1200 cents, SHAPE 0…1023, TEMPO 56.0…240.0) | "Basic parameters" sections (VCO, MULTI ENGINE, FILTER, EG, LFO, EFFECTS) | [program blob](../protocol/program-blob.md), [stepped params](../design/stepped-params.md) |
| Voice modes (POLY/UNISON/CHORD/ARP) and VOICE MODE DEPTH meaning | "VOICE MODE section", "Voice mode list" | [program blob](../protocol/program-blob.md) |
| Sequencer, motion sequences, step editing | "Playing the sequencer" and sequencer chapters | [sequencer blob](../protocol/sequencer-blob.md) |
| Global settings (MIDI channel, clock source, routing, Tx/Rx filters) | GLOBAL EDIT chapters | [global blob](../protocol/global-blob.md) |
| USB-MIDI and MIDI connections, clock behavior | "Connecting … to a computer or other MIDI devices", MIDI chapter | [USB ports](../protocol/usb-ports.md) |
| Favorites | "Using Favorites" | [global blob](../protocol/global-blob.md) |
| Front-panel layout | Front-panel image captions | [overview](../overview.md) |

## Caveats

- `CLAUDE.md` calls it the guide "for the Minilogue". The content is the **minilogue xd**
  manual, covering both the keyboard and the module.
- It is a converted PDF. Page anchors (`#page-N-M`) are conversion artifacts, and some tables
  lost their formatting.
- User-unit ranges here are the display scale. Wire and blob values are raw integers; the
  mapping between the two is on each protocol page.
- The wayfinder research cites a newer manual edition, E8. Check edition-specific facts such as
  the MIDI chart CC numbers against [errata](../protocol/errata.md).

## Sources

- `docs/korg-user-guide/book.md:1-7` — title ("minilogue xd … Owner's Manual").
- `docs/korg-user-guide/book.md:605-760` — basic parameters (MASTER, VOICE MODE, VCO, MULTI ENGINE).
- `docs/korg-user-guide/metadata.json` — conversion table of contents with page polygons.
