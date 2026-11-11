---
title: CC map (every Control Change parameter)
category: protocol
sources:
  - crates/minilogue-xd/src/param/cc.rs
  - crates/minilogue-xd/src/param/encoding.rs
  - crates/minilogue-xd/src/param/enums.rs
  - crates/minilogue-xd/src/device.rs
  - crates/minilogue-xd/tests/cc_coverage.rs
  - docs/wayfinder/research/xd-midi-capabilities.md
related:
  - protocol/parameter-encodings.md
  - protocol/nrpn-map.md
  - protocol/midi-messages.md
  - protocol/errata.md
  - protocol/program-blob.md
  - design/stepped-params.md
  - design/realtime-controller.md
  - zig/memory-budget.md
  - sources/korg-midi-implementation.md
  - glossary.md
updated: 2026-10-11
---

# CC map (every Control Change parameter)

> The minilogue xd exposes 64 Control Change numbers. The Rust crate types each one as a
> `CcParam` variant: 31 are **10-bit continuous** parameters (sent as a CC63 low-bits message, then
> the parameter's own CC), 16 are **stepped** enums, 3 are **on/off** switches, 2 are
> **context-dependent** sub-type selectors, and 12 carry a **raw 7-bit** value (bank select,
> modulation, data entry, NRPN address, damper, CV in). This page gives the full table with transmit
> and receive mappings, and the `CcParamReceiver` rules. It also covers the stale-CC63 cross-talk
> that a reimplementation must decide on deliberately.

## Facts

### Kinds

| Kind | Count | Transmit (TX) | Receive (RX, Rust) | Messages per write |
|---|---|---|---|---|
| 10-bit | 31 | `Bn 3F low` then `Bn cc high`, see [encodings](parameter-encodings.md) | `((high & 0x7F) << 3) \| pending_low_or_0` | 2 |
| stepped | 16 | `Bn cc tx_value(enum)` | `enum.from_rx_value(v)`: band lookup, total over 0..127 | 1 |
| on/off | 3 | `true` → `7F`, `false` → `00` | `v >= 64` | 1 |
| context-dependent | 2 | raw value, caller must choose it | raw `U7`; the caller decodes it by context | 1 |
| raw 7-bit | 12 | raw value | raw `U7` | 1 |

Source: `cc.rs:29-171` (variants), `cc.rs:174-242` (`cc_number`), `cc.rs:253-343`
(`to_cc_messages`), `cc.rs:384-487` (`feed`).

### Full table

Columns:

- **Name** is the `device::cc_name` string (`device.rs:264-339`); it is used for display.
- **TX values** are the exact bytes that `to_cc_messages` emits.
- **RX** is how `CcParamReceiver::feed` decodes the value.
- **Spec** gives the Korg rev 1.01 notes: `*1-4` / `*5-4` = 10-bit via CC63; `*2-xx` = TX values;
  `*6-xx` = RX bands.
- Stepped enum details (variant order, bands, display strings) are in
  [stepped params](../design/stepped-params.md).

| CC | Hex | `CcParam` variant | Name (`cc_name`) | Kind | TX values | RX (Rust) | Spec |
|---|---|---|---|---|---|---|---|
| 0 | `0x00` | `BankSelectMsb` | Bank Select MSB | raw | `vv` | raw | `vv=0`, `*1-1`/`*5-1`; see [MIDI messages](midi-messages.md#bank-select-and-program-numbers) |
| 1 | `0x01` | `Modulation1` | Modulation 1 (Joystick +Y) | raw | `vv` | raw | 0~127 |
| 2 | `0x02` | `Modulation2` | Modulation 2 (Joystick -Y) | raw | `vv` | raw | 0~127 |
| 5 | `0x05` | `PortamentoTime` | Portamento Time | raw | `vv` | raw | 0~127 |
| 6 | `0x06` | `DataEntryMsb` | Data Entry MSB | raw | `vv` | raw | NRPN data high part, see [NRPN map](nrpn-map.md) |
| 16 | `0x10` | `AmpEgAttack` | Amp EG Attack | 10-bit | `3F lo`, `10 hi` | 10-bit | `*1-4`/`*5-4` |
| 17 | `0x11` | `AmpEgDecay` | Amp EG Decay | 10-bit | `3F lo`, `11 hi` | 10-bit | `*1-4`/`*5-4` |
| 18 | `0x12` | `AmpEgSustain` | Amp EG Sustain | 10-bit | `3F lo`, `12 hi` | 10-bit | `*1-4`/`*5-4` |
| 19 | `0x13` | `AmpEgRelease` | Amp EG Release | 10-bit | `3F lo`, `13 hi` | 10-bit | `*1-4`/`*5-4` |
| 20 | `0x14` | `EgAttack` | EG Attack | 10-bit | `3F lo`, `14 hi` | 10-bit | `*1-4`/`*5-4` |
| 21 | `0x15` | `EgDecay` | EG Decay | 10-bit | `3F lo`, `15 hi` | 10-bit | `*1-4`/`*5-4` |
| 22 | `0x16` | `EgInt` | EG Int | 10-bit | `3F lo`, `16 hi` | 10-bit | `*1-4`/`*5-4` |
| 23 | `0x17` | `EgTarget` | EG Target | stepped `EgTarget` | 0, 64, 127 | 3 bands | `*2-01`/`*6-01` |
| 24 | `0x18` | `LfoRate` | LFO Rate | 10-bit | `3F lo`, `18 hi` | 10-bit | `*1-4`/`*5-4` |
| 26 | `0x1A` | `LfoInt` | LFO Int | 10-bit | `3F lo`, `1A hi` | 10-bit | `*1-4`/`*5-4` |
| 27 | `0x1B` | `VoiceModeDepth` | Voice Mode Depth | 10-bit | `3F lo`, `1B hi` | 10-bit | `*1-4`/`*5-4` |
| 28 | `0x1C` | `ModFxTime` | Mod FX Time | 10-bit | `3F lo`, `1C hi` | 10-bit | `*1-4`/`*5-4` |
| 29 | `0x1D` | `ModFxDepth` | Mod FX Depth | 10-bit | `3F lo`, `1D hi` | 10-bit | `*1-4`/`*5-4` |
| 32 | `0x20` | `BankSelectLsb` | Bank Select LSB | raw | `vv` | raw | `vv=0~4` (bank), `*1-1`/`*5-1` |
| 33 | `0x21` | `MultiLevel` | Multi Level | 10-bit | `3F lo`, `21 hi` | 10-bit | `*1-4`/`*5-4` |
| 34 | `0x22` | `Vco1Pitch` | VCO 1 Pitch | 10-bit | `3F lo`, `22 hi` | 10-bit | `*1-4`/`*5-4`; cents curve in [program blob](program-blob.md) |
| 35 | `0x23` | `Vco2Pitch` | VCO 2 Pitch | 10-bit | `3F lo`, `23 hi` | 10-bit | `*1-4`/`*5-4` |
| 36 | `0x24` | `Vco1Shape` | VCO 1 Shape | 10-bit | `3F lo`, `24 hi` | 10-bit | `*1-4`/`*5-4` |
| 37 | `0x25` | `Vco2Shape` | VCO 2 Shape | 10-bit | `3F lo`, `25 hi` | 10-bit | `*1-4`/`*5-4` |
| 39 | `0x27` | `Vco1Level` | VCO 1 Level | 10-bit | `3F lo`, `27 hi` | 10-bit | `*1-4`/`*5-4` |
| 40 | `0x28` | `Vco2Level` | VCO 2 Level | 10-bit | `3F lo`, `28 hi` | 10-bit | `*1-4`/`*5-4` |
| 41 | `0x29` | `CrossModDepth` | Cross Mod Depth | 10-bit | `3F lo`, `29 hi` | 10-bit | `*1-4`/`*5-4` |
| 43 | `0x2B` | `Cutoff` | Cutoff | 10-bit | `3F lo`, `2B hi` | 10-bit | `*1-4`/`*5-4` |
| 44 | `0x2C` | `Resonance` | Resonance | 10-bit | `3F lo`, `2C hi` | 10-bit | `*1-4`/`*5-4` |
| 48 | `0x30` | `Vco1Octave` | VCO 1 Octave | stepped `VcoOctave` | 0, 42, 84, 127 | 4 bands | `*2-02`/`*6-02` |
| 49 | `0x31` | `Vco2Octave` | VCO 2 Octave | stepped `VcoOctave` | 0, 42, 84, 127 | 4 bands | `*2-02`/`*6-02` |
| 50 | `0x32` | `Vco1Wave` | VCO 1 Wave | stepped `VcoWave` | 0, 64, 127 | 3 bands | `*2-03`/`*6-03` |
| 51 | `0x33` | `Vco2Wave` | VCO 2 Wave | stepped `VcoWave` | 0, 64, 127 | 3 bands | `*2-03`/`*6-03` |
| 53 | `0x35` | `MultiType` | Multi Type | stepped `MultiType` | 0, 64, 127 | 3 bands | `*2-04`/`*6-04` |
| 54 | `0x36` | `MultiShape` | Multi Shape | 10-bit | `3F lo`, `36 hi` | 10-bit | `*1-4`/`*5-4` |
| 56 | `0x38` | `LfoTarget` | LFO Target | stepped `LfoTarget` | 0, 64, 127 | 3 bands | `*2-05`/`*6-05` |
| 57 | `0x39` | `LfoWave` | LFO Wave | stepped `LfoWave` | 0, 64, 127 | 3 bands | `*2-06`/`*6-06` |
| 58 | `0x3A` | `LfoMode` | LFO Mode | stepped `LfoMode` | 0, 64, 127 | 3 bands | `*2-07`/`*6-07` |
| 59 | `0x3B` | `VoiceModeDepthAlt` | Voice Mode Depth (alt) | 10-bit (Rust) | `3F lo`, `3B hi` | 10-bit | "no display"; RX `*5-4`, **TX lacks `*1-4`** |
| 63 | `0x3F` | `DataEntryLsb` | Data Entry LSB (10-bit low 3 bits) | low-part carrier | `vv` | sets pending low = `vv & 7`; returns raw `vv` | "LSB VALUE", `*1-4`/`*5-4` |
| 64 | `0x40` | `Damper` | Damper / Hold | raw | `vv` | raw | TX "Hold1" `0,127`; RX "DAMPER" 0~127 |
| 80 | `0x50` | `Sync` | Sync | stepped `Sync` | 0, 127 | 0–63 Off, 64–127 On | `*2-08`/`*6-08` (RX note reversed, see errata) |
| 81 | `0x51` | `Ring` | Ring | stepped `Ring` | 0, 127 | 0–63 Off, 64–127 On | `*2-09`/`*6-09` (RX note reversed) |
| 83 | `0x53` | `CutoffKeytrack` | Cutoff Keytrack | stepped `CutoffKeytrack` | 0, 64, 127 | 3 bands | `*2-10`/`*6-10` |
| 84 | `0x54` | `CutoffDrive` | Cutoff Drive | stepped `CutoffDrive` | 0, 64, 127 | 3 bands | `*2-11`/`*6-11` |
| 88 | `0x58` | `ModFxType` | Mod FX Type | stepped `ModFxType` | 0, 38, 64, 84, 127 | 5 bands | `*2-12`/`*6-12` |
| 89 | `0x59` | `DelaySubType` | Delay Sub Type | stepped `DelaySubType` | 20 values: 0, 7, 13, … 116, 127 | 20 bands | `*2-13`/`*6-13` |
| 90 | `0x5A` | `ReverbSubType` | Reverb Sub Type | stepped `ReverbSubType` | 18 values: 0, 8, 15, … 117, 127 | 18 bands | `*2-14`/"`*5-14`" |
| 92 | `0x5C` | `ModFxOnOff` | Mod FX On/Off | on/off | 0, 127 | `v >= 64` | `*2-15`/`*6-15` |
| 93 | `0x5D` | `DelayOnOff` | Delay On/Off | on/off | 0, 127 | `v >= 64` | `*2-16`/`*6-16` |
| 94 | `0x5E` | `ReverbOnOff` | Reverb On/Off | on/off | 0, 127 | `v >= 64` | `*2-17`/`*6-17` |
| 96 | `0x60` | `ModFxSubType` | Mod FX Sub Type | context (CC88) | `vv` | raw | `*2-18`…`*2-21` + USER list / `*6-18` |
| 98 | `0x62` | `NrpnLsb` | NRPN LSB | raw | `vv` | raw | `*3` |
| 99 | `0x63` | `NrpnMsb` | NRPN MSB | raw | `vv` | raw | `*3` |
| 103 | `0x67` | `MultiSelect` | Multi Select | context (CC53) | `vv` | raw | `*2-22`…`*2-24` / `*6-19` |
| 104 | `0x68` | `MultiShiftShape` | Multi Shift Shape | 10-bit | `3F lo`, `68 hi` | 10-bit | `*1-4`/`*5-4` |
| 105 | `0x69` | `DelayTime` | Delay Time | 10-bit | `3F lo`, `69 hi` | 10-bit | `*1-4`/`*5-4` |
| 106 | `0x6A` | `DelayDepth` | Delay Depth | 10-bit | `3F lo`, `6A hi` | 10-bit | `*1-4`/`*5-4` |
| 107 | `0x6B` | `DelayDryWet` | Delay Dry/Wet | 10-bit | `3F lo`, `6B hi` | 10-bit | `*1-4`/`*5-4` |
| 108 | `0x6C` | `ReverbTime` | Reverb Time | 10-bit | `3F lo`, `6C hi` | 10-bit | `*1-4`/`*5-4` |
| 109 | `0x6D` | `ReverbDepth` | Reverb Depth | 10-bit | `3F lo`, `6D hi` | 10-bit | `*1-4`/`*5-4` |
| 110 | `0x6E` | `ReverbDryWet` | Reverb Dry/Wet | 10-bit | `3F lo`, `6E hi` | 10-bit | `*1-4`/`*5-4` |
| 118 | `0x76` | `CvIn1` | CV In 1 | raw | `vv` | raw | `*1-5`/`*5-5`: CV IN mode = Modulation only |
| 119 | `0x77` | `CvIn2` | CV In 2 | raw | `vv` | raw | `*1-5`/`*5-5` |

In the 10-bit TX column, `lo = v & 0x07` and `hi = (v >> 3) & 0x7F`. Every value byte is a
7-bit data byte. The kinds are the Rust classification [rust]. The CC numbers and value sets match
Korg rev 1.01 §1-1 and §2-1 [spec], except where the [errata](#errata--gotchas) say otherwise.

All other CC numbers are **not** `CcParam`s, including 3, 4, 7–15, 25, 30, 31, 38, 42, 45–47, 52,
55, 60–62, 65–79, 82, 85–87, 91, 95, 97, 100–102, 111–117 and 120–127. The spec's receive-only
channel-mode CCs (120 All Sound Off, 122 Local Sw, 123 All Note Off) are modelled as separate
message types; see [MIDI messages](midi-messages.md#control-changes-that-are-not-ordinary-parameters).

### Context-dependent CCs (96, 103)

The meaning of CC96 depends on the current Mod FX type (CC88). The meaning of CC103 depends on the
current Multi type (CC53). Rust stores these as raw `U7` and never decodes them in the receiver
[rust]. To decode one, keep the last type value and choose the enum:

| CC | Context value | Enum | TX values | RX bands |
|---|---|---|---|---|
| 96 | Mod FX type = Chorus | `ModFxSubTypeChorus` (8) | 0, 16, 32, 48, 64, 80, 96, 127 | 16 wide |
| 96 | Ensemble | `ModFxSubTypeEnsemble` (3) | 0, 64, 127 | 0–42, 43–85, 86–127 |
| 96 | Phaser | `ModFxSubTypePhaser` (8) | as Chorus | 16 wide |
| 96 | Flanger | `ModFxSubTypeFlanger` (8) | as Chorus | 16 wide |
| 96 | User | `ModFxSubTypeUser` (16) | 0, 8, …, 112, 127 | 8 wide |
| 103 | Multi type = Noise | `MultiSelectNoise` (4) | 0, 42, 84, 127 | 32 wide (see errata) |
| 103 | VPM | `MultiSelectVpm` (16) | 0, 8, …, 112, 127 | 8 wide |
| 103 | User | `MultiSelectUser` (16) | 0, 8, …, 112, 127 | 8 wide |

The NRPNs (`0x0D`–`0x0F` in Rust, `0x11`–`0x13` in the spec) give an unambiguous way to set the
multi selectors; see [NRPN map](nrpn-map.md).

### Receiver (`CcParamReceiver`)

State: one `TenBitReceiver` (`pending_low: optional u3`, empty at start), `cc.rs:365-367`. The
receiver **ignores the MIDI channel**, and the returned `CcParam` carries none. Use one receiver
per channel [rust].

| Incoming CC | Effect on `pending_low` | Returns |
|---|---|---|
| 63 | set to `v & 0x07` | `DataEntryLsb(v)` (raw, unmasked) |
| 31 ten-bit CCs (16–22, 24, 26–29, 33–37, 39–41, 43, 44, 54, 59, 104–110) | consumed, then empty | 10-bit variant, `((v & 0x7F) << 3) \| (pending or 0)` |
| 16 stepped CCs | unchanged | enum via `from_rx_value` (never fails for 0..127) |
| 92, 93, 94 | unchanged | `v >= 64` |
| 96, 103 | unchanged | raw `U7` |
| 0, 1, 2, 5, 6, 32, 64, 98, 99, 118, 119 | unchanged | raw `U7` |
| any other CC (e.g. 3, 7, 11, 38, 100, 101, 120, 121, 123) | **reset** to empty | `None` |

Source: `cc.rs:384-487`. Non-CC messages (notes, pitch bend) never reach this receiver; it takes
`ControlChange` only.

**Cross-talk [rust].** Data entry (CC6) and NRPN addresses (CC98/99) leave `pending_low` alone.
So the low part of an NRPN multi-byte write stays pending and is added to the next 10-bit CC.

```text
B0 63 00  B0 62 10  B0 3F 04  B0 06 3E    ; NRPN multi shape (noise) = 500
B0 2B 40                                  ; Cutoff high = 64 → decodes as 516, not 512
```

A stepped or on/off CC between CC63 and its 10-bit partner does not break the pair either. For
example, `B0 3F 07`, `B0 50 7F`, `B0 2B 40` decodes Cutoff = 519. Whether the device does the same
is unknown; see [Open questions](#open-questions).

## Invariants

- `cc_number(variant)` maps 64 variants one-to-one to 64 distinct CC numbers. Every number is
  `<= 119`.
- For every variant `p`: `feed_all(to_cc_messages(p)) == p` on a fresh receiver. Exceptions:
  `DataEntryLsb` (returns itself but also changes state) and stepped/on-off values (round trip
  through TX value then band).
- `to_cc_messages` returns 2 messages for exactly the 31 ten-bit kinds, with `[0].controller == 63`;
  it returns 1 message for every other kind. Max is 2 messages, or 6 bytes.
- Every emitted data byte is `<= 0x7F`. The status byte is `0xB0 | channel`.
- Receiver: `pending_low <= 7`. Output of a 10-bit decode is `<= 1023`. After a 10-bit decode,
  `pending_low` is empty.
- Each stepped TX value lies in its own RX band, so a stepped TX→RX round trip is the identity.
- The stepped RX bands of each enum cover 0..127 with no gaps or overlaps (Rust tests check
  `from_rx_value` for every byte).

## Errata & gotchas

| # | Item | Spec says | Rust does | Provenance / verdict |
|---|---|---|---|---|
| 1 | CC59 resolution | TX row has no `*1-4`; RX row has `*5-4` | 10-bit both ways | [spec] is internally inconsistent; [rust] chose 10-bit. No [hw] capture. |
| 2 | Sync/Ring RX polarity | `*6-08`/`*6-09`: "0~63, 64~127 = ON, OFF" | 0–63 Off, 64–127 On | [spec] typo. It contradicts `*2-08`/`*2-09` (OFF, ON = 0, 127) and `*6-15`–`*6-17`. Rust follows the consistent reading. |
| 3 | Mod FX type RX bands | `*6-12`: "…, 77~102, 102~127" (102 in two bands) | 77–102 Flanger, 103–127 User | [spec] typo; Rust removes the overlap. |
| 4 | Reverb RX note label | labelled `*5-14` | — | [spec] typo for `*6-14`. The bands match Rust. |
| 5 | Noise select RX bands | `*6-19` NOISE lists eight 16-wide bands for four values | four 32-wide bands (0–31, 32–63, 64–95, 96–127) | [spec] copy-paste error. Rust agrees with TX 0/42/84/127 and with VCO octave `*6-02`. |
| 6 | CC96 USER TX list | "0, 8, …, 104, 112, 128" (15 + impossible 128) | 0, 8, …, 112, 127 (16 values) | [spec] typo; 128 is not a 7-bit value. |
| 7 | Delay/reverb on-off CC numbers | TXT rev 1.01: 93/94. Owner's manual E8 chart: 94/95 | 93/94 | Editions conflict [spec]; Rust follows the TXT. See [errata](errata.md). |
| 8 | CC7 Volume, CC11 Expression | absent from the TXT; listed in the E8 chart | not modelled (`None`, resets the pending low) | [spec] editions conflict. `xd-midi-capabilities.md:54`. |
| 9 | Bank select | CC0 = 0, CC32 = bank 0~4 | `CcParam` is raw; the controller sends CC0 = bank, CC32 = slot | see [MIDI messages](midi-messages.md#bank-select-and-program-numbers) |
| 10 | Hold1 vs Damper | TX `0,127` (Hold1), RX 0~127 (Damper) | raw both ways | Rust is a superset. |
| 11 | CV IN 1/2 | sent only in Modulation mode with PolyChain = Master, not over USB (`*1-5`); received only in Modulation mode (`*5-5`) | unconditional | Device-side gating; see [poly chain](poly-chain.md). |
| 12 | CC63 not GM | — | — | Not the LSB of CC31. Standard tools mislabel it. See [encodings](parameter-encodings.md). |
| 13 | Low part first | `*1-4`: "lower 3 bits are first sent" | CC63 first | Opposite of GM MSB→LSB order [spec]. |

Rust-internal inconsistencies [rust]:

- `cc_name`'s doc comment says "63 CC numbers", but there are 64 names and the test asserts 64
  (`device.rs:264-268`, `device.rs:447-451`).
- `KNOWN_CCS` in `tests/cc_coverage.rs:19-23` lists 63 numbers and omits 23 (EG Target). The
  receiver handles 23, so this is only a test-coverage gap.
- Stepped enum `Display` strings are Rust variant names (`"Sqr"`, `"16'"`), not the `cc_name`
  style or the spec's upper-case names. See [stepped params](../design/stepped-params.md).

## Test vectors

| Case | Wire / values | Status | Source |
|---|---|---|---|
| CC63=5, CC43=64 | Cutoff 517 | asserted | `cc.rs:793-803` |
| CC43=64 alone | Cutoff 512 | asserted | `cc.rs:806-816` |
| CC63=7, CC16=10, CC17=10 | AmpEgAttack 87, then AmpEgDecay 80 (low consumed) | asserted | `cc.rs:819-829` |
| CC63=7, CC43=127 | 1023 | asserted | `cc.rs:831-840` |
| CC63=0, CC43=0 | 0 | asserted | `cc.rs:842-851` |
| CC63=127, CC43=0 | 7 (only 3 bits kept) | asserted | `cc.rs:1325-1336` |
| CC63=7, CC3=…, CC43=10 | 80 (unknown CC3 reset the low) | asserted | `cc.rs:1166-1177` |
| CC63=5, CC43=100 | Cutoff 805 | asserted | `tests/cc_coverage.rs:101-121` |
| CC 3, 4, 7, 120, 127 | `None` | asserted | `cc.rs:1156-1164` |
| Cutoff 517 → messages | controllers `[63, 43]` → `B0 3F 05 B0 2B 40` | controllers asserted | `cc.rs:1180-1194` |
| Cutoff round trip | 0, 1, 7, 8, 511, 512, 1023 | asserted | `cc.rs:1196-1210` |
| CC92 = 63 / 64 | false / true | asserted | `cc.rs:1052-1063` |
| CC88 = 64 | `ModFxType::Phaser` | asserted | `cc.rs:1005-1013` |
| Sync On / Off; Vco1Octave Eight | `B0 50 7F` / `B0 50 00`; `B0 30 2A` (42) | values asserted | `cc.rs:1296-1315` |
| Channel 5 (zero-based) | status `B5` kept on TX | asserted | `cc.rs:1285-1293` |
| Known CC sweep | every `KNOWN_CCS` entry yields `Some`; every `UNKNOWN_CCS` entry `None` | asserted | `tests/cc_coverage.rs:19-60` |
| Cross-talk (derived) | `B0 63 00 B0 62 10 B0 3F 04 B0 06 3E B0 2B 40` → last decodes 516 | derived from `cc.rs:384-487` | — |

## Rust design

- `CcParam` is one `#[non_exhaustive]` enum with payload types `TenBitParam`, a stepped enum,
  `bool` or `U7` (`cc.rs:28-171`). Three large `match` statements map variant → CC number,
  variant → messages, and CC number → variant. Keeping those three in sync by hand is the main
  maintenance risk. A table removes it (see below).
- `to_cc_messages` returns `Result<Vec<ControlChange>>`. The `Result` only covers `U7::new`
  failures, which cannot happen for valid typed values. The `Vec` is incidental: the length is
  always 1 or 2.
- `CcParamReceiver::feed` returns `Option<CcParam>`. Stepped decodes use `.ok()`, but bands cover
  0..127, so they never return `None`.
- Essential design: the CC63-first pairing, the pending low, assume-0, total stepped bands, and the
  `>= 64` switch threshold. Incidental: the `Vec`, the `Result` on encode, and the separate
  `TenBitParam` newtype.

## Zig / TigerStyle notes

- Replace the three matches with one comptime table indexed by CC number. An unused slot is
  `.none`:

  ```zig
  // sketch
  const CcKind = enum(u8) { none, ten_bit, stepped, on_off, context, raw, low_part };
  const CcInfo = struct { kind: CcKind, param: ParamId, stepped: ?SteppedId = null };
  const cc_table: [128]CcInfo = comptime build_cc_table();

  comptime {
      var ten_bit_count: u32 = 0;
      for (cc_table) |info| {
          if (info.kind == .ten_bit) ten_bit_count += 1;
      }
      assert(ten_bit_count == 31);
  }
  ```

- Encoding writes into a caller buffer and returns the length. The bound is
  `max_cc_messages_per_param = 2` and `max_bytes = 6` (no running status). Assert
  `len == 2` iff `kind == .ten_bit`.
- Value types: `u10` for 10-bit, `u7` for raw/context, `bool` for on/off, `enum(u8)` for stepped.
  A tagged union `CcValue` keyed by `ParamId` replaces `CcParam`.
- Receiver: `struct { pending_low: ?u3 = null }` with `fn feed(self: *CcReceiver, cc: u7, value: u7)
  ?CcParam`. Make the reset policy explicit with a comptime-known `options` field. Possible
  policies: Rust-compatible (only unknown CCs reset), or strict (anything but CC63 and a 10-bit
  CC resets). Document the choice; it changes decoding results (see cross-talk).
- Keep one receiver per MIDI channel (`[16]CcReceiver`, 16 bytes) or filter by channel before
  feeding. The Rust design assumes the caller filters.
- When CC parsing and NRPN parsing share one input stream, consider routing CC63 to the NRPN state
  machine while an NRPN is addressed, and to the 10-bit receiver otherwise. This removes the
  cross-talk. It is a deliberate deviation; record it in the port's errata.
- Context-dependent CCs: keep `last_mod_fx_type: ?ModFxType` and `last_multi_type: ?MultiType` in
  the receiver if the port wants typed sub-types. Return raw `u7` when the context is unknown.

## Open questions

- Does the device clear a pending CC63 on NRPN or stepped CCs? Does it time out? (Same question as on
  [encodings](parameter-encodings.md#open-questions).) The answer decides whether the Rust
  cross-talk matches hardware.
- Is CC59 transmitted with a CC63 low part? Capture the device output while turning the
  voice-mode-depth knob in a mode that emits CC59.
- Which CC numbers does current firmware use for delay/reverb on/off (93/94 vs 94/95)? Does it
  accept CC7/CC11? Capture against firmware ≥ 2.10.
- Does the device accept arbitrary 0..127 on CC64 as a half-damper, or treat it as a switch?

## Sources

- `crates/minilogue-xd/src/param/cc.rs:28-171` — `CcParam` variants.
- `crates/minilogue-xd/src/param/cc.rs:174-242` — `cc_number`.
- `crates/minilogue-xd/src/param/cc.rs:253-352` — `to_cc_messages`, `single_cc`.
- `crates/minilogue-xd/src/param/cc.rs:357-487` — `CcParamReceiver`.
- `crates/minilogue-xd/src/param/cc.rs:793-1336` — receiver and round-trip tests.
- `crates/minilogue-xd/src/param/encoding.rs:355-411` — `TenBitReceiver`.
- `crates/minilogue-xd/src/device.rs:264-339`, `:447-451` — `cc_name` and its count test.
- `crates/minilogue-xd/tests/cc_coverage.rs:19-121` — known/unknown sweeps, 805 vector.
- `docs/wayfinder/research/xd-midi-capabilities.md:52-56` — edition and resolution discrepancies.
- Korg MIDI Impl rev 1.01 §1-1 (TX table, `*1-1`…`*1-5`, `*2-01`…`*2-24`), §2-1 (RX table,
  `*5-1`…`*5-5`, `*6-01`…`*6-19`) — see [source page](../sources/korg-midi-implementation.md).
