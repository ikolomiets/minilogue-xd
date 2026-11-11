---
title: NRPN map (address table, wire sequence, receiver state machine)
category: protocol
sources:
  - crates/minilogue-xd/src/param/nrpn.rs
  - crates/minilogue-xd/src/param/encoding.rs
  - crates/minilogue-xd/src/param/enums.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/tests/nrpn_coverage.rs
  - crates/minilogue-xd/src/sysex/program/synth.rs
  - docs/wayfinder/research/xd-midi-capabilities.md
related:
  - protocol/parameter-encodings.md
  - protocol/cc-map.md
  - protocol/program-blob.md
  - protocol/errata.md
  - protocol/poly-chain.md
  - design/stepped-params.md
  - design/realtime-controller.md
  - zig/memory-budget.md
  - sources/korg-midi-implementation.md
  - glossary.md
updated: 2026-10-11
---

# NRPN map (address table, wire sequence, receiver state machine)

> Settings with no Control Change of their own (program name, voice mode type, multi-engine
> selectors and per-engine shapes, bend range, joystick and CV assignment, micro tuning, VPM and
> user-oscillator parameters, master volume) are written with **NRPN**. That is
> `CC99 = address high`, `CC98 = address low`, an optional `CC63 = data low part`, then
> `CC6 = data high part`. **The Rust address table disagrees with Korg's spec for 32 of its 51
> parameters.** Rust looks like a gap-free renumbering of the spec's sparse table. No hardware
> capture settles which is right, so this page gives both tables and treats the address map as a
> switchable *profile*. The 4-state receiver state machine and the value decoding rules are the
> same under either profile.

## Facts

### Wire sequence

Every NRPN write sends, in this order (status `Bn`, n = channel) [spec][rust]:

| # | CC | Data | When |
|---|---|---|---|
| 1 | 99 `0x63` | address high (always `0x00`) | always |
| 2 | 98 `0x62` | address low | always |
| 3 | 63 `0x3F` | data low part | only for 10-bit, 8-bit hi-res and 14-bit values |
| 4 | 6 `0x06` | data high part (or the whole value for 7-bit values) | always |

- 7-bit values use 3 messages (9 bytes). Wide values use 4 messages (12 bytes). See
  [encodings](parameter-encodings.md) for how values are split.
- Rust never sends CC38 (standard Data Entry LSB), never sends a null RPN/NRPN
  (`CC101/100 = 7F`), and never uses Data Increment/Decrement (CC96/97) [rust]. The spec documents
  none of these for NRPN either [spec].
- The address is re-sent for every value. Rust never relies on a "current address" left by an
  earlier write.
- The spec's MSB column shows `--` for every row. Rust always sends `0x00` and its receiver accepts
  only `0x00` (see [Open questions](#open-questions)).

Source: `nrpn.rs:192-399`. The spec's NRPN rules are in Korg MIDI Impl rev 1.01 `*3`, `*3-1`…`*3-4`.

### Value encodings

| Encoding | Data messages | Range | Rust payload type | Decode on receive (Rust) |
|---|---|---|---|---|
| 7-bit raw | `CC6 = v` | 0..127 | `U7` or `u8` | raw. **No semantic range check** (e.g. bend range 13..127 accepted) |
| 7-bit enum | `CC6 = program_value` | per enum | stepped enum, see [stepped params](../design/stepped-params.md) | `from_program_value`: exact match, otherwise dropped |
| 7-bit bool | `CC6 = 0` or `1` | 0, 1 | `bool` | `v != 0` |
| 10-bit | `CC63 = v & 7`, `CC6 = v >> 3` | 0..1023 | `TenBitSysex` | `(high << 3) \| (low & 7)`; missing CC63 → low 0 |
| 8-bit hi-res | `CC63 = v & 7`, `CC6 = v >> 3` | 0..200 | `EightBitHighRes` | `((high & 0x1F) << 3) \| (low & 7)`; > 200 → dropped |
| 14-bit | `CC63 = v & 0x7F`, `CC6 = v >> 7` | 0..16383 | `FourteenBitParam` | `((high & 0x7F) << 7) \| (low & 0x7F)`; missing CC63 → low 0 |

A CC63 sent before the `CC6` of a 7-bit parameter is accepted and ignored [rust].

### Address table: spec vs Rust

"Spec" is Korg MIDI Impl rev 1.01 `*3` [spec]. "Rust" is `NrpnParam::address` (`nrpn.rs:136-190`)
[rust]. The address high byte is 0 in both. Rust TX values are the enum *program values*; see
[stepped params](../design/stepped-params.md) for the full value lists.

| Parameter (spec name) | Spec addr | Rust addr | Rust variant | Encoding | Spec value range | Agree? |
|---|---|---|---|---|---|---|
| PROGRAM NAME 1–12 | `0x01`–`0x0C` | `0x00`–`0x0B` | `ProgramName(idx 0..11, U7)` | 7-bit raw | ASCII, `*note P1` charset | **no** (off by one) |
| VOICE MODE TYPE | `0x10` | `0x0C` | `VoiceModeType` | enum | 0~4, `*3-5` | **no** (address and value order) |
| MULTI SELECT NOISE | `0x11` | `0x0D` | `MultiSelectNoise` | enum | 0~3, `*note P6` | **no** |
| MULTI SELECT VPM | `0x12` | `0x0E` | `MultiSelectVpm` | enum | 0~15, `*note P7` | **no** |
| MULTI SELECT USER | `0x13` | `0x0F` | `MultiSelectUser` | enum | 0~15, `*note P8` | **no** |
| MULTI SHAPE NOISE | `0x14` | `0x10` | `MultiShapeNoise` | 10-bit | 0~1023, `*3-1` | **no** |
| MULTI SHAPE VPM | `0x15` | `0x11` | `MultiShapeVpm` | 10-bit | 0~1023 | **no** |
| MULTI SHAPE USER | `0x16` | `0x12` | `MultiShapeUser` | 10-bit | 0~1023 | **no** |
| MULTI SHIFT SHAPE NOISE | `0x17` | `0x13` | `MultiShiftShapeNoise` | 10-bit | 0~1023 | **no** |
| MULTI SHIFT SHAPE VPM | `0x18` | `0x14` | `MultiShiftShapeVpm` | 10-bit | 0~1023 | **no** |
| MULTI SHIFT SHAPE USER | `0x19` | `0x15` | `MultiShiftShapeUser` | 10-bit | 0~1023 | **no** |
| BEND RANGE (+) | `0x20` | `0x16` | `BendRangePlus(u8)` | 7-bit raw | 0~12 = Off, 1..12 notes | **no** |
| BEND RANGE (−) | `0x21` | `0x17` | `BendRangeMinus(u8)` | 7-bit raw | 0~12 | **no** |
| JOYSTICK ASSIGN (+) | `0x22` | `0x18` | `JoystickAssignPlus` | enum `ModAssignTarget` | 0~28, `*note P19` | **no** |
| JOYSTICK RANGE (+) | `0x23` | `0x19` | `JoystickRangePlus` | 8-bit | 0~200 = −100..+100 %, `*3-2` | **no** |
| JOYSTICK ASSIGN (−) | `0x24` | `0x1A` | `JoystickAssignMinus` | enum | 0~28 | **no** |
| JOYSTICK RANGE (−) | `0x25` | `0x1B` | `JoystickRangeMinus` | 8-bit | 0~200 | **no** |
| CV IN MODE | `0x28` | `0x1C` | `CvInMode` | enum | 0~2, `*note P20` | **no** |
| CV IN 1 ASSIGN | `0x29` | `0x1D` | `CvIn1Assign` | enum | 0~28 | **no** |
| CV IN 1 RANGE | `0x2A` | `0x1E` | `CvIn1Range` | 8-bit | 0~200 | **no** |
| CV IN 2 ASSIGN | `0x2B` | `0x1F` | `CvIn2Assign` | enum | 0~28 | **no** |
| CV IN 2 RANGE | `0x2C` | `0x20` | `CvIn2Range` | 8-bit | 0~200 | **no** |
| MICRO TUNING | `0x30` | `0x30` | `MicroTuning` | enum | 0~127, `*3-6` (0–22, 112–123) | address yes, **values differ from 11 up** |
| SCALE KEY | `0x31` | `0x31` | `ScaleKey(u8)` | 7-bit raw | 0~24 = −12..+12 notes | yes (Rust doc says 0-11) |
| PROGRAM TUNING | `0x32` | `0x32` | `ProgramTuning(u8)` | 7-bit raw | 0~100 = −50..+50 cent | yes |
| LFO KEY SYNC | `0x34` | `0x33` | `LfoKeySync(bool)` | bool | 0, 1 | **no** |
| LFO VOICE SYNC | `0x35` | `0x34` | `LfoVoiceSync(bool)` | bool | 0, 1 | **no** |
| LFO TARGET OSC | `0x36` | `0x35` | `LfoTargetOsc` | enum | 0~3, `*note P22` | **no** |
| EG VELOCITY | `0x38` | `0x36` | `CutoffVelocity(U7)` | 7-bit raw | 0~127 | **no** |
| AMP VELOCITY | `0x39` | `0x37` | `AmpVelocity(U7)` | 7-bit raw | 0~127 | **no** |
| MULTI OCTAVE | `0x3A` | `0x38` | `MultiOctave(VcoOctave)` | enum | 0~3 = 16', 8', 4', 2' | **no** |
| MULTI ROUTING | `0x3B` | `0x39` | `MultiRouting` | enum | 0, 1 = Pre VCF, Post VCF | **no** |
| EG LEGATO | `0x3C` | `0x3A` | `EgLegato(bool)` | bool | 0, 1 | **no** |
| PORTAMENTO MODE | `0x3D` | `0x3B` | `PortamentoMode` | enum | 0, 1 = Auto, On | **no** |
| PORTAMENTO BPM SYNC | `0x3E` | `0x3C` | `PortamentoBpmSync(bool)` | bool | 0, 1 | **no** |
| PROGRAM LEVEL | `0x3F` | `0x3F` | `ProgramLevel(u8)` | 7-bit raw | 0~120 = −18..+6 dB | yes |
| VPM PARAM 1–6 (Feedback, Noise Depth, Shape Mod Int, Mod Attack, Mod Decay, Mod Key Track) | `0x40`–`0x45` | `0x40`–`0x45` | `VpmParam1`…`6` | 8-bit | 0~200 = −100..+100 % | yes |
| USER PARAM 1–6 | `0x48`–`0x4D` | `0x48`–`0x4D` | `UserParam1`…`6` | 8-bit | by type, `*note P23`/`P24` | yes |
| Program Transpose | `0x50` | `0x50` | `ProgramTranspose(u8)` | 7-bit raw | 1~25 = −12..+12 notes | yes |
| (MIDI AFTER TOUCH ASSIGN) | — | `0x51` | `MidiAfterTouchAssign` | enum | — (program blob only) | **Rust-only** |
| MASTER VOLUME | `0x7F` | `0x7F` | `MasterVolume` | 14-bit | 0~16383, `*3-3`, `*3-4` | yes |

Totals: 18 Rust variants agree on address, 32 differ, and 1 is Rust-only. Rust uses 62 addresses
(12 for the name); the spec uses 61.

Spec addresses with no parameter: `0x00`, `0x0D`–`0x0F`, `0x1A`–`0x1F`, `0x26`, `0x27`,
`0x2D`–`0x2F`, `0x33`, `0x37`, `0x46`, `0x47`, `0x4E`, `0x4F`, `0x51`–`0x7E`.

#### Where Rust writes land under the spec map

If the device follows the spec, a Rust write to address *A* changes the spec parameter at *A*:

| Rust addr | Rust parameter | Spec parameter at that address |
|---|---|---|
| `0x00` | name char 1 | — (unassigned) |
| `0x01`–`0x0B` | name chars 2–12 | NAME 1–11: every character lands one position early |
| `0x0C` | voice mode type | NAME 12 |
| `0x0D`–`0x0F` | multi selects | — |
| `0x10` | multi shape noise | VOICE MODE TYPE (presumably the CC6 high part becomes the value) |
| `0x11`–`0x13` | multi shape VPM/user, shift shape noise | SELECT NOISE / VPM / USER |
| `0x14`, `0x15` | shift shape VPM/user | SHAPE NOISE, SHAPE VPM |
| `0x16`, `0x17` | bend range +/− | SHAPE USER, SHIFT SHAPE NOISE |
| `0x18`, `0x19` | joystick assign +/range + | SHIFT SHAPE VPM, SHIFT SHAPE USER |
| `0x1A`–`0x1F` | joystick −, CV mode, CV 1, CV 2 assign | — |
| `0x20` | CV 2 range | BEND RANGE (+) |
| `0x33` | LFO key sync | — |
| `0x34`–`0x36` | LFO voice sync, LFO target osc, cutoff velocity | LFO KEY SYNC, LFO VOICE SYNC, LFO TARGET OSC |
| `0x37` | amp velocity | — |
| `0x38`–`0x3C` | multi octave, routing, EG legato, porta mode, porta BPM | EG VEL, AMP VEL, MULTI OCTAVE, MULTI ROUTING, EG LEGATO |

Example: `set_bend_range_plus(7)` sends `B0 63 00 B0 62 16 B0 06 07`. Under the spec map that sets
MULTI SHAPE USER high = 7, low assumed 0, so 56/1023.

#### Evidence on which map is right

- The spec's map is regular. Groups start at aligned addresses (`0x10`, `0x20`, `0x28`, `0x30`,
  `0x34`, `0x38`, `0x40`, `0x48`). Inside each group, parameters follow program blob (TABLE 2)
  byte order: blob 111–116 → `0x20`–`0x25`, 117–121 → `0x28`–`0x2C`, 122–124 → `0x30`–`0x32`,
  125–127 → `0x34`–`0x36`, 128–135 → `0x38`–`0x3F` [spec].
- Rust packs the early groups with no gaps, but keeps the spec's `0x3F`, `0x40`, `0x48`, `0x50` and
  `0x7F`. That leaves an unexplained hole at `0x3D`/`0x3E`, which is exactly where the spec puts
  PORTAMENTO MODE/BPM SYNC. A consistent renumbering would not leave that hole [rust].
- Rust's own test calls `0x3D` "not used" (`nrpn.rs:1365-1371`), but the spec puts PORTAMENTO MODE
  there [spec].
- No test, capture or commit in the repo checks any NRPN address against hardware [rust]. The
  hardware-verified fixes in git (for example `1c16566`) touch only the SysEx program blob.
- Verdict: the spec map is more likely to be correct, but this is **unverified**. A port should
  default to the spec profile, keep a Rust-compatible profile for interoperability, and add a
  hardware capture test. `xd-midi-capabilities.md:52` reaches the same conclusion ("resolve
  address-by-address"; do not apply a universal offset).

### Enum value orders that differ

| Parameter | Spec values | Rust program values | Effect |
|---|---|---|---|
| Voice mode type (NRPN) | `*3-5`: 0 ARP LATCH, 1 ARP, 2 CHORD, 3 UNISON, 4 POLY | 0 Poly, 1 Unison, 2 Chord, 3 Arp, 4 ArpLatch | Reversed; only Chord (2) agrees. |
| Micro tuning (NRPN) | `*3-6`: 0–10 as Rust; 11 Major Penta, 12 Minor Penta, 13 Reverse, 14–19 AFX001–006, 20–22 DC001–003, 112–117 user scale 1–6, 118–123 user octave 1–6 | 0–38: 11 MajorBlues … 21–26 user scales, 27–32 user octaves, 33/34 Reverse1/2, 35–38 AFX001–004 | Differs from 11 up. The Rust receiver drops spec values 39..127, including all user scales. |
| Assign targets (`*note P19`) | 0 GATE TIME, 1 PORTAMENTO, 2 V.M DEPTH, 3 VCO1 PITCH … 28 DELAY DEPTH | 0 None, 1 GateTime, 2 PitchBend, … 28 PortamentoTime | Names differ at every index. The count (29) matches, so numeric round trips still work. |

Full lists are in [stepped params](../design/stepped-params.md#microtuning-39-variants). The program
blob stores voice mode as 1~4 (`*note P3`, which is garbled) and micro tuning as 0~139 (user
entries at 128–139, i.e. NRPN value + 16). See [program blob](program-blob.md).

### Receiver FSM

State (`nrpn.rs:404-429`):

| State | Payload | Meaning |
|---|---|---|
| `Idle` | — | waiting for CC99 |
| `HaveMsb` | `msb` | CC99 seen |
| `Addressed` | `msb, lsb` | CC99 then CC98 seen |
| `HaveDataLsb` | `msb, lsb, data_low` | CC63 seen after the address (raw 0..127 stored; masked when decoded) |

Transitions (`feed`, `nrpn.rs:447-497`). Rows are the current state; columns are the incoming CC
number with data `v`:

| State \ CC | 99 | 98 | 63 | 6 | any other |
|---|---|---|---|---|---|
| `Idle` | → `HaveMsb(v)` | → `Idle` | → `Idle` | → `Idle`, no output | → `Idle` |
| `HaveMsb(m)` | → `HaveMsb(v)` | → `Addressed(m, v)` | → `Idle` | → `Idle`, no output | → `Idle` |
| `Addressed(m, l)` | → `HaveMsb(v)` | → `Idle` | → `HaveDataLsb(m, l, v)` | → `Idle`, **emit** `resolve(m, l, none, v)` | → `Idle` |
| `HaveDataLsb(m, l, d)` | → `HaveMsb(v)` | → `Idle` | → `Idle` | → `Idle`, **emit** `resolve(m, l, d, v)` | → `Idle` |

Properties:

- The order must be exactly CC99 → CC98 → [CC63] → CC6. CC98 before CC99 is ignored.
- A second CC63 aborts the message. It does **not** overwrite the first one.
- Every CC6 ends in `Idle`. Repeated CC6 messages without re-addressing produce nothing.
- Any other CC (modulation, CC38, CC100/101, a 10-bit CC) aborts the message.
- The channel is ignored. Callers must use one receiver per channel or filter first.
- At most one value is produced per `feed`.

`resolve(msb, lsb, data_low, data_high)` (`nrpn.rs:500-629`):

1. If `msb != 0`, return none.
2. Look up `lsb` in the address table. Unassigned → none.
3. Decode by encoding (see [Value encodings](#value-encodings)). Enum decode failure or an 8-bit
   value > 200 → none. Raw values pass through with no range check. A `data_low` given for a 7-bit
   parameter is ignored.
4. `lsb` `0x00`–`0x0B` → `ProgramName(lsb, data_high)`. The character is not checked against the
   `*note P1` charset.

### Spec side conditions

- MASTER VOLUME (`0x7F`): "Only sent when PolyChain=Master, and not sent via USB MIDI"
  (`*3-4`). See [poly chain](poly-chain.md). The Rust doc comment claims it uses MSB 1 under
  PolyChain Master (`nrpn.rs:21-25`). The code uses MSB 0 and neither the spec nor any test supports
  MSB 1 [rust].
- User parameter meaning depends on the oscillator's declared param type (`*note P24`): 0 Percent
  (0~101), 1 Bipolar (0~200), 2 Select (0~100 shown as 1~101). Real factory programs also use a
  type value 3 (`enums.rs:476-479`) [hw].

## Invariants

- Encoded sequence length is 3 for 7-bit/enum/bool values and 4 for 10/8/14-bit values. Never
  more than 4 messages (12 bytes).
- `seq[0].controller == 99`, `seq[1].controller == 98`, last `.controller == 6`. If the length
  is 4, `seq[2].controller == 63`.
- Address high is 0. Address low is `<= 0x7F` and is a key of the active address profile.
- Within one profile, the address → parameter map is one-to-one (the comptime check sees no
  duplicate keys).
- For each value `p` in range: `decode(encode(p)) == p` through a fresh FSM.
- After any CC6, the FSM state is `Idle`.
- The FSM's `data_low` is set only in state `HaveDataLsb`.
- Program-name index is `< 12`. Rust does not enforce this; a port must.

## Errata & gotchas

- **Addresses (32 of 51)** differ between Rust and spec rev 1.01. See the table and evidence above.
  Highest risk: Rust writes to `0x10`–`0x19` and `0x34`–`0x3C` change *other* spec parameters
  [spec][rust].
- **Program-name index is not validated** [rust]. `ProgramName(idx, ch)` sends LSB = `idx`. An
  idx of 12..127 aliases other parameters (12 = voice mode type in Rust's own map), and an idx
  ≥ 128 makes encoding fail (`U7::new`). `nrpn.rs:138`.
- **No semantic range checks on 7-bit raw parameters** in `to_midi_sequence` or `resolve`. Bend range
  13, scale key 100 and program level 121 all encode and decode [rust]. Only the controller's
  bend and level setters check ranges (see [realtime controller](../design/realtime-controller.md)).
- **Doc comments disagree with the spec** [rust]: `ScaleKey` "(0-11)" vs spec 0~24 (`nrpn.rs:76`).
  `MasterVolume` "MSB=1" (`nrpn.rs:24-25`). `CutoffVelocity` is the spec's "EG VELOCITY".
- **`MidiAfterTouchAssign` (`0x51`) is not in `*3`.** The spec has MIDI AFTER TOUCH ASSIGN only as
  program blob byte 155 (`*note P19` title). The NRPN address is a Rust invention [rust].
- **CC38 is not the data LSB.** A sender using standard NRPN (CC38 for the data LSB) aborts the Rust
  FSM. The xd uses CC63 [spec].
- **Interleaving with 10-bit CCs.** The low part sent on CC63 for an NRPN also sets the CC
  receiver's pending low, which is then added to the next 10-bit CC. See
  [CC map](cc-map.md#receiver-ccparamreceiver).
- `CLAUDE.md` (repo root) says "29 NRPN parameters". That is stale: the spec has 61 addresses,
  and Rust has 51 variants over 62 addresses.

## Test vectors

Channel 1 (`B0`). "Derived" bytes follow from asserted controller order plus the encoding.

| Case | Bytes | Status | Source |
|---|---|---|---|
| Bend range + = 7 (controller) | `B0 63 00` `B0 62 16` `B0 06 07` | asserted | `controller.rs:1143-1152` |
| Bend range − = 5 | `B0 63 00` `B0 62 17` `B0 06 05` | asserted | `controller.rs:1161-1169` |
| Micro tuning PureMajor | `B0 63 00` `B0 62 30` `B0 06 01` | asserted | `controller.rs:1178-1190` |
| Program level 0 dB | `B0 63 00` `B0 62 3F` `B0 06 5A` | asserted | `controller.rs:1193-1202` |
| Multi shape noise = 500 | `B0 63 00` `B0 62 10` `B0 3F 04` `B0 06 3E` | controllers asserted, bytes derived | `nrpn.rs:1404-1414` |
| Bend range + = 12 | 3 messages: 99, 98, 6 | asserted | `nrpn.rs:1416-1426` |
| Joystick range + = 150 | `B0 63 00` `B0 62 19` `B0 3F 06` `B0 06 12` | round trip asserted, bytes derived | `tests/nrpn_coverage.rs:51-57` |
| Master volume = 10000 | `B0 63 00` `B0 62 7F` `B0 3F 10` `B0 06 4E` | round trip asserted, bytes derived | `tests/nrpn_coverage.rs:59-65` |
| No CC63, (0, `0x10`), CC6 = 10 | `MultiShapeNoise(80)` | asserted | `nrpn.rs:1326-1339` |
| 10-bit round trips | 0, 1, 7, 8, 511, 512, 1023 | asserted | `nrpn.rs:964-979` |
| 8-bit round trips | 0, 1, 7, 8, 100, 199, 200 | asserted | `nrpn.rs:980-995` |
| 14-bit round trips | 0, 1, 127, 128, 8192, 16382, 16383 | asserted | `nrpn.rs:996-1011` |
| CC98 then CC6 without CC99 | none | asserted | `nrpn.rs:1278-1285` |
| CC99, CC98 `0x0C`, CC1, CC6 | none (aborted) | asserted | `nrpn.rs:1287-1296` |
| CC99, CC99, CC98 `0x0C`, CC6 = 0 | `VoiceModeType::Poly` | asserted | `nrpn.rs:1298-1309` |
| CC63 while idle, then a full sequence | `VoiceModeType::Poly` | asserted | `nrpn.rs:1311-1324` |
| CC6 while idle / after only CC99 | none | asserted | `nrpn.rs:1340-1352` |
| CC6 again after a resolved value | none | asserted | `nrpn.rs:1354-1361` |
| (0, `0x3D`) | none (spec: PORTAMENTO MODE) | asserted | `nrpn.rs:1365-1371` |
| MSB 2 | none | asserted | `nrpn.rs:1373-1379` |
| CC99, CC43, CC98, CC6 | none | asserted | `tests/nrpn_coverage.rs:85-126` |
| Program name char 3 = `'X'`; Unison; shape VPM 512; LFO key sync; bend + 7 | round trip | asserted | `tests/nrpn_coverage.rs:29-83` |

## Rust design

- `NrpnParam` is a 51-variant `#[non_exhaustive]` enum carrying typed payloads (`nrpn.rs:28-132`).
  Three hand-written matches (`address`, `to_midi_sequence`, `resolve`) must agree. They do today,
  but nothing enforces it except round-trip tests.
- `to_midi_sequence` returns `Result<Vec<ControlChange>>` with length 3 or 4. The errors come only
  from `U7::new` on a raw `u8` payload (> 127) or on a ProgramName index ≥ 128.
- `NrpnReceiver` is an enum state with payloads. `resolve` is a pure function of
  `(msb, lsb, data_low?, data_high)`. Those two pieces are the essential design; the `Vec`,
  `Option` and `Result` plumbing is incidental.
- The NRPN receiver and the CC receiver are independent. Each sees every CC if the caller feeds
  both, and neither knows about the other.

## Zig / TigerStyle notes

- One comptime parameter table drives encode and decode. The address is a field of the profile, not
  hard-coded in a switch:

  ```zig
  // sketch
  const NrpnEncoding = enum(u8) { raw7, enum7, bool7, ten_bit, eight_bit, fourteen_bit };
  const NrpnInfo = struct { param: NrpnParamId, encoding: NrpnEncoding, max: u14 };
  const AddressProfile = enum { korg_spec_1_01, rust_crate_0_1 };

  fn nrpn_table(comptime profile: AddressProfile) [128]?NrpnInfo {
      // Fill from a list of .{ address, info } pairs; assert no duplicate addresses.
  }
  ```

- Encode into a caller-provided buffer: `fn encode(param: NrpnValue, channel: u4, out: *[12]u8) u8`
  returns the byte count, 9 or 12. Assert the count, the CC order and that every data byte is
  `<= 0x7F`.
- Validate semantic ranges at encode time from the table's `max`. For example, bend range ≤ 12,
  program level ≤ 120, transpose 1..25, name index < 12 and the name charset. Return
  `error.ValueOutOfRange`. This is stricter than Rust; document it.
- FSM as a tagged union with `enum(u2)` tag: `idle`, `have_msb: u7`,
  `addressed: struct { msb: u7, lsb: u7 }`, `have_data_low: struct { msb: u7, lsb: u7, low: u7 }`.
  The transition function is a single `switch` on `(state, cc)`. It is O(1) with no loops.
- If the port has one CC dispatcher, route CC63 to the NRPN FSM when the FSM is `addressed`, and to
  the 10-bit CC receiver otherwise. This fixes the cross-talk; see [CC map](cc-map.md).
- Keep a profile-level test that encodes every parameter and checks the address bytes against a
  literal table. A hardware capture should later pin the `korg_spec_1_01` profile.

## Open questions

- **Which address map does the firmware use?** Capture the device's own NRPN output (for example,
  change bend range on the panel with MIDI Tx CC on) and compare it with both profiles.
- Does the device accept or emit a non-zero address MSB? The spec shows `--`. Does it accept a
  CC6 without a fresh CC99/CC98 (standard "current NRPN" behavior)? The Rust FSM drops it.
- Does the device treat CC38 as a data LSB?
- Spec `*3-5` lists voice mode as `0 ARP LATCH … 4 POLY`. Blob `*note P3` lists `1 ARP, 2 CHORD,
  2 UNISON, 3 POLY` (duplicated 2, missing latch). Rust has a third order. Which does the hardware
  use on the NRPN?
- MIDI aftertouch assign: is there a real NRPN for it?

## Sources

- `crates/minilogue-xd/src/param/nrpn.rs:21-132` — `NrpnParam` variants and doc comments.
- `crates/minilogue-xd/src/param/nrpn.rs:136-190` — address table.
- `crates/minilogue-xd/src/param/nrpn.rs:192-399` — `to_midi_sequence`.
- `crates/minilogue-xd/src/param/nrpn.rs:404-497` — FSM state and `feed`.
- `crates/minilogue-xd/src/param/nrpn.rs:500-646` — `resolve` and wide-value helpers.
- `crates/minilogue-xd/src/param/nrpn.rs:656-1526` — tests.
- `crates/minilogue-xd/tests/nrpn_coverage.rs:17-126` — round trips and reset test.
- `crates/minilogue-xd/src/controller.rs:1143-1202` — controller NRPN byte vectors.
- `crates/minilogue-xd/src/param/enums.rs:476-479` — user param type 3 in factory patches.
- `docs/wayfinder/research/xd-midi-capabilities.md:52` — NRPN address discrepancy.
- Korg MIDI Impl rev 1.01 `*3` (NRPN table), `*3-1`…`*3-6`; TABLE 2 `*note P1`, `P3`, `P6`–`P8`,
  `P19`–`P24` — see [source page](../sources/korg-midi-implementation.md).
