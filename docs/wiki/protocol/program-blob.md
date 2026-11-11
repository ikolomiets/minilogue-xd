---
title: Program data blob — synth half (TABLE 2, offsets 0–155)
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/sysex/program/synth.rs
  - crates/minilogue-xd/src/sysex/helpers.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/src/param/enums.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/examples/quintal.rs
  - docs/wayfinder/research/xd-midi-capabilities.md
  - README.md
  - https://cdn.korg.com/us/support/download/files/5227b0b2f7c1da5acd0911ac3e8c4f38.txt
related:
  - protocol/sequencer-blob.md
  - protocol/sysex-framing.md
  - protocol/korg-7bit-codec.md
  - protocol/file-formats.md
  - protocol/global-blob.md
  - protocol/parameter-encodings.md
  - protocol/nrpn-map.md
  - protocol/tuning.md
  - protocol/errata.md
  - design/byte-fidelity.md
  - design/builders.md
  - design/stepped-params.md
  - design/sysex-transactions.md
  - design/errors.md
  - zig/memory-budget.md
updated: 2026-11-11
---

# Program data blob — synth half (TABLE 2, offsets 0–155)

> A minilogue xd program is a **1024-byte blob** (Korg MIDI Impl rev 1.01 TABLE 2). Bytes
> 0–155 hold the sound ("synth half", magic `PROG`). Bytes 156–1023 hold the step sequencer
> ([sequencer blob](sequencer-blob.md), magic `PREDSQ`). The same blob travels in four SysEx
> messages (functions `0x10`/`0x40` for the edit buffer, `0x1C`/`0x4C` for one of 500 stored
> slots) and in `.prog_bin` file entries ([file formats](file-formats.md)). The most important
> fact: the 34 continuous parameters are **10-bit little-endian** pairs. The low byte comes
> first. Bits 9..8 sit in bits 1..0 of the second byte. The upper 6 bits of that second byte
> hold device data that a writer must preserve. The spec labels these pairs `H`/`L`, in the
> wrong order **[hw]**.

## Facts

### Blob map

| Offsets | Size | Content | Page |
|---------|------|---------|------|
| 0–3 | 4 | ASCII `PROG` (`50 52 4F 47`) | this page |
| 4–155 | 152 | Synth parameters | this page |
| 156–1023 | 868 | `PREDSQ` + sequencer, motion, ARP | [sequencer blob](sequencer-blob.md) |

`156 + 868 = 1024`. The Rust crate splits the blob at offset 156. `SynthParams::SIZE = 156`,
`SequencerParams::SIZE = 868` and `ProgramData::SIZE = 1024` (`synth.rs:361`, `sequencer.rs:150`,
`mod.rs:107`).

### Field encodings

| Kind | Layout | Read | Write |
|------|--------|------|-------|
| `u8` | one byte | `b[o]` | `b[o] = v` |
| bool | one byte, `0,1 = Off,On` | Rust: `b[o] != 0` | Rust: `0` or `1` |
| inverted bool | bytes 34, 35 only: `0 = ON`, `1 = OFF` | Rust: `b[o] == 0` | Rust: `!on` |
| **10-bit LE** | `b[o]` = bits 7..0; `b[o+1]` bits 1..0 = bits 9..8; `b[o+1]` bits 7..2 = *not ours* | `b[o] \| (b[o+1] & 0x03) << 8` | `b[o] = v & 0xFF; b[o+1] = (b[o+1] & 0xFC) \| (v >> 8)` |
| 2-bit packed | bytes 148, 149 (see table) | `(b >> s) & 0x03` | read-modify-write |

The 10-bit helpers are `read_10bit` / `write_10bit` (`helpers.rs:16-35`). The writer masks
its input to 10 bits and only touches bits 1..0 of the high byte. See
[byte fidelity](../design/byte-fidelity.md) for why the upper 6 bits matter.

**Byte order evidence [hw]:** Commit `1c16566` changed the codec from big-endian to
little-endian after big-endian writes "corrupt[ed] every continuous parameter" on a real
device. A hardware-saved reverb time of 870 reads `66 03` at bytes 107–108; the comment
"byte 107 = 0x66" is at `examples/quintal.rs:716`. A Replicant xd cutoff of 665 reads `99 02`
(`helpers.rs:156-161`). All four community parsers read these pairs as `<H` (little-endian)
**[community]**. The spec's `H:0~7` / `L:0~7` Bit-column labels are therefore wrong.
Ironically, the spec's own BPM pair at 164–165 is labelled `L` then `H`
([sequencer blob](sequencer-blob.md)).

### Synth field table (offsets 0–155)

Spec column = Korg MIDI Impl rev 1.01 TABLE 2. In the Rust column, "strict" means
`from_program_value` rejects unknown values. One out-of-range byte then makes the **whole blob**
fail to parse with `OutOfRange` (`synth.rs:372-514`). "raw" means the byte is stored
unchanged.

| Off | Width | Spec name | Spec range → meaning | Rust field : decoding |
|-----|-------|-----------|----------------------|-----------------------|
| 0–3 | 4 B | `'PROG'` | ASCII magic | checked → `InvalidMagic` |
| 4–15 | 12 B | PROGRAM NAME | P1 charset, see [Program name](#program-name) | `name: ProgramName` (validated) |
| 16 | u8 | OCTAVE | 0~4 = −2~+2 | `octave` raw |
| 17 | u8 | PORTAMENTO | 0~127 (Range column says "0,1": typo) | `portamento` raw |
| 18 | bool | KEY TRIG | 0,1 = Off,On | `key_trig` |
| 19–20 | 10-bit | VOICE MODE DEPTH | 0~1023, meaning per mode (P2) | `voice_mode_depth` |
| 21 | u8 | VOICE MODE TYPE | 1~4 (P3 garbled, see errata) | `voice_mode_type` raw |
| 22 | u8 | VCO 1 WAVE | 0~2 = SQR, TRI, SAW (P4) | `VcoWave` strict |
| 23 | u8 | VCO 1 OCTAVE | 0~3 = 16', 8', 4', 2' | `VcoOctave` strict |
| 24–25 | 10-bit | VCO 1 PITCH | 0~1023, cents curve P5 | `vco1_pitch` |
| 26–27 | 10-bit | VCO 1 SHAPE | 0~1023 | `vco1_shape` |
| 28 | u8 | VCO 2 WAVE | as 22 | `VcoWave` strict |
| 29 | u8 | VCO 2 OCTAVE | as 23 | `VcoOctave` strict |
| 30–31 | 10-bit | VCO 2 PITCH | as 24 | `vco2_pitch` |
| 32–33 | 10-bit | VCO 2 SHAPE | 0~1023 | `vco2_shape` |
| 34 | u8 | SYNC | **0,1 = SYNC ON, SYNC OFF** | `sync = (b == 0)` |
| 35 | u8 | RING | **0,1 = RING ON, RING OFF** | `ring = (b == 0)` |
| 36–37 | 10-bit | CROSS MOD DEPTH | 0~1023 | `cross_mod_depth` |
| 38 | u8 | MULTI TYPE | 0~2 = NOISE, VPM, USER | `MultiType` strict |
| 39 | u8 | SELECT NOISE | 0~3 = HIGH, LOW, PEAK, DECIM (P6) | `MultiSelectNoise` strict |
| 40 | u8 | SELECT VPM | 0~15 (P7) | `select_vpm` raw |
| 41 | u8 | SELECT USER | 0~15 = USER1~16 (P8) | `select_user` raw |
| 42–43 | 10-bit | SHAPE NOISE | 0~1023 | `shape_noise` |
| 44–45 | 10-bit | SHAPE VPM | 0~1023 | `shape_vpm` |
| 46–47 | 10-bit | SHAPE USER | 0~1023 | `shape_user` |
| 48–49 | 10-bit | SHIFT SHAPE NOISE | 0~1023 | `shift_shape_noise` |
| 50–51 | 10-bit | SHIFT SHAPE VPM | 0~1023 | `shift_shape_vpm` |
| 52–53 | 10-bit | SHIFT SHAPE USER | 0~1023 | `shift_shape_user` |
| 54–55 | 10-bit | VCO1 LEVEL | 0~1023 | `vco1_level` |
| 56–57 | 10-bit | VCO2 LEVEL | 0~1023 | `vco2_level` |
| 58–59 | 10-bit | MULTI LEVEL | 0~1023 | `multi_level` |
| 60–61 | 10-bit | CUTOFF | 0~1023 | `cutoff` |
| 62–63 | 10-bit | RESONANCE | 0~1023 | `resonance` |
| 64 | u8 | CUTOFF DRIVE | 0~2 = 0%, 50%, 100% (P9) | `CutoffDrive` strict |
| 65 | u8 | CUTOFF KEYBOARD TRACK | 0~2 = 0%, 50%, 100% (P9) | `CutoffKeytrack` strict |
| 66–67 | 10-bit | AMP EG ATTACK | 0~1023 | `amp_eg_attack` |
| 68–69 | 10-bit | AMP EG DECAY | 0~1023 | `amp_eg_decay` |
| 70–71 | 10-bit | AMP EG SUSTAIN | 0~1023 | `amp_eg_sustain` |
| 72–73 | 10-bit | AMP EG RELEASE | 0~1023 | `amp_eg_release` |
| 74–75 | 10-bit | EG ATTACK | 0~1023 | `eg_attack` |
| 76–77 | 10-bit | EG DECAY | 0~1023 | `eg_decay` |
| 78–79 | 10-bit | EG INT | 0~1023, signed % curve P10 | `eg_int` |
| 80 | u8 | EG TARGET | 0~2 = CUTOFF, PITCH2, PITCH (Bit column "H:0~7": typo) | `EgTarget` strict |
| 81 | u8 | LFO WAVE | 0~2 = SQR, TRI, SAW (P4) | `LfoWave` strict |
| 82 | u8 | LFO MODE | 0~2 = 1-SHOT, NORMAL, BPM | `LfoMode` strict |
| 83–84 | 10-bit | LFO RATE | 0~1023; BPM-synced: 16 bands (P11) | `lfo_rate` |
| 85–86 | 10-bit | LFO INT | 0~1023 | `lfo_int` |
| 87 | u8 | LFO TARGET | 0~2 = CUTOFF, SHAPE, PITCH | `LfoTarget` strict |
| 88 | bool | MOD FX ON/OFF | 0,1 = Off,On | `mod_fx_on` |
| 89 | u8 | MOD FX TYPE | **1~5** = CHORUS, ENSEMBLE, PHASER, FLANGER, USER (P12) | `mod_fx_type` raw, 1-based |
| 90 | u8 | MOD FX CHORUS | 0~7 (P13) | `mod_fx_chorus` raw |
| 91 | u8 | MOD FX ENSEMBLE | 0~2 (P14) | `mod_fx_ensemble` raw |
| 92 | u8 | MOD FX PHASER | 0~7 (P15) | `mod_fx_phaser` raw |
| 93 | u8 | MOD FX FLANGER | 0~7 (P16) | `mod_fx_flanger` raw |
| 94 | u8 | MOD FX USER | 0~15 = USER1~16 (P8) | `mod_fx_user` raw |
| 95–96 | 10-bit | MOD FX TIME | 0~1023 | `mod_fx_time` |
| 97–98 | 10-bit | MOD FX DEPTH | 0~1023 | `mod_fx_depth` |
| 99 | bool | DELAY ON/OFF | 0,1 = Off,On | `delay_on` |
| 100 | u8 | DELAY SUB TYPE | 0~19 (P17) | `DelaySubType` strict |
| 101–102 | 10-bit | DELAY TIME | 0~1023 | `delay_time` |
| 103–104 | 10-bit | DELAY DEPTH | 0~1023 | `delay_depth` |
| 105 | bool | REVERB ON/OFF | 0,1 = Off,On | `reverb_on` |
| 106 | u8 | REVERB SUB TYPE | Range column says 0~19; P18 lists 0~17 | `ReverbSubType` strict (0..17) |
| 107–108 | 10-bit | REVERB TIME | 0~1023 | `reverb_time` |
| 109–110 | 10-bit | REVERB DEPTH | 0~1023 | `reverb_depth` |
| 111 | u8 | BEND RANGE (+) | 0~12 = OFF~+12 notes | `bend_range_plus` raw |
| 112 | u8 | BEND RANGE (−) | 0~12 = OFF~−12 notes | `bend_range_minus` raw |
| 113 | u8 | JOYSTICK ASSIGN (+) | 0~28 (P19) | `ModAssignTarget` strict, **mislabelled** |
| 114 | u8 | JOYSTICK RANGE (+) | 0~200 = −100%~+100% | `joystick_range_plus` raw |
| 115 | u8 | JOYSTICK ASSIGN (−) | 0~28 (P19) | `ModAssignTarget` strict |
| 116 | u8 | JOYSTICK RANGE (−) | 0~200 = −100%~+100% | `joystick_range_minus` raw |
| 117 | u8 | CV IN MODE | 0~2 = Modulation, CV/Gate(+), CV/Gate(−) (P20) | `CvInMode` strict |
| 118 | u8 | CV IN 1 ASSIGN | 0~28 (P19) | `ModAssignTarget` strict |
| 119 | u8 | CV IN 1 RANGE | 0~200 = −100%~+100% | `cv_in1_range` raw |
| 120 | u8 | CV IN 2 ASSIGN | 0~28 (P19) | `ModAssignTarget` strict |
| 121 | u8 | CV IN 2 RANGE | 0~200 = −100%~+100% | `cv_in2_range` raw |
| 122 | u8 | MICRO TUNING | 0~139, sparse (P21) | `micro_tuning` raw |
| 123 | u8 | SCALE KEY | 0~24 = −12~+12 notes | `scale_key` raw |
| 124 | u8 | PROGRAM TUNING | 0~100 = −50~+50 cents | `program_tuning` raw |
| 125 | bool | LFO KEY SYNC | 0,1 = Off,On | `lfo_key_sync` |
| 126 | bool | LFO VOICE SYNC | 0,1 = Off,On | `lfo_voice_sync` |
| 127 | u8 | LFO TARGET OSC | 0~3 = ALL, VCO1+VCO2, VCO2, MULTI (P22) | `LfoTargetOsc` strict |
| 128 | u8 | CUTOFF VELOCITY | 0~127 | `cutoff_velocity` raw |
| 129 | u8 | AMP VELOCITY | 0~127 | `amp_velocity` raw |
| 130 | u8 | MULTI OCTAVE | 0~3 = 16', 8', 4', 2' | `VcoOctave` strict |
| 131 | u8 | MULTI ROUTING | 0,1 = Pre VCF, Post VCF | `MultiRouting` strict |
| 132 | bool | EG LEGATO | 0,1 = Off,On | `eg_legato` |
| 133 | u8 | PORTAMENTO MODE | 0,1 = Auto, On | `PortamentoMode` strict |
| 134 | bool | PORTAMENTO BPM SYNC | 0,1 = Off,On | `portamento_bpm_sync` |
| 135 | u8 | PROGRAM LEVEL | 12~132 = −18 dB~+6 dB | `program_level` raw |
| 136 | u8 | VPM PARAM1 (Feedback) | 0~200 = −100%~+100% | `vpm_param1` raw |
| 137 | u8 | VPM PARAM2 (Noise Depth) | 0~200 | `vpm_param2` raw |
| 138 | u8 | VPM PARAM3 (ShapeModInt) | 0~200 | `vpm_param3` raw |
| 139 | u8 | VPM PARAM4 (Mod Attack) | 0~200 | `vpm_param4` raw |
| 140 | u8 | VPM PARAM5 (Mod Decay) | 0~200 | `vpm_param5` raw |
| 141 | u8 | VPM PARAM6 (ModKeyTrack) | 0~200 | `vpm_param6` raw |
| 142–147 | 6 × u8 | USER PARAM1~6 | range depends on its type (P23, P24) | `user_param1..6` raw |
| 148 | bits 1..0 | USER PARAM5 TYPE | P24 | `user_param5_type = b & 0x03` |
| 148 | bits 3..2 | USER PARAM6 TYPE | P24 | `user_param6_type = (b >> 2) & 0x03` |
| 148 | bits 7..4 | Reserved | — | `user_param_type_flags = b & 0xF0` (kept, **[hw]** non-zero) |
| 149 | bits 1..0 | USER PARAM1 TYPE | P24 | `user_param1_type` |
| 149 | bits 3..2 | USER PARAM2 TYPE | P24 | `user_param2_type` |
| 149 | bits 5..4 | USER PARAM3 TYPE | P24 | `user_param3_type` |
| 149 | bits 7..6 | USER PARAM4 TYPE | P24 | `user_param4_type` |
| 150 | u8 | PROGRAM TRANSPOSE | 1~25 = −12~+12 notes | `program_transpose` raw |
| 151–152 | 10-bit | DELAY DRY WET | "0~1024" (sic; 10 bits hold 0~1023) | `delay_dry_wet` |
| 153–154 | 10-bit | REVERB DRY WET | "0~1024" (sic) | `reverb_dry_wet` |
| 155 | u8 | MIDI AFTER TOUCH ASSIGN | 0~28 (P19) | `ModAssignTarget` strict |

The 34 10-bit fields start at offsets 19, 24, 26, 30, 32, 36, 42, 44, 46, 48, 50, 52, 54, 56,
58, 60, 62, 66, 68, 70, 72, 74, 76, 78, 83, 85, 95, 97, 101, 103, 107, 109, 151 and 153. Every
byte in 0–155 belongs to exactly one field. The synth half has **no unused byte**. The only
unmodelled bits are the upper 6 bits of each 10-bit high byte (34 × 6 bits) and the reserved
nibble of byte 148.

### Enumerations used in the synth half

The Rust enum types and their CC/NRPN encodings are documented in
[stepped parameters](../design/stepped-params.md). In the blob, every enum is a dense
zero-based "program value". The exceptions are VOICE MODE TYPE (1-based), MOD FX TYPE
(1-based) and MICRO TUNING (sparse).

| Spec note | Values [spec] | Rust agrees? |
|-----------|---------------|--------------|
| P4 wave | 0 SQR, 1 TRI, 2 SAW | yes (`VcoWave`, `LfoWave`) |
| P6 noise | 0 HIGH, 1 LOW, 2 PEAK, 3 DECIM | yes |
| P7 VPM | 0 SIN1, 1 SIN2, 2 SIN3, 3 SIN4, 4 SAW1, 5 SAW2, 6 SQU1, 7 SQU2, 8 FAT1, 9 FAT2, 10 AIR1, 11 AIR2, 12 DECAY1, 13 DECAY2, 14 CREEP, 15 THROAT | raw byte; enum via `select_vpm_enum` |
| P8 user slot | 0~15 = USER1~USER16 | raw byte |
| P9 drive/keytrack | 0 = 0%, 1 = 50%, 2 = 100% | yes (`Off`, `Half`, `Full`) |
| P12 mod FX type | **1** CHORUS, 2 ENSEMBLE, 3 PHASER, 4 FLANGER, 5 USER | yes: `ModFxType` program value + 1 |
| P13 chorus | 0 STEREO, 1 LIGHT, 2 DEEP, 3 TRIPHASE, 4 HARMONIC, 5 MONO, 6 FEEDBACK, 7 VIBRATO | raw |
| P14 ensemble | 0 STEREO, 1 LIGHT, 2 MONO | raw |
| P15 phaser | 0 STEREO, 1 FAST, 2 ORANGE, 3 SMALL, 4 SMALL RESO, 5 BLACK, 6 FORMANT, 7 TWINKLE | raw |
| P16 flanger | 0 STEREO, 1 LIGHT, 2 MONO, 3 HIGH SWEEP, 4 MID SWEEP, 5 PAN SWEEP, 6 MONO SWEEP, 7 TRIPHASE | raw |
| P17 delay | 0 STEREO, 1 MONO, 2 PING PONG, 3 HIPASS, 4 TAPE, 5 ONE TAP, 6 STEREO BPM, 7 MONO BPM, 8 PING BPM, 9 HIPASS BPM, 10 TAPE BPM, 11 DOUBLING, 12~19 USER1~USER8 | yes |
| P18 reverb | 0 HALL, 1 SMOOTH, 2 ARENA, 3 PLATE, 4 ROOM, 5 EARLY REF, 6 SPACE, 7 RISER, 8 SUBMARINE, 9 HORROR, 10~17 USER1~USER8 | yes |
| P20 CV in mode | 0 Modulation, 1 CV/Gate(+), 2 CV/Gate(−) | yes |
| P22 LFO target osc | 0 ALL, 1 VCO1+VCO2, 2 VCO2, 3 MULTI | yes |
| P24 user param type | 0 Percent (value 0~101 = 0~100%), 1 Bipolar (0~200 = −100~+100), 2 Select (0~100 = 1~101); 3 undocumented | raw 2-bit fields |
| **P3 voice mode** | see table below | **no** |
| **P19 mod assign** | see table below | **no** (labels) |
| **P21 micro tuning** | see table below | **no** |

#### P3 — VOICE MODE TYPE (byte 21)

The spec text reads "1: ARP, 2: CHORD, 2: UNISON, 3: POLY" with range 1~4. The number 2
appears twice and 4 is missing. The obvious repair is the community consensus.

| Byte | Spec (repaired) / community | Rust `voice_mode_type_enum()` |
|------|------------------------------|-------------------------------|
| 0 | — (invalid) | `Err(OutOfRange)` |
| 1 | ARP | `Poly` |
| 2 | CHORD | `Unison` |
| 3 | UNISON | `Chord` |
| 4 | POLY | `Arp` |
| 5 | — | `ArpLatch` |

**[community]** isnotinvain's list is `['NONE','ARP','CHORD','UNISON','POLY']`. loguetools
`xd.py` says "1:ARP, 2:CHORD (Mono verified), 3:UNISON, 4:POLY" and maps an og POLY patch to
4. **[hw]** hints:

- `quintal.rs:650` stores 4 "as saved by hardware" and calls it "ArpLatch". Quintal plays its
  own arpeggios as MIDI notes, so a POLY hardware setting is plausible.
- The logue-manager capture (see [Test vectors](#test-vectors)) also has byte 21 = 4.

Rust maps `byte − 1` onto `VoiceModeType`, whose program values are Poly 0, Unison 1, Chord 2,
Arp 3, ArpLatch 4 (`param/enums.rs:278-283`, `synth.rs:660-671`). That reverses the order:
byte 4 becomes `Arp`, which also contradicts quintal's own comment. The Rust doc comment
(`synth.rs:657-659`) gives a third ordering. The default byte 3 is commented "Poly"
(`synth.rs:715`); by the spec it is UNISON. The byte itself is stored raw, so round-trips are
unaffected.

#### P19 — JOYSTICK / CV IN / AFTER TOUCH ASSIGN (bytes 113, 115, 118, 120, 155)

| Byte | Spec P19 | Rust `ModAssignTarget` label |
|------|----------|------------------------------|
| 0 | GATE TIME | None |
| 1 | PORTAMENTO | GateTime |
| 2 | V.M DEPTH | PitchBend |
| 3 | VCO1 PITCH | CutoffVelocity |
| 4 | VCO1 SHAPE | AmpVelocity |
| 5 | VCO2 PITCH | MultiCtrl1 |
| 6 | VCO2 SHAPE | MultiCtrl2 |
| 7 | CROSS MOD | MultiCtrl3 |
| 8 | MULTI SHAPE | MultiCtrl4 |
| 9 | VCO1 LEVEL | Vco1Pitch |
| 10 | VCO2 LEVEL | Vco1Shape |
| 11 | MULTI LEVEL | Vco2Pitch |
| 12 | CUTOFF | Vco2Shape |
| 13 | RESONANCE | CrossModDepth |
| 14 | A.EG ATTACK | MultiShape |
| 15 | A.EG DECAY | MultiShiftShape |
| 16 | A.EG SUSTAIN | Cutoff |
| 17 | A.EG RELEASE | Resonance |
| 18 | EG ATTACK | AmpLevel |
| 19 | EG DECAY | AmpPan |
| 20 | EG INT | EgAttack |
| 21 | LFO RATE | EgDecay |
| 22 | LFO INT | EgSustain |
| 23 | MOD FX SPEED | EgRelease |
| 24 | MOD FX DEPTH | LfoRate |
| 25 | REVERB TIME | LfoInt |
| 26 | REVERB DEPTH | DelayDryWet |
| 27 | DELAY TIME | ReverbDryWet |
| 28 | DELAY DEPTH | PortamentoTime |

Both community parsers (isnotinvain, gekart) agree with the spec column **[community]**. The
Rust labels (`param/enums.rs:514-543`) match neither. The Rust **range** 0..28 is right, so
parsing and round-trips work; only the meaning is wrong.

#### P21 — MICRO TUNING (byte 122)

| Byte | Spec P21 | Rust `MicroTuning` (`param/enums.rs:553-592`) |
|------|----------|-----------------------------------------------|
| 0–7 | Equal Temp, Pure Major, Pure Minor, Pythagorean, Werckmeister, Kirnburger, Slendro, Pelog | same (spelled `Werkmeister`, `Kirnberger`) |
| 8–10 | Ionian, Dorian, Aeolian | same |
| 11–13 | Major Penta, Minor Penta, Reverse | MajorBlues, MinorBlues, Diminished |
| 14–19 | AFX001~AFX006 | CombiDiminished, MajorPentatonic, MinorPentatonic, RagaChalanata, RagaGamanashrama, RagaTodi |
| 20–22 | DC001~DC003 | RagaShruti, UserScale1, UserScale2 |
| 23–38 | — (invalid) | UserScale3~6, UserOctave1~6, Reverse1, Reverse2, AFX001~004 |
| 128–133 | USER SCALE 1~6 | `Err` |
| 134–139 | USER OCTAVE 1~6 | `Err` |

The blob byte is stored raw (`micro_tuning: u8`). Only `micro_tuning_enum()` is wrong
(`synth.rs:690-693`). The NRPN encoding of micro tuning differs from the blob encoding; see
[NRPN map](nrpn-map.md) and [tuning](tuning.md).

### Value curves (display semantics)

None of these affect the bytes. They matter only for converting between raw values and
musical units. CC/NRPN-side curves are on [parameter encodings](parameter-encodings.md).

| Note | Field | Mapping [spec] |
|------|-------|----------------|
| P2 | VOICE MODE DEPTH | POLY: 0~255 Poly, 256~1023 Duo (detune 0~1023). UNISON: 0~1023 = detune 0~50 cent. CHORD: 14 bands of ~73 (0~73 5th, 74~146 sus2, 147~219 m, 220~292 Maj, 293~365 sus4, 366~438 m7, 439~511 7, 512~585 7sus4, 586~658 Maj7, 659~731 aug, 732~804 dim, 805~877 m7b5, 878~950 mMaj7, 951~1023 Maj7b5). ARP: 13 bands of 78 (MANUAL 1/2, RISE 1/2, FALL 1/2, RISE FALL 1/2, POLY 1/2, RANDOM 1/2/3; the last band is 937~1023). |
| P5 | VCO PITCH | piecewise linear cents: 0~4 → −1200; 4~356 → −1200~−256; 356~476 → −256~−16; 476~492 → −16~0; 492~532 → 0; 532~548 → 0~16; 548~668 → 16~256; 668~1020 → 256~1200; 1020~1023 → +1200 |
| P10 | EG INT | 0~11 → −100%; 11~492 → −((492−v)² · 4641 · 100) / 0x40000000 %; 492~532 → 0%; 532~1013 → ((v−532)² · 4641 · 100) / 0x40000000 %; 1013~1023 → +100% |
| P11 | LFO RATE | BPM sync off: 0~1023 linear. BPM sync on: 16 bands of 64 → 4, 2, 1, 3/4, 1/2, 3/8, 1/3, 1/4, 3/16, 1/6, 1/8, 1/12, 1/16, 1/24, 1/32, 1/36 |
| — | PROGRAM LEVEL | 12~132 = −18~+6 dB. If linear, that is 0.2 dB per step and 102 = 0 dB (the Rust default, `synth.rs:798`) |

### Program name

The name is 12 bytes at offsets 4–15, ASCII, with no terminator.

| Allowed byte (P1 [spec]) | Characters |
|--------------------------|------------|
| 32, 33 | space, `!` |
| 35–42 | `#$%&'()*` |
| 44–47 | `,-./` |
| 48–57 | `0`–`9` |
| 58 | `:` |
| 63 | `?` |
| 65–90 | `A`–`Z` |
| 97–122 | `a`–`z` |
| **0** | NUL padding — not in P1, but written by hardware **[hw]** (`1c16566` bug 5) |

Excluded: `"` `+` `;` `<` `=` `>` `@` `[` `\` `]` `^` `_` `` ` `` `{` `|` `}` `~`, all
control characters except NUL, and every byte ≥ 0x7F.

Rust behaviour (`synth.rs:23-105`):

- `ProgramName::new([u8;12])` rejects any other byte with `InvalidProgramNameChar(b)`
  (`synth.rs:48-55`). A blob with such a byte fails to parse.
- `from_string(s)` copies at most 12 **bytes** and pads with **spaces** (0x20), not NUL
  (`synth.rs:67-72`).
- `as_str()` trims trailing spaces only. NUL padding from hardware remains in the string
  (`synth.rs:80-84`).
- The default name is 12 spaces (`synth.rs:101-105`).

### Program numbers, banks, and slots

| Concept | Range | Formula |
|---------|-------|---------|
| Program number `n` | 0..=499 | `ProgramNumber::MAX = 499` (`mod.rs:35`); 500 slots |
| Bank | 0..=4 | `n / 100` (`mod.rs:55-57`) |
| Slot in bank | 0..=99 | `n % 100` (`mod.rs:60-62`) |
| Front-panel number | 1..=500 | `n + 1` (the Rust `Display` prints `n`, `mod.rs:65-69`) |
| SysEx LSB byte | 0..=127 | `n & 0x7F` |
| SysEx MSB byte | 0..=3 | `(n >> 7) & 0x7F` |

The two program-number bytes are raw 7-bit bytes placed **before** the 7-bit-encoded payload.
They are not part of the codec stream. Decoding is `n = lsb | (msb << 7)`. Examples: 0 →
`00 00`; 42 → `2A 00`; 150 → `16 01`; 499 → `73 03`.

Selecting (not transferring) a program uses Bank Select and Program Change. The spec says CC0
= 0, CC32 = bank 0~4, Program Change = 0~99 (Korg MIDI Impl rev 1.01 §1 *1-1). The Rust
controller sends CC0 = bank and CC32 = slot (`controller.rs:464-468`). See
[realtime controller](../design/realtime-controller.md) and [errata](errata.md).

### SysEx messages that carry program data

The header (`F0 42 3g 00 01 51`, where `g` is the global MIDI channel 0–15) and status replies
are on [SysEx framing](sysex-framing.md). The codec is on [7-bit codec](korg-7bit-codec.md).
Function IDs are at `frame.rs:25,41,52,66`.

| # | Message | Fn | Dir | Exact bytes | Length |
|---|---------|----|-----|-------------|--------|
| (1) | CURRENT PROGRAM DATA DUMP REQUEST | `0x10` | host → XD | `F0 42 3g 00 01 51 10 F7` | 8 |
| (2) | PROGRAM DATA DUMP REQUEST | `0x1C` | host → XD | `F0 42 3g 00 01 51 1C ll mm F7` | 10 |
| (4) | CURRENT PROGRAM DATA DUMP | `0x40` | both | `F0 42 3g 00 01 51 40 <1171 encoded bytes> F7` | 1179 |
| (5) | PROGRAM DATA DUMP | `0x4C` | both | `F0 42 3g 00 01 51 4C ll mm <1171 encoded bytes> F7` | 1181 |

`1024` raw bytes encode to `146 × 8 + (2 + 1) = 1171` wire bytes. Behaviour per the spec
(exclusive formats (1)–(5)):

- `0x10` → the device answers `0x40` from the **edit buffer**, or `0x24` (DATA LOAD ERROR).
- `0x1C` → the device answers `0x4C` from **internal memory**, or `0x24`.
- Receiving `0x40` writes the edit buffer and replies `0x23` (ACK) or `0x24` (NAK). The device
  also *transmits* `0x40` unsolicited when the user runs "Program Dump".
- Receiving `0x4C` writes internal memory (it persists) and replies `0x23` or `0x24`. The
  device transmits `0x4C` for each program during "All Dump".
- A blob without `PREDSQ` is rejected with `0x26` DATA FORMAT ERROR **[hw]** (`1c16566` bug
  3).

The Rust request/response wrappers are `SysexTransaction::{request_current_program,
send_current_program, request_program, send_program}` (`transaction.rs:218-259`). See
[SysEx transactions](../design/sysex-transactions.md).

#### Parse algorithm for `0x4C` (Rust order of checks)

```text
parse_program_dump(msg):
  if len(msg) < 10                    -> InvalidHeader("message too short")
  if msg[0] != F0 or msg[last] != F7  -> InvalidHeader
  if msg[1] != 42                     -> InvalidHeader
  if msg[2] & F0 != 30                -> InvalidHeader      ; channel nibble is ignored
  if msg[3..6] != 00 01 51            -> InvalidHeader
  if msg[6] != 4C                     -> WrongFunctionId{expected 4C, found}
  n = msg[7] | msg[8] << 7            ; bit 7 of msg[7]/msg[8] not checked
  if n > 499                          -> InvalidProgramNumber(n)
  raw = decode_7bit(msg[9 .. last])   ; error if any wire byte has bit 7 set
  blob = ProgramData.from_bytes(raw)  ; PayloadTooShort if len(raw) < 1024
  return (n, blob)
```

`parse_current_program_dump` delegates header checks to the generic frame parser, then
requires function `0x40` (`mod.rs:210-220`). `ProgramData::from_bytes` requires
`len ≥ 1024` and **ignores extra bytes** (`mod.rs:131-152`). It then parses the synth half
(`PROG` magic, name, strict enums) and the sequencer half (`PREDSQ` magic).

### The 448-byte blob claim: not implemented

`README.md:226-227` and `CLAUDE.md` state that `.mnlgxdpreset` files from firmware v2.10+ use
448-byte blobs and that "both formats are supported". **The Rust code does not support this.**
No code path accepts a blob shorter than 1024 bytes. There is no `.mnlgxdpreset` reader or
writer. `ProgramData::from_bytes` rejects anything under 1024 bytes with `PayloadTooShort`.
**[community]** In loguetools' `patch_ident` table, 448 bytes is the **og minilogue /
monologue** program size, and 1024 is the xd size. The README claim most likely confuses the
two products. See [file formats](file-formats.md#mnlgxdpreset).

## Invariants

- `synth_size == 156`, `sequencer_size == 868`, `program_size == 1024`, and
  `synth_size + sequencer_size == program_size`.
- `blob[0..4] == "PROG"` after parse and before serialize.
- Every 10-bit field value is `≤ 1023`. `read_10bit` never returns > 1023.
- `write_10bit(b, o, v)` leaves `b[o+1] & 0xFC` unchanged. Assert old and new upper bits are
  equal.
- Each 10-bit offset `o` satisfies `o + 1 ≤ 155`. No two field spans overlap.
- Every name byte is in the P1 set or is `0x00`.
- `program_number ≤ 499`. `bank ≤ 4`. `slot ≤ 99`. `msb ≤ 3`. `lsb ≤ 0x7F`.
- `encoded_len(1024) == 1171`. A current dump is 1179 bytes and a stored dump is 1181 bytes.
- `serialize(parse(blob)) == blob` for every blob that parses (byte-exact, all 1024 bytes).

## Errata & gotchas

| # | Topic | Spec says | Evidence / implementation | Provenance |
|---|-------|-----------|---------------------------|------------|
| 1 | 10-bit byte order | Bit column `H:0~7` then `L:0~7` | low byte first; bits 9..8 in the next byte | **[hw]** `1c16566`, `quintal.rs:716`; **[community]** |
| 2 | Upper 6 bits of 10-bit high byte | not mentioned (implied unused) | device stores data there; writers must preserve it | **[hw]** `1c16566` bug 2 |
| 3 | Byte 148 bits 7..4 | Reserved | non-zero on hardware; must be preserved | **[hw]** `1c16566` bug 4 |
| 4 | Name padding | P1 lacks 0x00 | hardware pads with NUL | **[hw]** `1c16566` bug 5 |
| 5 | Dump size | message diagrams: "384Bytes(7bit) → 336Bytes(8bit)" | TABLE 2 spans 0–1023; Rust sends 1024 → 1171 and the device accepts it **[hw]**. 336 is the **prologue** blob size (loguetools `patch_ident`), so the diagram is copy-paste **[community]** | **[spec]** erratum |
| 6 | Program-number bits | request: "LSB bit 7~0, MSB bit 8"; dump: "LSB bit 6~0, MSB bit 7", with `0000 000p` drawn | neither diagram can address 256–499 as drawn; Rust and logue-manager use `lsb = n & 0x7F`, `msb = n >> 7` | **[spec]** inconsistent; **[rust]**, **[community]** |
| 7 | VOICE MODE TYPE | "1 ARP, 2 CHORD, 2 UNISON, 3 POLY", range 1~4 | community: 1 ARP, 2 CHORD, 3 UNISON, 4 POLY. Rust: 1 Poly … 4 Arp, 5 ArpLatch | **[spec]** typo; Rust mapping suspect |
| 8 | SYNC/RING polarity | 0 = ON, 1 = OFF | Rust follows the spec. isnotinvain, gekart and loguetools all treat 1 = ON | conflict, **unverified on [hw]** |
| 9 | ASSIGN targets (P19) | 0 GATE TIME … 28 DELAY DEPTH | Rust labels are a different list | Rust bug (labels) |
| 10 | MICRO TUNING (P21) | sparse 0–22, 128–139 | Rust lists 0–38 dense | Rust bug (enum) |
| 11 | PORTAMENTO range | Range column "0,1", description 0~127 | 0~127 | **[spec]** typo |
| 12 | EG TARGET | Bit column "H:0~7" on a 1-byte field | single byte 0~2 | **[spec]** typo |
| 13 | REVERB SUB TYPE | Range column 0~19; list 0~17 | Rust accepts 0~17 only | **[spec]** inconsistent |
| 14 | DRY WET | "0~1024" | a 10-bit field holds 0~1023 | **[spec]** typo |
| 15 | USER PARAM TYPE | values 0~2 | factory patches use 3 | **[hw]** (`param/enums.rs:477-480`) |
| 16 | Bank select | CC0 = 0, CC32 = bank | Rust sends CC0 = bank, CC32 = slot | conflict, no [hw] capture (wayfinder) |
| 17 | MOD FX TYPE byte | P12: 1 = CHORUS, 2 = ENSEMBLE | quintal set Ensemble over CC (`quintal.rs:441`), and the hardware saved byte 89 = 1. Quintal's comment calls that "Ensemble" (`quintal.rs:701`), but Rust `mod_fx_type_enum` maps 1 → Chorus. Either the CC value table or P12's 1-based numbering is wrong | conflict; see [CC map](cc-map.md) |
| 18 | 448-byte presets | — | README claims support; nothing implemented | README wrong |
| 19 | `0x4C` from internal memory | S1: old programs use `SEQD`, converted to `SQ` on load | a real `0x4C` reply carries `PREDSEQD`. Rust `SequencerParams::from_bytes` requires `PREDSQ`, so `parse_program_dump` / `request_program` **fail with `InvalidMagic`** for such slots | **[hw]** logue-manager capture; Rust bug |
| 20 | ARP bytes in stored data | — | the stored copy has 1022–1023 = `00 00`; the edit buffer shows `37 05` | **[hw]** capture |

## Test vectors

All from Rust unit tests unless marked.

| Vector | Bytes / values | Source |
|--------|----------------|--------|
| 10-bit read | `00 00` → 0; `FF 03` → 1023; `FF FF` → 1023 (upper bits masked) | `helpers.rs:86-104` |
| 10-bit read at offset 1 | `AA 00 01` @1 → 256 | `helpers.rs:106-110` |
| 10-bit write 0 into `FF FF` | → `00 FC` | `helpers.rs:113-119` |
| 10-bit write 1023 / 0xFFFF into `00 00` | → `FF 03` | `helpers.rs:121-135` |
| 10-bit write 0x1FF into `00 A4` | → `FF A5` (upper 6 bits kept) | `helpers.rs:137-145` |
| Hardware cutoff | `99 02` → 665 (Replicant xd) **[hw]** | `helpers.rs:156-161` |
| Hardware reverb time | 870 → byte 107 = `0x66` **[hw]** | `quintal.rs:716` |
| SYNC/RING | blob 34 = 1, 35 = 0 → `sync=false`, `ring=true`; re-serializes to 1, 0 | `synth.rs:1128-1142` |
| Byte 148/149 all types = 2 | 148 = `0x0A`, 149 = `0xAA` | `synth.rs:1223-1243` |
| Byte 148 flags | `0x01 \| 0x02<<2 \| 0xF0` = `0xF9` → types 1, 2, flags `0xF0`, round-trips | `synth.rs:1245-1256` |
| Voice mode | byte 3 → `Chord` (Rust); 1..4 → Poly, Unison, Chord, Arp; 0 → error | `synth.rs:1158-1213` |
| Mod FX type | 1..5 → Chorus..User; 0 → error | `synth.rs:1167-1221` |
| Program request (n = 0) | length 10, `[6] = 1C`, `[7] = 00`, `[8] = 00` | `mod.rs:458-466` |
| Program request (n = 499) | `[7] = 0x73`, `[8] = 0x03` | `mod.rs:468-475` |
| Current request | length 8, `[6] = 0x10` | `mod.rs:417-422` |
| Dump round trips | n = 42 on channel 5; n = 499; n = 150 with name "Stored", reverb on, BPM 2000 | `mod.rs:477-525` |
| All channels | channels 0–15 round-trip the program number | `mod.rs:527-535` |
| Too short | `F0 42 30 00 01 51 4C F7` → error | `mod.rs:505-509` |
| Wrong function | a `0x40` message passed to `parse_program_dump` → error | `mod.rs:497-503` |

The Rust tests contain no captured device dump. The only real-device bytes in the repo are the
two single-field vectors above.

### Real hardware captures (community) **[hw]**

`irgendwatt/logue-manager` @a04b2ea ships two real XD replies for the same program,
"TyoCityLoop": `doc/analyze/dump_current_editbuf.syx` (a `0x40` reply, 1179 bytes) and
`doc/analyze/dump_saved_slot2.syx` (a `0x4C` reply for n = 1, 1181 bytes). Both payloads are
1171 encoded bytes and decode to 1024 bytes. They make good Zig fixtures
([test plan](../zig/test-plan.md)). Decoded facts:

| Offset | Bytes | Meaning |
|--------|-------|---------|
| wire 0–15 | `F0 42 3B 00 01 51 40 00 50 52 4F 47 54 79 6F 00` | global ch. 12; first codec group = MSB byte `00` + `PROGTyo` |
| 4–15 | `54 79 6F 43 69 74 79 4C 6F 6F 70 00` | "TyoCityLoop" + **NUL** pad |
| 16, 21 | `02`, `04` | octave centre; voice mode 4 with depth 0 (POLY per community) |
| 24–25 | `0D 02` | VCO1 pitch 525 (≈ 0 cents) |
| 30–31 | `DD 01` | VCO2 pitch 477 |
| 34, 35 | `00`, `00` | sync, ring: ON/ON per the spec, OFF/OFF per community |
| 54–55 | `FF 03` | VCO1 level 1023 |
| 107–108 | `F8 02` | reverb time 760 |
| 113, 120 | `0C`, `0C` | assign 12 = CUTOFF (spec P19; Rust label `Vco2Shape`) |
| 122, 123, 124 | `00`, `0C`, `32` | equal temperament, scale key 0, tuning 0 cents |
| 135 | `66` | program level 102 = 0 dB |
| 148, 149 | `FF`, `FF` | all user-param types = 3; reserved nibble = `0xF` |
| 150, 151–152, 153–154 | `0D`, `00 02`, `00 02` | transpose 0; dry/wet 512, 512 |
| 155 | `17` | aftertouch assign 23 (MOD FX SPEED per spec) |

In both captures the upper 6 bits of all 34 10-bit high bytes are **zero**. That neither
confirms nor refutes the `1c16566` claim that the device stores flags there.

The two dumps differ at only these offsets: 60–61 (cutoff, edited after loading), 155,
161–163 and 1022–1023. **The stored-slot (`0x4C`) dump carries the old firmware-1.xx
sequencer magic `SEQD` at 160–163 and zeros at 1022–1023.** The edit-buffer (`0x40`) copy
carries `SQ` + `FF FF` and ARP values `37 05`. This is the conversion described in spec note
S1, observed on a real device. See [sequencer blob](sequencer-blob.md#magic-pred--sq-or-seqd).

## Rust design

- `ProgramData { synth: SynthParams, sequencer: SequencerParams, raw_blob:
  Option<Box<[u8;1024]>> }` (`mod.rs:74-97`). `from_bytes` keeps the input as `raw_blob`.
  `to_bytes` overlays the typed fields onto it. `PartialEq` ignores `raw_blob`
  (`mod.rs:99-103`). See [byte fidelity](../design/byte-fidelity.md).
- `SynthParams` is a flat struct with about 120 public fields. 10-bit fields are `u16`.
  Enums are typed where the crate has an enum. Fields whose enum is unreliable or 1-based
  (`voice_mode_type`, `mod_fx_type`, `micro_tuning`, `select_vpm`/`select_user`, user param
  types) are raw `u8` with `*_enum()` accessors (`synth.rs:110-357`, `synth.rs:660-703`).
- Parsing is all-or-nothing: one unknown value in a strict enum byte fails the whole blob.
- Serialization returns a `Vec<u8>` (`ProgramData`) or `[u8;156]` (`SynthParams`). SysEx
  builders allocate `Vec`s.
- *Incidental Rust idioms:* `Option<Box<…>>` for the base, `Vec` returns, `thiserror` enums,
  `Display` impls.
- *Essential design:* a 1024-byte buffer is the source of truth. Fields are accessed by
  explicit offsets. 10-bit writes preserve the upper bits. Serialization is an overlay onto
  the last-read blob. The program number is validated at construction.

## Zig / TigerStyle notes

- **The blob is the model.** Store `[1024]u8` and read and write fields with explicit offset
  accessors. Do not use `packed struct`. The 10-bit fields are unaligned little-endian pairs
  whose high byte shares bits with unknown device data. A packed struct would need 34
  `u2`+`u6` splits and still could not express "preserve bits 7..2". Offset accessors make
  overlay semantics free: there is nothing to forget, because unmodelled bits are never
  copied out.
- Use a comptime field table such as `.{ .name = "cutoff", .offset = 60, .kind = .u10 }` and
  `comptime` asserts that spans do not overlap, stay within `[4, 156)`, and that exactly 34
  `u10` entries exist.
- Use exact integer types: `u10` for continuous values, `u7` for MIDI-range bytes, `u9` for
  the program number (`assert(n <= 499)`), and `u2` for user-param types.
- Parse should check sizes and magics only, so it never fails on a legal-but-unknown enum
  byte. Accept both sequencer magics, `PREDSQ` and `PREDSEQD`, because real `0x4C` replies
  carry the old one. Expose `validate(blob) -> error{…}!void` and typed getters that return
  `error.FieldValueInvalid` per field. This keeps byte fidelity (forward-compatible with new
  firmware) and still gives strict checking where a caller wants it. Pair the assertions:
  `validate` before encode and after decode.
- Error set: `BlobError || FrameError` from [errors](../design/errors.md). The Rust names map as
  `PayloadTooShort` → `BlobTooShort`, `InvalidMagic` → `MagicMismatch`, `InvalidProgramNameChar`
  → `ProgramNameByteInvalid`, `InvalidProgramNumber` → `ProgramNumberInvalid`, `OutOfRange` →
  `FieldValueInvalid`, `WrongFunctionId` → `FunctionIdMismatch`, `InvalidHeader` → the specific
  `FrameError` tag, and a bad 7-bit byte → `WireByteInvalid`.
- Use caller-provided buffers: `[1181]u8` covers the largest program message and `[1024]u8`
  the blob. Encoding writes into a slice of exactly `1171` bytes. See
  [memory budget](../zig/memory-budget.md).
- Fix the known label bugs in the Zig enums: P3 order, the P19 list, the P21 sparse list.
  Keep sync/ring polarity behind one named constant until it is confirmed on hardware.

```zig
// sketch
const program_size = 1024;
const synth_size = 156;
comptime { assert(synth_size + sequencer_size == program_size); }

fn read_u10(blob: *const [program_size]u8, offset: u16) u10 {
    assert(offset + 1 < synth_size);
    return @intCast(@as(u16, blob[offset]) | (@as(u16, blob[offset + 1] & 0x03) << 8));
}

fn write_u10(blob: *[program_size]u8, offset: u16, value: u10) void {
    const upper = blob[offset + 1] & 0xFC;
    blob[offset] = @truncate(value);
    blob[offset + 1] = upper | @as(u8, @intCast(value >> 8));
    assert(blob[offset + 1] & 0xFC == upper);
    assert(read_u10(blob, offset) == value);
}
```

## Open questions

- **SYNC/RING polarity.** The spec says 0 = ON. Three community tools say 1 = ON. Capture a
  `0x40` dump with sync on and ring off to settle it. The Rust code inherits the spec's
  convention, and `quintal.rs:662-663` writes `sync = ring = true` → bytes 0, 0. The
  logue-manager capture has both bytes 0. Under the spec that means sync and ring are both
  on, which is unusual for a user patch. This weakly favours the community reading but proves
  nothing.
- **VOICE MODE TYPE.** Confirm 1 ARP, 2 CHORD, 3 UNISON, 4 POLY with one dump per mode. If
  confirmed, the Rust `voice_mode_type_enum` and its default comment are wrong.
- **What lives in the upper 6 bits** of 10-bit high bytes and in byte 148 bits 7..4? This is
  unknown. Preserve it.
- **Byte 148 ordering.** Spec and loguetools: 148 = params 5/6, 149 = params 1–4. isnotinvain
  reads 148–149 as one little-endian `u16` with param 1 in the low bits, which contradicts
  the spec. Hardware data in 148's upper nibble supports neither reading conclusively.
- **MOD FX TYPE numbering.** A CC set to Ensemble produced a saved byte of 1. Quintal calls
  that byte "Ensemble"; P12 and Rust `mod_fx_type_enum` say 1 = CHORUS. Settle it with a dump
  after selecting each type on the front panel. Owned jointly with [CC map](cc-map.md).
- `parse_program_dump` does not check bit 7 of the two program-number bytes, and it never
  checks `len(raw) == 1024` exactly (extra bytes are ignored).
- `ProgramName::as_str` leaves NUL padding in place, so displayed names may carry `\0`.

## Sources

- `crates/minilogue-xd/src/sysex/program/mod.rs:27-69` — `ProgramNumber`.
- `crates/minilogue-xd/src/sysex/program/mod.rs:74-180` — `ProgramData`, raw blob, overlay.
- `crates/minilogue-xd/src/sysex/program/mod.rs:192-309` — the four SysEx builders/parsers.
- `crates/minilogue-xd/src/sysex/program/mod.rs:324-535` — tests.
- `crates/minilogue-xd/src/sysex/program/synth.rs:23-105` — `ProgramName`.
- `crates/minilogue-xd/src/sysex/program/synth.rs:372-514` — `from_bytes` (offset table).
- `crates/minilogue-xd/src/sysex/program/synth.rs:520-654` — `to_bytes_with_base`.
- `crates/minilogue-xd/src/sysex/program/synth.rs:656-703` — 1-based enum accessors.
- `crates/minilogue-xd/src/sysex/program/synth.rs:707-824` — `Default`.
- `crates/minilogue-xd/src/sysex/program/synth.rs:950-1420` — tests and the synthetic blob.
- `crates/minilogue-xd/src/sysex/helpers.rs:16-35`, `:84-161` — 10-bit helpers and vectors.
- `crates/minilogue-xd/src/sysex/frame.rs:25`, `:41`, `:52`, `:66` — function IDs.
- `crates/minilogue-xd/src/sysex/transaction.rs:212-259` — transaction wrappers.
- `crates/minilogue-xd/src/param/enums.rs:278-283`, `:477-480`, `:514-543`, `:553-592`.
- `crates/minilogue-xd/src/controller.rs:464-468` — bank select as sent.
- `crates/minilogue-xd/examples/quintal.rs:441`, `:642-735` — hardware-observed values.
- `git show 1c16566` — five hardware-found bugs.
- `README.md:219-228`; `docs/wayfinder/research/xd-midi-capabilities.md:51-57`.
- Korg MIDI Impl rev 1.01 — exclusive formats (1), (2), (4), (5); TABLE 2; notes P1–P24;
  NOTE 2 (status codes); §1 *1-1 (bank select).
- [community] isnotinvain/minilogue-xd-util `xd_prog_bin.py` @6f8343c; gekart `mnlgxd.py`
  gist; gazzar/loguetools `xd.py`, `common.py` @ba0fc46; irgendwatt/logue-manager
  `logue_lib.py` @a04b2ea. See [community references](../sources/community-references.md).
- **[hw]** irgendwatt/logue-manager @a04b2ea `doc/analyze/dump_current_editbuf.syx` (`0x40`)
  and `doc/analyze/dump_saved_slot2.syx` (`0x4C`): real device replies, decoded for this page.
