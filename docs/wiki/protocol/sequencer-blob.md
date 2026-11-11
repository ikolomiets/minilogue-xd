---
title: Program data blob — sequencer half (TABLE 2, offsets 156–1023)
category: protocol
sources:
  - crates/minilogue-xd/src/sysex/program/sequencer.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/sysex/enums.rs
  - crates/minilogue-xd/src/sysex/helpers.rs
  - crates/minilogue-xd/src/builder/sequence.rs
  - crates/minilogue-xd/examples/read_patch.rs
  - https://cdn.korg.com/us/support/download/files/5227b0b2f7c1da5acd0911ac3e8c4f38.txt
related:
  - protocol/program-blob.md
  - protocol/sysex-framing.md
  - protocol/korg-7bit-codec.md
  - protocol/file-formats.md
  - protocol/errata.md
  - design/byte-fidelity.md
  - design/builders.md
  - zig/test-plan.md
updated: 2026-10-11
---

# Program data blob — sequencer half (TABLE 2, offsets 156–1023)

> The last 868 bytes of every 1024-byte [program blob](program-blob.md) hold the 16-step
> sequencer:
>
> - a 34-byte header (magic, active-step bits, BPM, length, resolution, swing, default gate,
>   step and motion bitmasks, four motion-slot descriptors);
> - 16 step events of 52 bytes each (8 notes, 8 velocities, 8 gate+trigger bytes, 4 × 7 bytes
>   of packed 10-bit motion data);
> - two arpeggiator bytes.
>
> Two warnings. **The Rust crate reads and writes the first eight header bytes after the magic
> at the wrong offsets.** A real device capture confirms the spec layout **[hw]**. And
> internal-memory dumps may still carry the firmware-1.xx magic `SEQD`, which the Rust parser
> rejects.

## Facts

### Layout overview

Absolute offsets index the 1024-byte blob. Relative offsets (`rel = abs − 156`) index the
868-byte slice that Rust `SequencerParams` receives.

| Abs | Rel | Size | Content |
|-----|-----|------|---------|
| 156–189 | 0–33 | 34 | Header |
| 190–1021 | 34–865 | 16 × 52 = 832 | Step events; step `k` (0-based) at abs `190 + 52k` |
| 1022 | 866 | 1 | ARP gate time |
| 1023 | 867 | 1 | ARP rate |

`34 + 832 + 2 = 868` and `156 + 868 = 1024`. The last step spans abs 970–1021. The spec prints
"426~1021", which is a typo.

### Header — spec layout (authoritative)

Multi-byte fields are little-endian. In a step bitmask, bit `n` of the 16-bit LE value is step
`n + 1`: abs 162 bit 0 = step 1, abs 163 bit 7 = step 16.

| Abs | Rel | Width | Field | Range / encoding [spec] |
|-----|-----|-------|-------|-------------------------|
| 156–159 | 0–3 | 4 B | `'PRED'` | ASCII `50 52 45 44` |
| 160–161 | 4–5 | 2 B | `'SQ'` | ASCII `53 51` (old: see [Magic](#magic-pred--sq-or-seqd)) |
| 162–163 | 6–7 | u16 LE | Active step bits | bit n = step n+1 participates; 0,1 = Off,On |
| 164–165 | 8–9 | u12 LE | BPM × 10 | 100~3000 = 10.0~300.0; abs 165 uses bits 3..0 only (`H:0~3`) |
| 166 | 10 | u8 | Step length | 1~16 |
| 167 | 11 | u8 | Step resolution | 0~4 = 1/16, 1/8, 1/4, 1/2, 1/1 |
| 168 | 12 | u8 | Swing | −75~+75 (encoding unstated; community and Rust use `stored = swing + 75`) |
| 169 | 13 | u8 | Default gate time | 0~72 = 0%~100% |
| 170–171 | 14–15 | u16 LE | Step on/off bits | bit n = step n+1 on |
| 172–173 | 16–17 | u16 LE | Step motion on/off bits | bit n = motion plays on step n+1 |
| 174–175 | 18–19 | 2 B | Motion slot 1 parameter | [S2](#motion-slot-parameter-s2) |
| 176–177 | 20–21 | 2 B | Motion slot 2 parameter | S2 |
| 178–179 | 22–23 | 2 B | Motion slot 3 parameter | S2 |
| 180–181 | 24–25 | 2 B | Motion slot 4 parameter | S2 |
| 182–183 | 26–27 | u16 LE | Motion slot 1 step bits | bit n = slot 1 active on step n+1 |
| 184–185 | 28–29 | u16 LE | Motion slot 2 step bits | as above |
| 186–187 | 30–31 | u16 LE | Motion slot 3 step bits | as above |
| 188–189 | 32–33 | u16 LE | Motion slot 4 step bits | as above |

"Active step" (firmware 2.0+) selects which steps the playback cursor visits. "Step on/off"
mutes a step that is still visited. The user guide covers both ("Active Step" shift function;
step edit) — see [user guide](../sources/korg-user-guide.md).

### Header — what Rust actually does (bug)

`SequencerParams::from_bytes` / `to_bytes_with_base` use this layout (`sequencer.rs:183-198`,
`sequencer.rs:200-207`, `sequencer.rs:263-270`):

| Rel | Abs | Rust field | What is really there (spec + [hw] capture) |
|-----|-----|------------|---------------------------------------------|
| 0–5 | 156–161 | magic `PREDSQ` | ✔ same |
| 6–7 | 162–163 | `bpm` (u16 LE) | ✘ active step bits |
| 8 | 164 | `step_length` | ✘ BPM low byte |
| 9 | 165 | `step_resolution` | ✘ BPM high nibble |
| 10 | 166 | `swing` | ✘ step length |
| 11 | 167 | `default_gate_time` | ✘ step resolution |
| 12–13 | 168–169 | `active_steps` (u16 LE) | ✘ swing, default gate |
| 14–33 | 170–189 | `steps_on`, `motion_on`, slots, slot steps | ✔ same |

So the eight bytes at rel 6–13 are a **rotation** of the spec order: Rust puts the two
active-step bytes last instead of first. Everything from rel 14 onwards is correct.
**[hw]** The logue-manager edit-buffer capture has `FF FF D8 04 10 01 4B 48` at abs 162–169.
Under the spec layout that is all steps active, 124.0 BPM, 16 steps, 1/8 resolution, swing 0
and gate 72 — a coherent patch. Through the Rust layout it reads as `bpm = 65535`,
`step_length = 216`, `step_resolution = 4`, `swing = 16`, `default_gate_time = 1` and
`active_steps = 0x484B`.

Consequences:

- Byte-exact round-trips still hold: the same wrong offsets are used on read and on write,
  and `to_bytes_with_base` rewrites the same bytes. That is why the hardware round-trip test
  in `1c16566` (which never edited sequencer fields) passed.
- **Every semantic read is wrong.** `bpm_f32()`, `swing_signed()` and `read_patch.rs:143`
  ("BPM:") report garbage for real patches.
- **Every semantic write is wrong.** `SequenceBuilder::bpm` and the default `SequencerParams`
  write misplaced bytes. The default (`sequencer.rs:318-336`) serializes abs 162–169 as
  `B0 04 10 00 4B 36 FF FF`. A device would read that as: active steps 5, 6, 8 and 11 only
  (`0x04B0`); BPM raw 16 = 1.6 (below the 10.0 minimum); length 75; resolution 54; swing
  `0xFF`; gate 255. All but the first are out of range. The device response — a `0x26`
  DATA FORMAT ERROR, clamping, or misbehaviour — is **unknown**. Every
  [`PatchBuilder::build`](../design/builders.md) output carries this header.
- History: before `1c16566` the code had no magic (rel 0–5 "reserved") and BPM at rel 6. The
  fix added the `PREDSQ` check but did not insert the 2-byte active-step field.

### Magic: `PRED` + `SQ` or `SEQD`

Spec note S1: data saved with firmware 1.xx has a 4-byte `'SEQD'` at abs 160–163 and **no
active-step field**. When the device loads such data, it replaces `SEQD` with `SQ` and fills
162–163 with all-on (`FF FF`). Everything from abs 164 onwards is identical in both formats.

| Variant | Abs 156–163 | Active steps |
|---------|-------------|--------------|
| Current (fw 2.x) | `50 52 45 44 53 51 xx xx` = `PREDSQ` + bits | stored at 162–163 |
| Legacy (fw 1.xx) | `50 52 45 44 53 45 51 44` = `PREDSEQD` | implicit `0xFFFF` |

**[hw]** The logue-manager capture `dump_saved_slot2.syx` — a `0x4C` reply from internal
memory — carries `PREDSEQD`. The `0x40` edit-buffer reply for the same program carries `PREDSQ`
+ `FF FF`. So the conversion happens on load into the edit buffer; it does not rewrite
storage. Rust checks `bytes[0..6] == "PREDSQ"` and fails with `InvalidMagic`
(`sequencer.rs:174-181`). As a result, `request_program` and `parse_program_dump` cannot read
such slots. A tolerant reader should accept `PREDSEQD`, report active steps as `0xFFFF`, and
(to keep byte fidelity) write back the variant it read. A blob without `PRED…` at all is
rejected by the device with `0x26` **[hw]** (`1c16566` bug 3).

### Motion slot parameter (S2)

Two bytes per slot at abs 174 + 2s (s = 0..3).

| Byte | Bits | Field | Range |
|------|------|-------|-------|
| 0 | 0 | Motion on/off | 0,1 = Off,On |
| 0 | 1 | Smooth on/off | 0,1 = Off,On |
| 0 | 7..2 | Reserved | preserve |
| 1 | 7..0 | Parameter ID | S2-1 table below |

Read as one LE `u16` `v`: `id = v >> 8`, `on = v & 1`, `smooth = (v >> 1) & 1`. This matches
isnotinvain's `MotionParameter` **[community]**. Rust keeps the raw `u16` in
`MotionSlotConfig.parameter_id` (`sequencer.rs:13-21`) without decoding it. The field name is
misleading, because the parameter ID is the **high** byte. **[hw]** The capture has
`03 10 | 03 11 | 03 39 | 03 32`: all four slots on and smooth, with IDs 16, 17, 57 and 50.

#### S2-1 motion parameter IDs

| ID | Parameter | ID | Parameter | ID | Parameter |
|----|-----------|----|-----------|----|-----------|
| 0 | None | 31 | MULTI ENGINE VPM TYPE | 51 | EG DECAY |
| 15 | PORTAMENTO | 33 | MULTI SHAPE NOISE | 52 | EG INT |
| 16 | VOICE MODE DEPTH | 34 | MULTI SHAPE VPM | 53 | EG TARGET |
| 17 | VOICE MODE TYPE | 35 | MULTI SHAPE USER | 54 | LFO WAVE |
| 18 | VCO 1 WAVE | 36 | MULTI SHIFT SHAPE NOISE | 55 | LFO MODE |
| 19 | VCO 1 OCTAVE | 37 | MULTI SHIFT SHAPE VPM | 56 | LFO RATE |
| 20 | VCO 1 PITCH | 38 | MULTI SHIFT SHAPE USER | 57 | LFO INT |
| 21 | VCO 1 SHAPE | 39 | VCO 1 LEVEL | 58 | LFO TARGET |
| 22 | VCO 2 WAVE | 40 | VCO 2 LEVEL | 59 | MOD FX ON/OFF |
| 23 | VCO 2 OCTAVE | 41 | MULTI ENGINE LEVEL | 66 | MOD FX TIME |
| 24 | VCO 2 PITCH | 42 | CUTOFF | 67 | MOD FX DEPTH |
| 25 | VCO 2 SHAPE | 43 | RESONANCE | 68 | DELAY ON/OFF |
| 26 | SYNC | 45 | KEYTRACK | 70 | DELAY TIME |
| 27 | RING | 46 | AMP EG ATTACK | 71 | DELAY DEPTH |
| 28 | CROSS MOD DEPTH | 47 | AMP EG DECAY | 72 | REVERB ON/OFF |
| 29 | MULTI ENGINE TYPE | 48 | AMP EG SUSTAIN | 74 | REVERB TIME |
| 30 | MULTI ENGINE NOISE TYPE | 49 | AMP EG RELEASE | 75 | REVERB DEPTH |
| | | 50 | EG ATTACK | 126 | PITCH BEND |
| | | | | 129 | GATE TIME |

The unlisted IDs are 1–14, 32, 44, 60–65, 69, 73, 76–125, 127, 128 and 130–255. The gaps look
like a shared ID space (32 might be "USER TYPE", 44 "DRIVE"), but they are undocumented.
**[community]** isnotinvain notes that ID 124 appears in a factory preset. Rust has **no enum**
for these IDs. Treat unknown IDs as opaque and round-trip them.

### Step event (S3) — 52 bytes

The step base is `b = 190 + 52k` (abs) or `34 + 52k` (rel), for k = 0..15.

| Step off | Width | Field | Range [spec] |
|----------|-------|-------|--------------|
| 0–7 | 8 × u8 | Note number (1)–(8) | 0~127 |
| 8–15 | 8 × u8 | Velocity (1)–(8) | 0 = no event; 1~127 = velocity |
| 16–23 | 8 × u8 | Gate time (1)–(8), bits 6..0 | 0~72 = 0%~100%; 73~127 = TIE |
| 16–23 | bit 7 | Trigger switch (1)–(8) | 0,1 = Off,On |
| 24–30 | 7 B | Motion slot 1 data | S3-2 |
| 31–37 | 7 B | Motion slot 2 data | S3-2 |
| 38–44 | 7 B | Motion slot 3 data | S3-2 |
| 45–51 | 7 B | Motion slot 4 data | S3-2 |

- A note lane `i` is present only when `velocity[i] != 0`. The note byte of an empty lane is
  meaningless, and the device may leave stale values there. Preserve them.
- **TIE** (S3-1): a gate of 127 (the spec range says any 73~127 displays as TIE), followed by
  a next step whose trigger switch is 0 for that lane, continues the note into that step
  instead of re-triggering it.
- Rust helpers: `gate_time(n) = gate_times[n] & 0x7F`, `trigger_switch(n) = gate_times[n] &
  0x80 != 0`, `set_gate_time(n, g, t)` (`sequencer.rs:94-107`). No accessor clamps to 72 or
  checks note/velocity ≤ 127.

### Motion data (S3-2) — 5 × 10-bit points in 7 bytes

| Byte | Bits | Content |
|------|------|---------|
| 0 | 7..0 | DATA 1 bits 9..2 |
| 1 | 7..0 | DATA 2 bits 9..2 |
| 2 | 7..0 | DATA 3 bits 9..2 |
| 3 | 7..0 | DATA 4 bits 9..2 |
| 4 | 7..0 | DATA 5 bits 9..2 |
| 5 | 1..0 | DATA 1 bits 1..0 |
| 5 | 3..2 | DATA 2 bits 1..0 |
| 5 | 5..4 | DATA 3 bits 1..0 |
| 5 | 7..6 | DATA 4 bits 1..0 |
| 6 | 1..0 | DATA 5 bits 1..0 |
| 6 | 7..2 | Reserved (preserve) |

Note the different packing from the synth half. Here the **high** 8 bits are stored whole and
the low 2 bits are packed. In the synth half's 10-bit LE fields, the low 8 bits are whole and
the high 2 bits are packed.

```text
decode(m[0..7]) -> d[0..5]:            encode(d[0..5], m[0..7]):   ; m[6] bits 7..2 kept
  for i in 0..4:                         m[5] = 0
    d[i] = m[i] << 2 | (m[5] >> 2i) & 3   for i in 0..4:
  d[4] = m[4] << 2 | m[6] & 3               m[i] = d[i] >> 2
                                            m[5] |= (d[i] & 3) << 2i
                                          m[4] = d[4] >> 2
                                          m[6] = (m[6] & 0xFC) | (d[4] & 3)
```

Playback [spec]: with smooth on, the step is split into quarters, interpolating DATA 1→2,
2→3, 3→4 and 4→5. With smooth off, or for a switch-type parameter, only DATA 1 is used.
The value scale presumably follows the target parameter (0~1023 for knobs). For switch types
the capture shows small raw values; see [Test vectors](#test-vectors). Rust stores the seven
bytes raw (`motion_data: [[u8;7];4]`) and offers no decode.

### ARP and enumerations

| Abs | Rel | Field | Range [spec] | Rust |
|-----|-----|-------|--------------|------|
| 1022 | 866 | ARP gate time | 0~72 = 0%~100% | `arp_gate_time` raw |
| 1023 | 867 | ARP rate | 0~10 (S4) | `arp_rate` raw |

| Value | S4 ARP rate [spec] | Rust `ArpRate` label (`sysex/enums.rs:247-275`) |
|-------|--------------------|-------------------------------------------------|
| 0 | 64th | 1/1 |
| 1 | 48th | 3/4 |
| 2 | 32th | 2/3 |
| 3 | 24th | 1/2 |
| 4 | 16th | 3/8 |
| 5 | 16.th (dotted 16th) | 1/3 |
| 6 | 12th | 1/4 |
| 7 | 8th | 3/16 |
| 8 | 8.th (dotted 8th) | 1/6 |
| 9 | 6th | 1/8 |
| 10 | 4th | 1/12 |

The Rust `ArpRate` labels match the spec nowhere. They look like the LFO BPM-sync divisions
(P11) shifted. The enum is unused in code: it appears only in a doc link at
`sequencer.rs:144`. The Rust default `arp_rate: 9` is commented "1/8" (`sequencer.rs:333`);
per S4 it is "6th". **[community]** loguetools writes 5 with the comment "off by 1 in Korg
docs; 5 gives 16th". That suggests that S4 itself may be shifted by one on hardware; see
[Open questions](#open-questions).

`StepResolution` (`sysex/enums.rs:231-246`) matches the spec: 0 = 1/16, 1 = 1/8, 2 = 1/4,
3 = 1/2, 4 = 1/1. It is not used by `SequencerParams`, which stores a raw `u8`.

### Rust defaults (`SequencerParams::default`, `sequencer.rs:318-336`)

| Field | Value | Note |
|-------|-------|------|
| `bpm` | 1200 | 120.0 BPM (written at the wrong offset) |
| `step_length` | 16 | |
| `step_resolution` | 0 | 1/16 |
| `swing` | 75 | 0 % |
| `default_gate_time` | 54 | 75 % |
| `active_steps` | `0xFFFF` | |
| `steps_on`, `motion_on` | 0 | |
| `motion_slots[*]`, `motion_slot_steps[*]` | 0 | |
| `steps[*]` | all zero | no notes, no motion |
| `arp_gate_time` | 54 | 75 % |
| `arp_rate` | 9 | "1/8" per Rust; "6th" per S4 |

A spec-correct default for a port would be `PREDSQ FF FF B0 04 10 00 4B 36 00 00 00 00 …`
(abs 156–171) — active all, 120.0 BPM, 16 steps, 1/16, swing 0, gate 54, no steps on. Choose
the ARP rate after the S4 question is settled.

## Invariants

- `sequencer_size == 868 == 34 + 16 * 52 + 2`; `step_size == 52 == 8 + 8 + 8 + 4 * 7`.
- `blob[156..160] == "PRED"` and (`blob[160..162] == "SQ"` or `blob[160..164] == "SEQD"`).
- After a spec-correct parse: `100 <= bpm_x10 <= 3000`, `blob[165] & 0xF0` preserved,
  `1 <= step_length <= 16`, `resolution <= 4`, `swing_stored <= 150`, `default_gate <= 72`,
  `arp_gate <= 72` and `arp_rate <= 10`. Validate these **only** in a `validate()` pass, never
  in the parser, so out-of-range hardware data still round-trips.
- Each note byte and each velocity byte is `<= 127` (MIDI data). Each gate low-7 value is
  `<= 127` by construction.
- `decode(encode(d)) == d` for every `d` in `[0, 1023]^5`. `encode` leaves `m[6] & 0xFC`
  unchanged.
- `serialize(parse(blob))[156..1024] == blob[156..1024]` byte-exact.

## Errata & gotchas

| # | Item | Detail | Provenance |
|---|------|--------|------------|
| 1 | **Header rotation in Rust** | Rust rel 6–13 = bpm, len, res, swing, gate, active. The spec and the hardware have active, bpm, len, res, swing, gate | **[hw]** capture + **[spec]** + **[community]** (isnotinvain, gekart, loguetools) vs **[rust]** |
| 2 | `SEQD` legacy magic | Internal-memory `0x4C` dumps can carry `PREDSEQD`; Rust rejects them | **[hw]** capture; **[spec]** S1 |
| 3 | BPM high byte | Only bits 3..0 are BPM (`H:0~3`). Rust reads and writes a full u16 | **[spec]**; unknown bits, preserve |
| 4 | Step 16 range | Printed "426~1021"; actually 970~1021 | **[spec]** typo |
| 5 | Swing encoding | The spec gives the range −75~+75 but no encoding. The `+75` offset comes from isnotinvain and Rust. The capture value 75 is consistent with swing 0 | **[community]**, weak **[hw]** |
| 6 | ARP rate labels | Rust `ArpRate` ≠ S4. loguetools says S4 is off by one on hardware | **[spec]** vs **[rust]** vs **[community]** |
| 7 | Motion slot `u16` | The parameter ID is the high byte; bits 0/1 of the low byte are on/smooth. Rust's `parameter_id` holds both | **[spec]**, **[hw]** capture |
| 8 | Stored vs edit ARP | The same program shows ARP bytes `00 00` in storage (`0x4C`) and `37 05` in the edit buffer (`0x40`) | **[hw]** capture |
| 9 | Dump-size diagram | "336 bytes" in the dump messages is wrong; the sequencer half alone is 868 | **[spec]** erratum; see [program blob](program-blob.md#errata--gotchas) |

## Test vectors

### Rust unit tests (synthetic, Rust layout)

These vectors use the Rust (rotated) header offsets. They prove only Rust self-consistency;
do not reuse them as spec fixtures without moving the header bytes.

| Test | Setup (rel offsets) | Expect | Source |
|------|---------------------|--------|--------|
| `make_seq_blob` | `PREDSQ`; rel 6–7 = `B0 04` (1200); 8 = 16; 9 = 0; 10 = 75; 11 = 54; 12–13 = `FF FF`; 14–15 = `01 00`; 16–17 = `01 00`; 18–19 = `10 00`; 26–27 = `03 00`; rel 34 = 60, 42 = 100, 50 = 54; 866 = 36; 867 = 6 | parsed fields equal the inputs | `sequencer.rs:343-389` |
| round trip | the blob above | `to_bytes() == blob` | `sequencer.rs:392-397` |
| BPM | rel 6–7 = 100 / 3000 | `bpm_f32` = 10.0 / 300.0 | `sequencer.rs:413-427` |
| swing | rel 10 = 0 / 75 / 150 | `swing_signed` = −75 / 0 / +75 | `sequencer.rs:429-443` |
| gate packing | gate byte `72 \| 0x80` = `0xC8` | gate 72, trigger true | `sequencer.rs:492-505` |
| motion data | slot bytes `[1,2,3,4,5,6,7]` | stored raw, round-trips | `sequencer.rs:507-520` |
| empty sequencer | 868 zeros + `PREDSQ` | parses | `sequencer.rs:522-538` |
| bad magic | `XXXXXX` | `InvalidMagic` | `sequencer.rs:540-544` |
| too short | 100 bytes | `PayloadTooShort` | `sequencer.rs:400-404` |

### Real hardware capture **[hw]** (spec layout)

From irgendwatt/logue-manager @a04b2ea, `doc/analyze/dump_current_editbuf.syx` (`0x40`),
decoded:

| Abs | Bytes | Decoded |
|-----|-------|---------|
| 156–163 | `50 52 45 44 53 51 FF FF` | `PREDSQ`, all 16 steps active |
| 164–169 | `D8 04 10 01 4B 48` | BPM 1240 = 124.0; length 16; resolution 1 (1/8); swing 75 → 0; gate 72 |
| 170–173 | `FF EF FF FF` | step 13 off; motion on all steps |
| 174–181 | `03 10 03 11 03 39 03 32` | slots on+smooth: IDs 16 V.M DEPTH, 17 V.M TYPE, 57 LFO INT, 50 EG ATTACK |
| 182–189 | `FF EF FF FF FF FF FF FF` | slot 1 off on step 13; slots 2–4 on all steps |
| 190–197 | `45 32 48 41 00 00 00 00` | step 1 notes 69, 50, 72, 65 |
| 198–205 | `72 7B 6C 60 00 00 00 00` | velocities 114, 123, 108, 96; lanes 5–8 empty |
| 206–213 | `C8 C8 C8 C8 00 00 00 00` | gate 72 (100 %) + trigger on lanes 1–4 |
| 214–220 | `A0 A0 A0 A0 A0 FF 03` | slot 1 data = 643, 643, 643, 643, 643 |
| 221–227 | `00 00 00 00 00 55 01` | slot 2 data = 1, 1, 1, 1, 1 (switch-type parameter) |
| 228–234 | `84 84 8A 80 80 20 00` | slot 3 data = 528, 528, 554, 512, 512 |
| 235–241 | `0E 02 00 00 00 06 00` | slot 4 data = 58, 9, 0, 0, 0 |
| 1022–1023 | `37 05` | ARP gate 55; ARP rate 5 |

The companion `dump_saved_slot2.syx` (`0x4C`, n = 1) is identical from abs 164 on, except
for 1022–1023 = `00 00`. Its abs 156–163 is `PREDSEQD`.

## Rust design

- `SequencerParams` (`sequencer.rs:119-146`) is a plain struct: a raw `u16` for BPM and the
  bitmasks, raw `u8` for everything else, `[MotionSlotConfig; 4]`, `[u16; 4]` and
  `[StepEvent; 16]`. `StepEvent` holds `[u8;8]` × 3 and `[[u8;7];4]`. These are fixed arrays
  with no `Vec`, apart from the serialization output.
- No enum is decoded at parse time, so the only parse failures are length and magic. That is
  more tolerant than the synth half.
- `to_bytes_with_base(base)` copies `base[..868]` and then overwrites **every** byte. The
  header's 34 bytes, all 16 × 52 step bytes and both ARP bytes are rewritten
  (`sequencer.rs:253-301`). The base only matters for its length assertion. Byte fidelity for
  the sequencer half therefore comes entirely from storing every byte in a field. This works
  because no sequencer byte is left unmodelled, but it would silently fail if a field were
  ever dropped. See [byte fidelity](../design/byte-fidelity.md).
- `StepEvent::write_to(&mut Vec)` exists but `to_bytes_with_base` does not use it.
- *Essential:* fixed offsets, raw storage of the motion/slot bytes, and a legacy-magic
  decision. *Incidental:* the `Vec` outputs, `f32` BPM display, and the `assert!` on base
  length.

## Zig / TigerStyle notes

- Work on the same `[1024]u8` as the synth half, through offset accessors. Write a comptime
  table that includes `step_base = 190`, `step_size = 52`, `step_count = 16` and
  `motion_slot_size = 7`, plus `comptime assert(190 + 16 * 52 + 2 == 1024)`.
- **Use the spec offsets** (abs 162 = active bits, 164 = BPM). Keep a test that loads the
  capture above and asserts BPM 1240. That test is what catches the Rust rotation bug.
- Represent BPM as `u12` tenths (`100..3000`), not as a float. Read it as `b[164] | (b[165] &
  0x0F) << 8`. Write it with read-modify-write that preserves `b[165] & 0xF0`, like the synth
  10-bit fields.
- Model swing as `i8` in the API (`-75..75`) and store it as `u8` (`swing + 75`). Keep that
  conversion in one function, with `assert` on both sides.
- Give the step index type `u4`, the lane index `u3` and the motion slot index `u2`. With
  these exact types, bounds checks disappear, and every loop over steps, lanes or points is
  bounded by a comptime constant.
- Motion data: give `decode_motion(*const [7]u8) [5]u10` and `encode_motion([5]u10, *[7]u8)`
  pure-function tests over the complete `u10` domain for each point (1024 × 5 cases), plus the
  capture rows above.
- Return a `SeqMagic = enum { sq, seqd }` from parsing and keep it in the blob (it is already
  in the bytes). Accessors for active steps return `0xFFFF` when the variant is `.seqd`, and a
  setter on `.seqd` blobs either upgrades the header explicitly (shifting nothing — abs 164+
  are unchanged) or returns `error.LegacySequencerFormat`. Pick one and document it.
- Keep motion parameter IDs as `u8` with a comptime name table for known IDs. Do not make it
  an exhaustive `enum(u8)`, because factory data uses undocumented IDs (124).

```zig
// sketch
const seq_base = 156;
const step_base = 190;
const step_size = 52;
comptime { assert(step_base + 16 * step_size + 2 == 1024); }

fn bpm_x10(blob: *const [1024]u8) u12 {
    return @intCast(@as(u16, blob[164]) | (@as(u16, blob[165] & 0x0F) << 8));
}

fn motion_point(m: *const [7]u8, i: u3) u10 {
    assert(i < 5);
    const low: u8 = if (i < 4) (m[5] >> (i * 2)) & 3 else m[6] & 3;
    return (@as(u10, m[i]) << 2) | low;
}
```

## Open questions

- **Swing encoding:** offset +75 (community, Rust) or two's-complement `i8`? The capture value
  75 is "0 %" under the offset reading and "+75 %" under the signed one. Capture with a known
  front-panel swing.
- **ARP rate:** does the hardware follow S4 exactly or S4 + 1 (loguetools)? Capture values
  for each displayed rate.
- **Bits 7..4 of abs 165** and **bits 7..2 of each slot's byte 0 and each motion byte 6** are
  reserved. Are they always 0 on hardware? (They are 0 in the capture.)
- **Device reaction to out-of-range headers** — for example, a Rust default blob sent via
  `0x40`: is it NAKed with `0x26`, clamped, or accepted?
- **Motion value scale for switch-type parameters:** the capture's slot 2 (VOICE MODE TYPE)
  holds 1 at all five points, while the synth byte 21 holds 4. Is the switch value stored in
  its own units, or scaled?
- **Why the stored copy has ARP bytes `00 00`:** do legacy (`SEQD`) programs lack ARP fields?
  Does the device substitute global defaults on load?

## Sources

- `crates/minilogue-xd/src/sysex/program/sequencer.rs:13-21` — `MotionSlotConfig`.
- `crates/minilogue-xd/src/sysex/program/sequencer.rs:27-108` — `StepEvent`.
- `crates/minilogue-xd/src/sysex/program/sequencer.rs:114-251` — `SequencerParams`, layout comment, parse.
- `crates/minilogue-xd/src/sysex/program/sequencer.rs:253-315` — serialize and display helpers.
- `crates/minilogue-xd/src/sysex/program/sequencer.rs:318-336` — `Default`.
- `crates/minilogue-xd/src/sysex/program/sequencer.rs:338-619` — tests.
- `crates/minilogue-xd/src/sysex/enums.rs:231-275` — `StepResolution`, `ArpRate`.
- `crates/minilogue-xd/examples/read_patch.rs:143` — BPM display through the misplaced field.
- `git show 1c16566` — `PREDSQ` magic added; header offsets otherwise unchanged.
- Korg MIDI Impl rev 1.01 — TABLE 2 (offsets 156–1023); notes S1, S2, S2-1, S3, S3-1, S3-2,
  S4.
- [community] isnotinvain/minilogue-xd-util `xd_prog_bin.py` @6f8343c (sequencer fields and
  `MotionParameter`); gazzar/loguetools `xd.py` @ba0fc46 (ARP rate comment).
- **[hw]** irgendwatt/logue-manager @a04b2ea `doc/analyze/dump_current_editbuf.syx` and
  `doc/analyze/dump_saved_slot2.syx`.
- [User guide](../sources/korg-user-guide.md) — "Active Step" shift function, step edit.
