---
title: Builders — PatchBuilder and SequenceBuilder
category: design
sources:
  - crates/minilogue-xd/src/builder/mod.rs
  - crates/minilogue-xd/src/builder/patch.rs
  - crates/minilogue-xd/src/builder/sequence.rs
  - crates/minilogue-xd/src/sysex/program/synth.rs
  - crates/minilogue-xd/src/sysex/program/sequencer.rs
  - crates/minilogue-xd/tests/builder_roundtrip.rs
  - README.md
related:
  - protocol/program-blob.md
  - protocol/sequencer-blob.md
  - design/byte-fidelity.md
  - design/realtime-controller.md
  - design/stepped-params.md
  - design/testing-strategy.md
  - protocol/errata.md
  - sources/wayfinder-dsl.md
  - zig/module-layout.md
updated: 2026-11-11
---

# Builders — PatchBuilder and SequenceBuilder

> Two consuming fluent builders create programs offline.
>
> - `PatchBuilder` starts from `SynthParams::default()`. It sets about 50 of the roughly
>   120 synth fields through grouped methods that take floats in 0..1 and enums. It returns
>   `ProgramData` with a default sequencer.
> - `SequenceBuilder` sets tempo, length, resolution, swing, gates and up to eight notes
>   per step. It returns `SequencerParams`.
>
> Both builders **clamp** out-of-range input, while the realtime controller **rejects** it.
> Neither keeps a raw base. No example or hardware test uses either builder. Through the
> Rust sequencer header bug, every built program carries a malformed sequencer header. A Zig
> port should build on a comptime **template blob** and the same in-place accessors used for
> device blobs, with integer inputs and explicit errors.

## Facts

### `PatchBuilder` (`builder/patch.rs:39-224`)

- State: `PatchBuilder { synth: SynthParams }`. `new()` is `SynthParams::default()`
  (`patch.rs:43-48`). Every method takes `self` by value and returns `Self`, except `name()`,
  which returns `Result<Self>`.
- `clamp_to_10bit(f) = round(clamp(f, 0, 1) × 1023)` (`patch.rs:17-19`). Rust's
  `f32::clamp` passes NaN through, and `NaN as u16` saturates to 0, so **NaN becomes 0
  silently**.

| Method | Fields set | Conversion |
|---|---|---|
| `name(&str)` | `name` | `ProgramName::from_string`. Pads with spaces, truncates to 12 and errors on a byte outside the P1 charset |
| `vco1(wave, octave, pitch, shape)` / `vco2(...)` | wave, octave, pitch, shape | enums; floats → 10-bit |
| `vco1_level(f)` / `vco2_level(f)` | level | 10-bit |
| `sync_ring(Sync, Ring)` | `sync`, `ring` | `== On` → bool (the blob inverts them later) |
| `cross_mod_depth(f)` | `cross_mod_depth` | 10-bit |
| `multi(MultiType, level)` | `multi_type`, `multi_level` | |
| `filter(cutoff, reso, CutoffDrive, CutoffKeytrack)` | 4 fields | |
| `amp_eg(a, d, s, r)` / `eg(a, d, int, EgTarget)` | 4 fields each | |
| `lfo(LfoWave, LfoMode, rate, int, LfoTarget)` | 5 fields | |
| `mod_fx(on, ModFxType)` | `mod_fx_on`, `mod_fx_type` | **stores `to_program_value() + 1`** (1-based raw, `patch.rs:162-167`) |
| `mod_fx_params(time, depth)` | 2 fields | |
| `delay(on, DelaySubType, time, depth, dry_wet)` / `reverb(...)` | 5 fields each | |
| `portamento(u8)` | `portamento` | `min(127)` |
| `build()` | — | `ProgramData::new(synth, SequencerParams::default())` (`patch.rs:220-223`) |
| `build_with_sequencer(seq)` | — | `ProgramData::new(synth, seq)` |

**Not reachable through the builder:** octave, key trig, voice mode type and depth,
multi-engine select/shape/shift-shape, mod FX sub-type, bend ranges, joystick and CV
assigns, micro tuning, scale key, program tuning, LFO sync flags, velocity sensitivity,
multi routing and octave, EG legato, portamento mode, program level, and VPM/user params.
These require writing `SynthParams` fields directly (the fields are `pub`).

**Defaults that end up in every built patch** (`synth.rs:707-815`):

- VCO1 Saw 8' at level 1023; VCO2 Saw 8' at level 0; pitch 512.
- Cutoff 1023, resonance 0. Amp EG 0/1023/1023/0. EG decay 1023, int 512, target Cutoff.
- LFO Tri Normal, rate 512, int 0, target Cutoff.
- All FX off. Mod FX type raw 1; delay Stereo; reverb Hall.
- Bend ±2; joystick/CV ranges 100; scale key 12 (centre); program tuning 50 (centre).
- Program level 102 (0 dB); octave 2 (centre).
- `voice_mode_type = 3`, read three different ways:
  - the Rust comment says "Poly (1-based)";
  - Rust's own `voice_mode_type_enum()` decodes raw 3 as **Chord** (raw − 1 = prog 2 in
    `Poly 0, Unison 1, Chord 2, Arp 3, ArpLatch 4`; `param/enums.rs:279-283`);
  - the community mapping reads 3 as **UNISON**
    ([program blob](../protocol/program-blob.md#errata--gotchas) #7).

### `SequenceBuilder` (`builder/sequence.rs:31-140`)

- State: `SequenceBuilder { seq: SequencerParams }`, starting from
  `SequencerParams::default()`. Defaults: 120.0 BPM (1200), 16 steps, 1/16, swing raw 75
  (0), gate 54 (75 %), `active_steps = 0xFFFF`, ARP gate 54, ARP rate 9
  (`sequencer.rs:318-336`).

| Method | Effect | Clamp |
|---|---|---|
| `bpm(f32)` | `bpm = round(clamp(b, 10, 300) × 10)` | 10.0–300.0 |
| `length(u8)` | `step_length` | 1–16 |
| `resolution(u8)` | `step_resolution` (0 = 1/16 … 4 = 1/1) | `min(4)` |
| `swing(i8)` | `swing = clamp(s, -75, 75) + 75` | ±75 |
| `default_gate_time(u8)` | `default_gate_time` | `min(72)` |
| `step(i, note, vel)` | lane 0: note, velocity, `gate = default_gate_time \| 0x80` (trigger on). Sets bit `i` in `steps_on` and `active_steps` | `i ≥ 16` silently ignored; note and velocity unchecked |
| `step_poly(i, &[(note, vel)])` | lanes 0..n (`n ≤ 8`), same gate rule | as `step` |
| `arp_gate_time(u8)` / `arp_rate(u8)` | ARP bytes | `min(72)` / `min(10)` |
| `build()` | returns `SequencerParams` | — |

No method covers motion sequences, motion slots, step on/off without a note, ties, or
per-lane gate.

## Invariants

What the Rust builders guarantee:

1. `build()` always succeeds, and its output always serializes to exactly 1024 bytes.
2. Every 10-bit field set through the builder is in 0..1023, and every clamped integer is
   within its stated range.
3. A builder output round-trips through `to_bytes` / `from_bytes` with `==` equality
   (`tests/builder_roundtrip.rs:12-93`). Because `PartialEq` ignores the raw blob, that is
   typed-field equality ([byte fidelity](byte-fidelity.md)).

What they **do not** guarantee: that the device accepts the blob, because the header bug
below breaks it.

## Errata & gotchas

| # | Item | Detail | Provenance |
|---|---|---|---|
| 1 | **Malformed sequencer header in every built program** | The default `SequencerParams` serializes abs 162–169 as `B0 04 10 00 4B 36 FF FF`. Under the spec layout that means active steps `0x04B0`, BPM 1.6, length 75, resolution 54, swing 255 and gate 255. `SequenceBuilder::bpm/length/...` write to the wrong bytes the same way ([sequencer blob](../protocol/sequencer-blob.md#header--what-rust-actually-does-bug)). | [hw] + [spec] vs [rust] |
| 2 | Never tested on hardware | No example uses `PatchBuilder` or `SequenceBuilder` (checked by grep over `crates/`); the hardware fix in `1c16566` used device-read blobs. `quintal.rs` builds sounds over CC and saves on the device. | [rust] |
| 3 | Clamp vs reject | Builders clamp (and NaN → 0). `RealtimeController` returns `OutOfRange`. The same float means different things in the two APIs at the edges. | [rust] |
| 4 | `step_poly` doc vs code | The doc says "Empty (0) notes are skipped", but the code writes every pair, including `(0, 0)` (`sequence.rs:107-121`). Lane presence is defined by `velocity != 0` ([sequencer blob](../protocol/sequencer-blob.md#step-event-s3--52-bytes)). | [rust] |
| 5 | Stale lanes | `step()` writes lane 0 only and never clears lanes 1..7. `step_poly` with fewer than 8 pairs leaves the higher lanes as they were. That is harmless from defaults (all 0), but wrong if the builder ever starts from a device blob. | [rust] |
| 6 | Order-dependent gate | `step()` copies `default_gate_time` at call time. Calling `default_gate_time(g)` after `step()` does not update existing steps. | [rust] |
| 7 | Silent index drop | `step(16, ...)` does nothing and reports no error (`sequence.rs:95-99`). | [rust] |
| 8 | `active_steps` redundancy | It defaults to `0xFFFF`, so `step()` setting the bit is a no-op. A shorter `length()` doesn't clear the bits. The device semantics of "active" vs "on" are on [sequencer blob](../protocol/sequencer-blob.md). | [rust] |
| 9 | Mod FX type raw | `mod_fx(.., Chorus)` stores 1. Hardware evidence suggests raw 1 may be Ensemble ([CC map](../protocol/cc-map.md#errata--gotchas), [program blob](../protocol/program-blob.md#errata--gotchas) #17). | disputed |
| 10 | ARP rate labels | `arp_rate(9)` is the default, commented "1/8". Per S4 it is "6th", and the community reports S4 itself is off by one. | [spec] vs [rust] vs [community] |
| 11 | README drift | The README and `CLAUDE.md` describe `SequenceBuilder` "with motion sequences". The code has none. | [rust] |

## Test vectors

| Call | Result | Source |
|---|---|---|
| `SequenceBuilder::new().bpm(130.0).length(8).build()` | `bpm == 1300`, `step_length == 8` | `sequence.rs:18-29` (doc test) |
| `.bpm(5.0)` / `.bpm(500.0)` | 100 / 3000 | `sequence.rs:178-187` |
| `.swing(50)` / `.swing(-30)` / `.swing(100)` / `.swing(-100)` | raw 125 / 45 / 150 / 0 | `sequence.rs:262-279` |
| `.step(0, 60, 100)` with the default gate | `notes[0] = 60`, `velocities[0] = 100`, `gate_times[0] = 54 \| 0x80 = 0xB6`, `steps_on & 1` | `sequence.rs:228-233` |
| `PatchBuilder.vco1(.., 1.5, -0.5)` | pitch 1023, shape 0 | `tests/builder_roundtrip.rs:106-117` |
| `PatchBuilder::new().build().to_bytes()` | 1024 bytes; abs 162–169 = `B0 04 10 00 4B 36 FF FF` (**invalid**) | [sequencer blob](../protocol/sequencer-blob.md) |

For the Zig port, replace the last row with the spec-layout template. With 120.0 BPM, 16
steps, 1/16, swing 0 and gate 54, abs 162–169 should be `FF FF B0 04 10 00 4B 36`: active
steps `0xFFFF`, BPM 1200 LE, length 16, resolution 0, swing 75, gate 54.

## Rust design

- **Consuming builder over a public struct.** The builder adds grouping (`filter(...)`,
  `amp_eg(...)`) and clamping. It adds no validation and no invariants beyond those of
  `SynthParams`, whose fields are all `pub` anyway.
- **Floats as the musical interface.** Floats work for sliders and recipes in
  `assets/ai/SOUND_DESIGN.md`. But the controller and the builder treat the edges
  differently.
- **Fresh from defaults, not from a device template.** No raw base means zero-filled
  unmodelled bits. That is probably acceptable ([byte fidelity](byte-fidelity.md) open
  question), but it combines with the header bug.

## Zig / TigerStyle notes

- **Builders are just setters on a template.** Make `Program.init()` copy a comptime
  `program_template: [1024]u8` with valid magic numbers, a spec-order header, NUL-padded
  name and documented defaults. "Building" is then calling the in-place setters from
  [byte fidelity](byte-fidelity.md). A device blob and a fresh blob take the same path, so
  editing a device patch preserves its bits.
- **Integers in the core, floats at the edge.** Setters take `u10`, `u7` and enums. If a
  float convenience layer exists, it must use **the same policy as the controller**. Prefer
  rejecting with `error.OutOfRange`; clamping hides bugs, which goes against TigerStyle's
  "handle all errors".
- **Explicit step errors.** `fn set_step(p, index: u4, lane: u3, note: u7, vel: u7, gate: Gate)`.
  `u4` makes `index ≥ 16` unrepresentable. `Gate = union(enum) { percent72: u7, tie }`, with
  an assert that `percent72 ≤ 72`. Clearing a lane is an explicit `clear_lane`.
- **Motion sequences as a first-class API**, with explicit over-limit errors (the
  [wayfinder DSL](../sources/wayfinder-dsl.md) needs this): 4 slots × 16 steps × 5 points,
  10-bit each (S3-2). Slot parameter IDs come from the S2-1 table.
- **Comptime-checked template.** Use `comptime { assert(std.mem.eql(u8, template[0..4], "PROG")); ... }`,
  and add a unit test that parses the template with the strict validator and gets zero
  findings.
- **Name setter.** Validate the P1 charset, return `error.NameTooLong` instead of
  truncating, and pad with `0x00` to match the hardware.

## Open questions

- Does the device accept a blob whose unmodelled bits are all zero? The answer decides
  whether the template must be captured from hardware (an INIT program dump) or can be
  synthesized.
- What is the correct default for `voice_mode_type`, and which raw value means POLY on
  hardware?

## Sources

- `crates/minilogue-xd/src/builder/patch.rs:1-238`, tests `:241-476`.
- `crates/minilogue-xd/src/builder/sequence.rs:1-140`, tests `:149-355`.
- `crates/minilogue-xd/src/sysex/program/synth.rs:707-815` (defaults).
- `crates/minilogue-xd/src/sysex/program/sequencer.rs:318-336` (defaults).
- `crates/minilogue-xd/tests/builder_roundtrip.rs`.
- Korg MIDI Implementation rev 1.01 TABLE 2, notes S1–S4, P1.
