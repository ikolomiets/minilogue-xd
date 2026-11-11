---
title: Stepped parameters (SteppedParam trait and the 29 enums)
category: design
sources:
  - crates/minilogue-xd/src/param/enums.rs
  - crates/minilogue-xd/src/param/mod.rs
  - crates/minilogue-xd/src/param/cc.rs
  - crates/minilogue-xd/src/param/nrpn.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/src/sysex/program/synth.rs
  - crates/minilogue-xd/src/builder/patch.rs
  - crates/minilogue-xd/src/midi_file.rs
  - crates/minilogue-xd/examples/quintal.rs
  - docs/korg-user-guide/book.md
related:
  - protocol/cc-map.md
  - protocol/nrpn-map.md
  - protocol/program-blob.md
  - protocol/parameter-encodings.md
  - protocol/errata.md
  - protocol/tuning.md
  - design/typed-values.md
  - design/realtime-controller.md
  - design/errors.md
  - zig/memory-budget.md
  - sources/korg-midi-implementation.md
  - glossary.md
updated: 2026-10-11
---

# Stepped parameters (SteppedParam trait and the 29 enums)

> A *stepped parameter* is a discrete control: a switch, a selector or a sub-type list. Each
> step has three representations:
>
> - the **TX value**: the exact CC data byte the synth sends for that step;
> - the **RX band**: the range of incoming CC bytes that select that step;
> - the **program value**: the dense index stored in SysEx program blobs and sent as NRPN data.
>
> The crate defines 29 such enums (238 variants) with one macro. The most important fact: in
> the 3-, 4- and 5-way enums the TX value is *not* the start of its band (Tri sends 64, but
> its band is 43–85). Decode incoming CCs by band, and never by exact match.

## Facts

### The three representations

| Representation | Type | Direction | Rule | Used by (Rust) |
|---|---|---|---|---|
| TX value | `u7` | encode, outgoing CC | One exact byte per variant (`*2-xx` notes) [spec] | `CcParam::to_messages` (`cc.rs:299-315`), `send_stepped_cc` (`controller.rs:130-132`), MIDI file export (`midi_file.rs:255`, `midi_file.rs:360`) |
| RX band | `u7..=u7` | decode, incoming CC | Inclusive range; the bands tile 0..127 for the 23 "tiled" enums (`*6-xx` notes) [spec] | `CcParamReceiver` (`cc.rs:428-450`) |
| Program value | `u8` | both, blob and NRPN | Dense 0-based index, equal to the declaration order for all 29 enums [rust] | Program blob (`synth.rs:402-512`, `synth.rs:533-642`); NRPN data byte CC6 (`nrpn.rs:278-356`, `nrpn.rs:511-621`) |

The 6 "point" enums (`CvInMode`, `MultiRouting`, `PortamentoMode`, `LfoTargetOsc`,
`ModAssignTarget`, `MicroTuning`) have no CC. Their TX value equals their program value, and
each RX band is the single value `prog..=prog`. So TX, RX and program values are all the same
number [rust].

### Decode functions and their errors

The macro generates four conversions plus `TryFrom<u8>` (`param/enums.rs:31-130`).

```text
to_tx_value(v)           = TX[v]                                  # total
to_program_value(v)      = PROG[v]                                # total
from_rx_value(b: u8)     = the v with RX_LO[v] <= b <= RX_HI[v]   # first match in declaration order
                           else OutOfRange{type_name, value=b, min=0, max=127}
from_program_value(b)    = the v with PROG[v] == b
                           else OutOfRange{type_name, value=b, min=0, max=max(PROG)}
try_from(b)  (TryFrom)   = the v with TX[v] == b                  # exact TX only
                           else OutOfRange{type_name, value=b, min=0, max=127}
```

- For a tiled enum, `from_rx_value` fails only for `b >= 128`. The input is a `u8`, not a `U7`.
- For a point enum, it fails for every `b` above the last value. The error still reports
  `min 0, max 127`, which is misleading; the valid range is `0..=n-1` (`param/enums.rs:63-73`).
- `TryFrom<u8>` matches exact TX values only (`param/enums.rs:94-109`). Library code never
  calls it; only tests do. Do not use it to decode wire data: `VcoWave::try_from(50)` fails,
  even though the synth treats 50 as Tri.
- The CC receiver turns a decode failure into "no event" (`.ok()`, `cc.rs:428-450`). For
  tiled enums that cannot happen, because a CC value is always ≤ 127.
- The NRPN receiver also turns a program-value failure into "no event"
  (`nrpn.rs:511-621`). The message is dropped silently.
- `from_program_value` errors surface only from blob parsing. There they fail the whole
  `SynthParams::from_bytes` (`synth.rs:402-512`).

### Band families (tiled enums)

| Family | Count | TX values (by program value) | RX bands | Spec |
|---|---|---|---|---|
| 2-way | 2 | 0, 127 | 0–63, 64–127 | `*2-08/09`, `*6-08/09` |
| 3-way | 9 | 0, 64, 127 | 0–42, 43–85, 86–127 | `*2-01…07,10,11,19`; `*6-01…07,10,11,18` |
| 4-way | 2 | 0, 42, 84, 127 | 0–31, 32–63, 64–95, 96–127 | `*2-02`, `*2-22`; `*6-02` |
| 5-way | 2 | `ModFxType`: 0, 38, 64, 84, 127; `VoiceModeType`: 0, 32, 64, 96, 127 | 0–25, 26–51, 52–76, 77–102, 103–127 | `*2-12`, `*6-12` (voice mode: none) |
| 8-way | 3 | 0, 16, 32, 48, 64, 80, 96, **127** | 16 wide: `16i … 16i+15` | `*2-18/20/21`, `*6-18` |
| 16-way | 3 | 0, 8, 16, …, 104, 112, **127** | 8 wide: `8i … 8i+7` | `*2-23/24`, `*6-18/19` |
| Delay | 1 | see table | see table (6 or 7 wide) | `*2-13`, `*6-13` |
| Reverb | 1 | see table | see table (7 or 8 wide) | `*2-14`, `*5-14` (sic) |

In every tiled enum, the first TX value is 0, the last is 127, and each TX value lies inside
its own band. Use explicit tables rather than formulas, because the 3-, 4-, 5-way, delay and
reverb tables are irregular.

### Catalogue: 2-way and 3-way enums

Columns are program values. The cell is the variant name; the Display string follows in
quotes only where it differs from the name. Line numbers are in `param/enums.rs`.

| Enum | Lines | 0: TX 0 | 1: TX 64 (2-way: 127) | 2: TX 127 |
|---|---|---|---|---|
| `Sync` (2-way) | 136-142 | Off | On | — |
| `Ring` (2-way) | 144-150 | Off | On | — |
| `VcoWave` | 156-163 | Sqr | Tri | Saw |
| `LfoWave` | 165-172 | Sqr | Tri | Saw |
| `LfoMode` | 174-181 | OneShot | Normal | Bpm |
| `LfoTarget` | 183-190 | Cutoff | Shape | Pitch |
| `EgTarget` | 192-199 | Cutoff | Pitch2 | Pitch |
| `MultiType` | 201-208 | Noise | Vpm | User |
| `CutoffKeytrack` | 210-217 | Off "0%" | Half "50%" | Full "100%" |
| `CutoffDrive` | 219-226 | Off "0%" | Half "50%" | Full "100%" |
| `ModFxSubTypeEnsemble` | 228-235 | Stereo | Light | Mono |

### Catalogue: 4-way and 5-way enums

| Enum | Lines | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|---|
| `VcoOctave` | 241-249 | Sixteen "16'" | Eight "8'" | Four "4'" | Two "2'" | — |
| `MultiSelectNoise` | 251-259 | High | Low | Peak | Decim | — |
| `ModFxType` | 265-274 | Chorus | Ensemble | Phaser | Flanger | User |
| `VoiceModeType` | 276-285 | Poly | Unison | Chord | Arp | ArpLatch |

TX values and bands for these are in [Band families](#band-families-tiled-enums).
`VoiceModeType`'s TX values and bands are invented by Rust: the spec has no voice-mode CC,
and no code path sends or receives them [rust]. See [Voice mode](#voicemodetype).

### Catalogue: 8-way and 16-way enums

| Prog | TX | RX band | `ModFxSubTypeChorus` (292-304) | `ModFxSubTypePhaser` (306-318) | `ModFxSubTypeFlanger` (320-332) |
|---|---|---|---|---|---|
| 0 | 0 | 0–15 | Stereo | Stereo | Stereo |
| 1 | 16 | 16–31 | Light | Fast | Light |
| 2 | 32 | 32–47 | Deep | Orange | Mono |
| 3 | 48 | 48–63 | Triphase | Small | HighSweep |
| 4 | 64 | 64–79 | Harmonic | SmallReso | MidSweep |
| 5 | 80 | 80–95 | Mono | Black | PanSweep |
| 6 | 96 | 96–111 | Feedback | Formant | MonoSweep |
| 7 | 127 | 112–127 | Vibrato | Twinkle | Triphase |

| Prog | TX | RX band | `MultiSelectVpm` (340-360) | `MultiSelectUser` (362-382), `ModFxSubTypeUser` (384-404) |
|---|---|---|---|---|
| 0 | 0 | 0–7 | Sin1 | User1 |
| 1 | 8 | 8–15 | Sin2 | User2 |
| 2 | 16 | 16–23 | Sin3 | User3 |
| 3 | 24 | 24–31 | Sin4 | User4 |
| 4 | 32 | 32–39 | Saw1 | User5 |
| 5 | 40 | 40–47 | Saw2 | User6 |
| 6 | 48 | 48–55 | Squ1 | User7 |
| 7 | 56 | 56–63 | Squ2 | User8 |
| 8 | 64 | 64–71 | Fat1 | User9 |
| 9 | 72 | 72–79 | Fat2 | User10 |
| 10 | 80 | 80–87 | Air1 | User11 |
| 11 | 88 | 88–95 | Air2 | User12 |
| 12 | 96 | 96–103 | Decay1 | User13 |
| 13 | 104 | 104–111 | Decay2 | User14 |
| 14 | 112 | 112–119 | Creep | User15 |
| 15 | 127 | 120–127 | Throat | User16 |

All five sub-type lists match the spec names and orders (`*2-18…24`, `*note P7`, `*note P8`,
`*note P13…P16`) [spec].

### Catalogue: `DelaySubType` (20) and `ReverbSubType` (18)

| Prog | `DelaySubType` (410-434) | TX | RX band | `ReverbSubType` (440-462) | TX | RX band |
|---|---|---|---|---|---|---|
| 0 | Stereo | 0 | 0–6 | Hall | 0 | 0–7 |
| 1 | Mono | 7 | 7–12 | Smooth | 8 | 8–14 |
| 2 | PingPong | 13 | 13–19 | Arena | 15 | 15–21 |
| 3 | Hipass | 20 | 20–25 | Plate | 22 | 22–28 |
| 4 | Tape | 26 | 26–31 | Room | 29 | 29–35 |
| 5 | OneTap | 32 | 32–38 | EarlyRef | 36 | 36–42 |
| 6 | StereoBpm | 39 | 39–44 | Space | 43 | 43–49 |
| 7 | MonoBpm | 45 | 45–51 | Riser | 50 | 50–56 |
| 8 | PingBpm | 52 | 52–57 | Submarine | 57 | 57–63 |
| 9 | HipassBpm | 58 | 58–63 | Horror | 64 | 64–71 |
| 10 | TapeBpm | 64 | 64–70 | User1 | 72 | 72–78 |
| 11 | Doubling | 71 | 71–76 | User2 | 79 | 79–85 |
| 12 | User1 | 77 | 77–83 | User3 | 86 | 86–92 |
| 13 | User2 | 84 | 84–89 | User4 | 93 | 93–99 |
| 14 | User3 | 90 | 90–95 | User5 | 100 | 100–106 |
| 15 | User4 | 96 | 96–102 | User6 | 107 | 107–113 |
| 16 | User5 | 103 | 103–108 | User7 | **117** | 114–120 |
| 17 | User6 | 109 | 109–115 | User8 | **127** | 121–127 |
| 18 | User7 | 116 | 116–121 | — | | |
| 19 | User8 | **127** | 122–127 | — | | |

The TX value equals the band start except where shown in bold. TX values come from
`*2-13`/`*2-14`; bands come from `*6-13`/`*5-14`. Both match Rust exactly [spec].

### Catalogue: point enums (no CC; TX = RX = program value)

| Enum | Lines | Values |
|---|---|---|
| `CvInMode` | 468-474 | 0 Modulation, 1 CvGatePlus, 2 CvGateMinus (`*note P20` [spec] agrees) |
| `MultiRouting` | 482-488 | 0 PreVcf, 1 PostVcf (blob byte 131 [spec] agrees) |
| `PortamentoMode` | 490-496 | 0 Auto, 1 On (blob byte 133 [spec] agrees) |
| `LfoTargetOsc` | 498-506 | 0 All, 1 Vco1Vco2, 2 Vco2, 3 Multi (`*note P22` [spec] agrees) |
| `ModAssignTarget` | 512-545 | 29 values; see below |
| `MicroTuning` | 551-594 | 39 values; see below |

### `ModAssignTarget` (29 variants)

The Rust program values are: 0 None, 1 GateTime, 2 PitchBend, 3 CutoffVelocity, 4 AmpVelocity,
5 MultiCtrl1, 6 MultiCtrl2, 7 MultiCtrl3, 8 MultiCtrl4, 9 Vco1Pitch, 10 Vco1Shape, 11 Vco2Pitch,
12 Vco2Shape, 13 CrossModDepth, 14 MultiShape, 15 MultiShiftShape, 16 Cutoff, 17 Resonance,
18 AmpLevel, 19 AmpPan, 20 EgAttack, 21 EgDecay, 22 EgSustain, 23 EgRelease, 24 LfoRate,
25 LfoInt, 26 DelayDryWet, 27 ReverbDryWet, 28 PortamentoTime (`param/enums.rs:515-543`).

The spec list (`*note P19`) is different at every index. It runs 0 GATE TIME,
1 PORTAMENTO, 2 V.M DEPTH, … 28 DELAY DEPTH [spec]. The user guide's joystick list agrees with
the spec (`docs/korg-user-guide/book.md:1384-1388`). The side-by-side table is canonical on
[program blob](../protocol/program-blob.md#enumerations-used-in-the-synth-half). The count
(29, values 0–28) agrees, so numeric round-trips work; only the labels are wrong [rust].

### MicroTuning (39 variants)

The Rust program values are 0–38 (`param/enums.rs:554-592`): 0 EqualTemp, 1 PureMajor,
2 PureMinor, 3 Pythagorean, 4 Werkmeister, 5 Kirnberger, 6 Slendro, 7 Pelog, 8 Ionian,
9 Dorian, 10 Aeolian, 11 MajorBlues, 12 MinorBlues, 13 Diminished, 14 CombiDiminished,
15 MajorPentatonic, 16 MinorPentatonic, 17 RagaChalanata, 18 RagaGamanashrama, 19 RagaTodi,
20 RagaShruti, 21–26 UserScale1–6, 27–32 UserOctave1–6, 33 Reverse1, 34 Reverse2,
35–38 AFX001–AFX004.

The spec numbers micro tuning differently for NRPN data and for the blob byte. The table maps
each spec entry to the Rust variant with the same meaning, if one exists:

| Spec name | NRPN value (`*3-6`) | Blob byte 122 (`*note P21`) | Rust variant (its value) | Same number? |
|---|---|---|---|---|
| Equal Temp … Aeolian | 0–10 | 0–10 | `EqualTemp` … `Aeolian` (0–10) | yes |
| Major Penta | 11 | 11 | `MajorPentatonic` (15) | no |
| Minor Penta | 12 | 12 | `MinorPentatonic` (16) | no |
| Reverse | 13 | 13 | `Reverse1` (33)? | no |
| AFX001–AFX004 | 14–17 | 14–17 | `AFX001`–`AFX004` (35–38) | no |
| AFX005, AFX006 | 18, 19 | 18, 19 | — | — |
| DC001–DC003 | 20–22 | 20–22 | — | — |
| USER SCALE 1–6 | 112–117 | 128–133 | `UserScale1`–`6` (21–26) | no |
| USER OCTAVE 1–6 | 118–123 | 134–139 | `UserOctave1`–`6` (27–32) | no |
| — | — | — | `MajorBlues`, `MinorBlues`, `Diminished`, `CombiDiminished`, `Raga*` ×4, `Reverse2` | not in the spec |

The user guide lists the same set as the spec: "Equal Temp...Reverse, AFX 001...006, DC
001...003, USER SCALE 1...6, USER OCTAVE 1...6" (`docs/korg-user-guide/book.md:1442`). So the
spec list is corroborated, and the Rust list has no source in the repository [spec]. The blob
byte is stored raw, so blob round-trips are lossless. Only `micro_tuning_enum()` and the NRPN
path use the wrong table (`synth.rs:690-693`, `nrpn.rs:564-566`). Tuning tables themselves
are on [tuning](../protocol/tuning.md).

### VoiceModeType

| Source | Order |
|---|---|
| Rust program value (`param/enums.rs:279-283`) | 0 Poly, 1 Unison, 2 Chord, 3 Arp, 4 ArpLatch |
| Spec NRPN data (`*3-5`) | 0 ARP LATCH, 1 ARP, 2 CHORD, 3 UNISON, 4 POLY [spec] |
| Spec blob byte 21 (`*note P3`, range 1~4) | "1 ARP, 2 CHORD, 2 UNISON, 3 POLY" (garbled; the obvious repair is 1 ARP, 2 CHORD, 3 UNISON, 4 POLY) [spec] |
| User guide spec sheet | "4 (ARP/LATCH, CHORD, UNISON, POLY)" (`docs/korg-user-guide/book.md:2667`) |
| Rust blob doc comment (`synth.rs:657-659`) | 1 Arp, 2 Chord, 3 Poly, 4 ArpLatch |
| Rust blob code and tests (`synth.rs:660-671`, `synth.rs:1191-1205`) | blob − 1 → program value, i.e. 1 Poly, 2 Unison, 3 Chord, 4 Arp |
| `examples/quintal.rs:650` | writes 4, commented "ArpLatch (as saved by hardware)" |

Every Korg source puts ARP(/LATCH) first and POLY last, so Rust's order is reversed. Only
CHORD (NRPN 2) lands on the same number. The quintal value 4 is a [hw] hint, and so is the
community capture listed on [program blob](../protocol/program-blob.md). Read as the repaired
P3, it means POLY; Rust decodes it as `Arp`, and the comment calls it `ArpLatch`. All three
readings disagree.

### Usage map: where each enum appears on the wire

"raw" means the crate stores or sends a plain `u7`/`u8` there and does not use the enum. NRPN
addresses are Rust's; the spec's differ (see [NRPN map](../protocol/nrpn-map.md)).

| Enum | CC (TX value out, RX band in) | NRPN (CC6 = program value) Rust / spec | Blob byte (program value) | Controller setter |
|---|---|---|---|---|
| `Sync` | CC80 | — | 34, **not via the enum**: inverted bool, 0 = ON | `set_sync` |
| `Ring` | CC81 | — | 35, inverted bool, 0 = ON | `set_ring` |
| `VcoWave` | CC50, CC51 | — | 22, 28 | `set_vco1_wave`, `set_vco2_wave` |
| `VcoOctave` | CC48, CC49 | `MultiOctave` `0x38` / `0x3A` | 23, 29, 130 | `set_vco1_octave`, `set_vco2_octave` |
| `MultiType` | CC53 | — | 38 | `set_multi_type` |
| `MultiSelectNoise` | CC103 raw | `0x0D` / `0x11` | 39 | — |
| `MultiSelectVpm` | CC103 raw | `0x0E` / `0x12` | 40 raw; `select_vpm_enum()` | — |
| `MultiSelectUser` | CC103 raw | `0x0F` / `0x13` | 41 raw; `select_user_enum()` | — |
| `CutoffDrive` | CC84 | — | 64 | `set_cutoff_drive` |
| `CutoffKeytrack` | CC83 | — | 65 | `set_cutoff_keytrack` |
| `EgTarget` | CC23 | — | 80 | `set_eg_target` |
| `LfoWave` | CC57 | — | 81 | `set_lfo_wave` |
| `LfoMode` | CC58 | — | 82 | `set_lfo_mode` |
| `LfoTarget` | CC56 | — | 87 | `set_lfo_target` |
| `ModFxType` | CC88 | — | 89 = **program value + 1** | `set_mod_fx_type` |
| `ModFxSubTypeChorus` / `Ensemble` / `Phaser` / `Flanger` / `User` | CC96 raw | — | 90 / 91 / 92 / 93 / 94 raw | — |
| `DelaySubType` | CC89 | — | 100 | `set_delay_sub_type` |
| `ReverbSubType` | CC90 | — | 106 | `set_reverb_sub_type` |
| `VoiceModeType` | — (TX/RX unused) | `0x0C` / `0x10` | 21 = program value + 1 (order disputed) | — |
| `ModAssignTarget` | — | `0x18`, `0x1A`, `0x1D`, `0x1F`, `0x51` / `0x22`, `0x24`, `0x29`, `0x2B`, none | 113, 115, 118, 120, 155 | — |
| `CvInMode` | — | `0x1C` / `0x28` | 117 | — |
| `MicroTuning` | — | `0x30` / `0x30` (values differ) | 122 raw; `micro_tuning_enum()` | `set_micro_tuning` (NRPN) |
| `LfoTargetOsc` | — | `0x35` / `0x36` | 127 | — |
| `MultiRouting` | — | `0x39` / `0x3B` | 131 | — |
| `PortamentoMode` | — | `0x3B` / `0x3D` | 133 | — |

Sources: CC encode `cc.rs:299-315`, CC decode `cc.rs:428-461`, NRPN `nrpn.rs:278-356` and
`nrpn.rs:511-621`, blob `synth.rs:402-512` and `synth.rs:533-642`, `ModFxType` + 1
`builder/patch.rs:167` and `synth.rs:676-688`, controller `controller.rs:316-393` and
`controller.rs:515-517`.

## Invariants

Each bullet is a comptime or runtime assert in a Zig port.

- Program values are dense: variant `i` has program value `i`, for all 29 enums.
- Variant counts: 2-way 2, 3-way 3, 4-way 4, 5-way 5, 8-way 8, 16-way 16, delay 20,
  reverb 18, `CvInMode` 3, `MultiRouting` 2, `PortamentoMode` 2, `LfoTargetOsc` 4,
  `ModAssignTarget` 29, `MicroTuning` 39. Total: 238.
- Tiled enums: `band[0].lo == 0`, `band[n-1].hi == 127`, and `band[i].hi + 1 == band[i+1].lo`.
- For every variant, `band.lo <= tx <= band.hi`, so `from_rx_value(to_tx_value(v)) == v`.
- TX values strictly increase with program value; `tx[0] == 0` and `tx[n-1] == 127` for
  tiled enums.
- Point enums: `tx == prog == band.lo == band.hi`, for every variant.
- `from_program_value(to_program_value(v)) == v`; `from_program_value(n)` fails for `n >= count`.
- A tiled decode of any `u7` never fails. Encode that in the type: return `E`, not `?E`.
- Blob adapters: `ModFxType` blob byte ∈ 1..=5 and `VoiceModeType` blob byte ∈ 1..=4; both
  equal program value + 1.

## Errata & gotchas

| # | Topic | Spec | Rust | Notes / provenance |
|---|---|---|---|---|
| 1 | SYNC/RING RX polarity | `*6-08/09` say "0~63,64~127 = ON, OFF". TX `*2-08/09` say "OFF, ON = 0,127". | Off 0–63, On 64–127 | The RX note contradicts the TX note, so a loopback would invert. Treat it as a typo [spec]. Not verified on [hw]. |
| 2 | SYNC/RING blob polarity | Bytes 34/35: "0,1 = SYNC ON, SYNC OFF" | Blob code bypasses the enum: `sync = (b == 0)` (`synth.rs:410-412`) | The enum's program values (Off 0, On 1) are **not** the blob encoding. Community tools disagree with the spec; see [program blob](../protocol/program-blob.md). |
| 3 | MOD FX TYPE RX | `*6-12`: `77~102,102~127` (102 overlaps) | User band 103–127 | Spec typo [spec]. |
| 4 | Reverb RX note label | Printed as `*5-14` | — | Should be `*6-14`; the values match Rust [spec]. |
| 5 | Noise RX bands | `*6-19` NOISE lists 8 bands 16 wide for 4 names | 4 bands 32 wide (`param/enums.rs:254-257`) | Rust follows the TX spacing `*2-22`. Moot in Rust, because CC103 is raw; it matters if Zig decodes CC103. See [CC map](../protocol/cc-map.md#context-dependent-ccs-96-103). |
| 6 | CC96 USER TX list | Ends "…104,112,128" | User16 TX 127 | 128 is not a 7-bit value; read it as 127 [spec]. |
| 7 | Voice mode order | `*3-5` ARP LATCH…POLY; P3 garbled | Reversed (Poly…ArpLatch) | See [VoiceModeType](#voicemodetype). Rust bug [rust]. |
| 8 | Voice mode TX/RX | No CC exists | TX 0/32/64/96/127 and bands invented | Unused; do not port [rust]. |
| 9 | Micro tuning list | 0–22, plus user entries (NRPN 112–123, blob 128–139) | 0–38 with different names from 11 up | See [MicroTuning](#microtuning-39-variants). The NRPN receiver drops spec values ≥ 39 [rust]. |
| 10 | Assign-target labels | P19 GATE TIME…DELAY DEPTH | None…PortamentoTime | Labels wrong at every index; numbers unaffected [rust]. |
| 11 | 1-based blob enums | Byte 21 is 1~4; byte 89 is 1~5 | Enum program values are 0-based; ±1 at the blob boundary | NRPN voice mode is 0-based (`*3-5`); the blob is 1-based [spec]. |
| 12 | Undocumented enum values | P24 documents user-param types 0~2 | `UserParamType` enum removed; fields stored raw | Factory patches contain 3 [hw] (`param/enums.rs:477-480`, commit `213020c`). Blob decode must not reject unknown values where round-trip fidelity matters. |
| 13 | Display strings | Panel labels (e.g. `1-SHOT`, `SMALL RESO`, `Werckmeister`) | Rust identifiers (`OneShot`, `SmallReso`, `Werkmeister`), except the overrides in `CutoffKeytrack`, `CutoffDrive` and `VcoOctave` | Display is cosmetic and has no protocol meaning [rust]. |
| 14 | Unused enums | — | `ModFxSubType*` ×5 are never used by CC96 or the blob; `MultiSelect*` RX bands are never used (CC103 is raw) | Public API only [rust]. |

## Test vectors

All from `crates/minilogue-xd/src/param/enums.rs` unless noted.

| Input | Expected | Citation |
|---|---|---|
| `Sync::from_rx_value(63)`, `(64)` | `Off`, `On` | `param/enums.rs:1239-1243` |
| `VcoWave::from_rx_value` 42, 43, 85, 86 | Sqr, Tri, Tri, Saw | `param/enums.rs:1245-1251` |
| `VcoOctave::from_rx_value` 31, 32, 63, 64, 95, 96 | Sixteen, Eight, Eight, Four, Four, Two | `param/enums.rs:1253-1261` |
| `DelaySubType::from_rx_value` 0, 6, 7, 127, 122 | Stereo, Stereo, Mono, User8, User8 | `param/enums.rs:1263-1283` |
| `ReverbSubType::from_rx_value` 0, 7, 8, 127, 121 | Hall, Hall, Smooth, User8, User8 | `param/enums.rs:1285-1307` |
| `ModFxSubTypeChorus::from_rx_value` 15, 16, 112, 127 | Stereo, Light, Vibrato, Vibrato | `param/enums.rs:1309-1326` |
| Every tiled enum: `from_rx_value(0..=127)` | all succeed | macro `test_stepped_enum!`, `param/enums.rs:647-689` |
| Every tiled enum: `try_from(200)`, `from_program_value(255)` | both error | `param/enums.rs:677-687` |
| `CvInMode::from_rx_value` 3, 127 | error | `param/enums.rs:997-1000` |
| `MultiRouting` / `PortamentoMode::from_rx_value(2)`; `LfoTargetOsc::from_rx_value(4)` | error | `param/enums.rs:1022-1024`, `1041-1043`, `1070-1072` |
| `ModAssignTarget::from_rx_value` 29, 127 | error; count is 29 | `param/enums.rs:1123-1131` |
| `MicroTuning::from_rx_value` 39, 127 | error; count is 39 | `param/enums.rs:1201-1209` |
| Display `CutoffKeytrack`/`CutoffDrive` Off/Half/Full | `"0%"`, `"50%"`, `"100%"` | `param/enums.rs:1215-1227` |
| Display `VcoOctave` | `"16'"`, `"8'"`, `"4'"`, `"2'"` | `param/enums.rs:1229-1236` |
| `set_vco1_wave(Saw)` | `B0 32 7F` | `controller.rs:843-848` |
| `set_vco1_octave(Four)` | `B0 30 54` | `controller.rs:867-872` |
| `set_mod_fx_type(Flanger)` | `B0 58 54` | `controller.rs:955-960` |
| `set_reverb_sub_type(Hall)` | `B0 5A 00` | `controller.rs:971-976` |
| Blob byte 21 = 1, 2, 3, 4 | Poly, Unison, Chord, Arp (Rust; disputed) | `sysex/program/synth.rs:1191-1205` |
| NRPN `B0 63 00`, `B0 62 0C`, `B0 06 00` | `VoiceModeType::Poly` (Rust address and order) | `param/nrpn.rs:1298-1309` |

## Rust design

- **Trait.** `SteppedParam: Sized + Copy` has four methods: `to_tx_value`, `from_rx_value`,
  `to_program_value` and `from_program_value` (`param/mod.rs:15-43`). It exists so that
  `send_stepped_cc<P: SteppedParam>` (`controller.rs:130-132`) and the generic test helpers
  (`param/enums.rs:603-641`) can work over every enum.
- **Macro.** `stepped_param_enum!` (`param/enums.rs:16-130`) takes one row per variant:
  `Variant => { tx, rx: lo..=hi, prog, display? }`. It emits the enum (`#[non_exhaustive]`,
  `Copy`, `Hash`), the trait impl as `match` arms, `TryFrom<u8>` and `Display`. RX bounds must
  be literals because they become match patterns. TX and program values are expressions, so
  the program decode is a guard chain, `x if x == $prog`. `@max_prog` computes the error's
  `max` field.
- **Essential.** Keep the three-representation model, the per-enum tables, total band decode
  for tiled enums, and exact decode for program values.
- **Incidental.** Drop the trait-based generic dispatch, the macro, `#[non_exhaustive]`,
  `TryFrom<u8>` (unused and misleading), Display via `fmt`, `Result` on decodes that cannot
  fail, and `i64` error payloads.
- **Gaps.**
  - There is no type that ties the CC96/CC103 value to the current MOD FX / MULTI type.
  - There is no blob adapter type: the +1 offsets, the inverted sync/ring bytes and the sparse
    micro-tuning values are handled ad hoc in `synth.rs` and `builder/patch.rs`.

## Zig / TigerStyle notes

- **Represent each enum as `enum(u8)` whose tag is the program value.** Then
  `@intFromEnum`/`@enumFromInt` *are* the program-value codec. Bounds-check before
  `@enumFromInt` and return `error.ValueInvalid` for blob/NRPN input.
- **Generate tables at comptime from one row array per enum.** Each row holds `tx: u7`,
  `rx_lo: u7`, `rx_hi: u7` and a panel `label` taken from the spec, not the Rust name. Use
  comptime asserts for every [invariant](#invariants). One generic function replaces the
  macro.

  ```zig
  // sketch
  pub const Step = struct { tx: u7, rx_lo: u7, rx_hi: u7, label: []const u8 };

  pub fn StepTable(comptime E: type, comptime steps: []const Step, comptime tiled: bool) type {
      comptime {
          assert(steps.len > 0 and steps.len <= 39);
          for (steps, 0..) |step, i| {
              assert(step.rx_lo <= step.tx and step.tx <= step.rx_hi);
              if (tiled and i > 0) assert(@as(u8, steps[i - 1].rx_hi) + 1 == step.rx_lo);
          }
          if (tiled) assert(steps[0].rx_lo == 0 and steps[steps.len - 1].rx_hi == 127);
      }
      return struct {
          pub fn tx_value(value: E) u7 {
              return steps[@intFromEnum(value)].tx;
          }
          /// Tiled tables only: total over u7.
          pub fn from_rx_value(data: u7) E {
              comptime assert(tiled);
              for (steps, 0..) |step, i| { // bounded: steps.len <= 20 for tiled tables
                  if (data <= step.rx_hi) return @enumFromInt(i);
              }
              unreachable;
          }
          pub fn from_program_value(raw: u8) error{ValueInvalid}!E {
              if (raw >= steps.len) return error.ValueInvalid;
              return @enumFromInt(raw);
          }
      };
  }
  ```

- **Decode speed vs. size.** A linear scan of at most 20 bands is bounded and needs no extra
  memory. If you want O(1), build a comptime `[128]u8` lookup table per tiled enum:
  23 × 128 = 2944 bytes of read-only data.
- **Separate wire contexts explicitly.** Name the functions `cc_tx_value`, `cc_from_rx_value`,
  `program_value`, `nrpn_value` and `blob_value`. Give the blob adapters their own small
  functions with paired asserts:
  - `mod_fx_type`: blob = program + 1;
  - voice mode: blob = NRPN value, valid for 1..=4 (assuming the P3 repair);
  - sync/ring: inverted bool;
  - micro tuning: a sparse map with NRPN user entries + 16 = blob.
- **Use the spec numbering for `VoiceModeType`, `ModAssignTarget` and `MicroTuning`, and pin
  it with tests.** Make `MicroTuning` an `enum(u8)` with explicit sparse tags: 0–22, plus
  112–123 for NRPN. Add a blob-mapping function that asserts `blob ∈ {0..22, 128..139}`.
- **Do not port the invented `VoiceModeType` TX/RX bands.** Do not port `TryFrom`.
- **Make CC96/CC103 decoding context-dependent.** Take the current `ModFxType`/`MultiType` as
  an explicit argument; see [CC map](../protocol/cc-map.md#context-dependent-ccs-96-103).
- **Keep labels for UI only.** Labels are `[]const u8` constants, and no formatting happens in
  the core library.
- **Sizes for the [memory budget](../zig/memory-budget.md):**
  - Largest enum: 39 values (`MicroTuning`). Largest tiled enum: 20 (`DelaySubType`).
  - Each stepped CC write is exactly 1 ControlChange (3 bytes).
  - Each enum NRPN write is 3 ControlChanges (CC99, CC98, CC6; 9 bytes).

## Open questions

- **Voice mode on [hw].** Capture NRPN and blob values for each voice mode. In particular,
  where does ARP LATCH live in the blob, when byte 21 is only 1~4?
- **Where did Rust's micro-tuning list come from?** The Blues, Diminished and Raga names
  appear in neither the spec nor the user guide. Check whether a later firmware changed the
  list.
- **Aftertouch assign list.** The user guide titles it "MIDI Aftertouch [BALANCE...GATE TIME]"
  (`docs/korg-user-guide/book.md:1526`). That hints that byte 155 may use a different list
  from P19. Also, `0x51` as its NRPN address is Rust-only.
- **Hardware checks for the RX notes.**
  - Does CC80 = 127 turn SYNC on? This resolves erratum 1.
  - Does CC103 with NOISE selected use 4 bands of 32 or 8 bands of 16?
- **Does the XD transmit CC96/CC103 sub-type changes** when the MOD FX/MULTI type changes?
  This affects whether a receiver can track context from the CC stream alone.

## Sources

- `crates/minilogue-xd/src/param/mod.rs:15-43` — `SteppedParam` trait.
- `crates/minilogue-xd/src/param/enums.rs:16-130` — macro; `:136-594` all 29 enums;
  `:477-480` UserParamType removal; `:603-641` test helpers; `:643-1327` tests.
- `crates/minilogue-xd/src/param/cc.rs:299-315` (encode), `:428-461` (decode).
- `crates/minilogue-xd/src/param/nrpn.rs:278-356` (encode), `:511-621` (decode),
  `:1298-1309` (voice-mode test).
- `crates/minilogue-xd/src/controller.rs:130-132`, `:316-393`, `:515-517`, `:843-976`.
- `crates/minilogue-xd/src/sysex/program/synth.rs:402-512`, `:533-642`, `:656-703`,
  `:1191-1205`.
- `crates/minilogue-xd/src/builder/patch.rs:167` — `ModFxType` + 1.
- `crates/minilogue-xd/examples/quintal.rs:650` — voice mode 4 "as saved by hardware".
- `docs/korg-user-guide/book.md:1384-1388` (assign list), `:1442` (micro tuning),
  `:1526` (aftertouch), `:2667` (voice modes).
- Korg MIDI Impl rev 1.01 §1-1 TX notes `*2-01…*2-24` and NRPN notes `*3-5`, `*3-6`; §2-1 RX
  notes `*6-01…*6-19` (incl. `*5-14`); TABLE 2 `*note P3, P4, P6–P9, P12–P22`. See
  [Korg MIDI implementation](../sources/korg-midi-implementation.md).
