---
title: Examples
category: design
sources:
  - crates/minilogue-xd/examples/list_ports.rs
  - crates/minilogue-xd/examples/test_drive.rs
  - crates/minilogue-xd/examples/berlin_school.rs
  - crates/minilogue-xd/examples/export_midi.rs
  - crates/minilogue-xd/examples/exit.rs
  - crates/minilogue-xd/examples/quintal.rs
  - crates/minilogue-xd/examples/read_patch.rs
  - crates/minilogue-xd/examples/read_edit_buffer.rs
  - crates/minilogue-xd/examples/sysex_roundtrip_diag.rs
  - crates/minilogue-xd/examples/berlin_school-legend.txt
  - crates/minilogue-xd/examples/berlin_school.mid
  - crates/minilogue-xd/Cargo.toml
  - crates/minilogue-xd/src/device.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/src/lib.rs
  - README.md
related:
  - design/testing-strategy.md
  - design/realtime-controller.md
  - design/sysex-transactions.md
  - design/byte-fidelity.md
  - design/midi-file-export.md
  - protocol/usb-ports.md
  - protocol/cc-map.md
  - protocol/program-blob.md
  - protocol/standard-midi-file.md
  - protocol/errata.md
  - zig/port-overview.md
  - zig/dependency-elimination.md
updated: 2026-10-11
---

# Examples

> The crate ships nine runnable examples in `crates/minilogue-xd/examples/`. Six talk to the
> XD in real time over USB MIDI:
>
> - **Port discovery:** `list_ports`.
> - **Live performance:** `test_drive`, `berlin_school`, `exit`, `quintal`.
> - **SysEx readers and diagnostics:** `read_patch`, `read_edit_buffer`,
>   `sysex_roundtrip_diag`.
>
> One, `export_midi`, writes a Standard MIDI File with no hardware. The examples are the
> crate's only hardware tests. Four of them use the `music_comp_mt` music-theory crate, and
> only to turn note names and chord symbols into MIDI note numbers. A Zig port can replace it
> with about 40 lines of tables.

## Facts

### Summary

All examples except `export_midi` declare `required-features = ["midi-io"]`
(`crates/minilogue-xd/Cargo.toml:29-62`). `export_midi` deliberately has no
`required-features`, so it builds under `--no-default-features`
(`docs/design/05-active/0002-midi-file-export-implementation-plan.md:97-101`).

| Example | Lines | Hardware | Ports | Main APIs | CLI | Acceptance demo for Zig |
|---------|------:|----------|-------|-----------|-----|-------------------------|
| `list_ports` | 24 | enumerate only | outputs | `device::list_output_ports`, `device::find_output_port` | none | **yes, M1** (port discovery) |
| `test_drive` | 122 | send only | SOUND | `RealtimeController` setters, `play_note`/`stop_note`, theory `Chord` | none | yes (live control smoke test) |
| `berlin_school` | 373 | send only | SOUND | `RealtimeController`, host-timed 16th-note sequence | none | yes (timing under load) |
| `export_midi` | 117 | **none** | — | `MidiFileBuilder`, `SynthParams` literal | none | **yes** (golden `berlin_school.mid`) |
| `exit` | 353 | send only | SOUND | `RealtimeController`, patch reconfiguration, theory `Chord` | none | optional |
| `quintal` | 763 | send; SysEx for `--save` | SOUND, plus KBD/KNOB for `--save` | Event score, `RealtimeController`, `MidiFileBuilder`, `SysexTransaction` read-modify-write | `--export`, `--save [N]` | yes (`--save` = edit-buffer write; `--export`) |
| `read_patch` | 150 | request/reply | SOUND → KBD/KNOB | `SysexTransaction::request_program`, `ProgramData` fields | `[N]` (0-based, default 0) | **yes** |
| `read_edit_buffer` | 199 | request/reply | SOUND → KBD/KNOB | `request_current_program`; emits Rust source | none | **yes, M6** |
| `sysex_roundtrip_diag` | 188 | request/reply | SOUND → KBD/KNOB | raw request, manual receive, `parse_sysex`, `ProgramData` byte diff | `--compare A B` | **yes** (byte-perfect proof) |

The README's examples list (`README.md:238-253`) omits `quintal`, `read_edit_buffer`, and
`sysex_roundtrip_diag`.

### Common plumbing

**Port discovery.** See [USB ports](../protocol/usb-ports.md). The XD exposes four USB MIDI
ports (`crates/minilogue-xd/src/device.rs:1-18`):

| Port | Direction | Use |
|------|-----------|-----|
| SOUND | host → XD | Drives the synth engine. Send all notes, CCs, and SysEx here |
| MIDI OUT | host → XD | Forwarded to the physical DIN out jack |
| KBD/KNOB | XD → host | Panel events and SysEx replies. Listen here |
| MIDI IN | XD → host | Forwarded from the physical DIN in jack |

Port selection works like this:

1. `find_output(prefer)` takes the first port whose name *contains* the preferred substring:
   `"minilogue xd SOUND"` or `"minilogue xd MIDI OUT"`.
2. If none matches, it **falls back to the other XD output** (`device.rs:183-204`).
   `find_input` does the same with `"minilogue xd KBD/KNOB"` and `"minilogue xd MIDI IN"`
   (`device.rs:221-242`).
3. `find_output_port()` is `find_output(Sound)` and `find_input_port()` is
   `find_input(KbdKnob)` (`device.rs:248-258`).

The performance examples call `find_output_port()` and `MidirOutput::connect`. The SysEx
examples call `find_output(OutputPort::Sound)`, `find_input(InputPort::KbdKnob)`, and both
`connect`s. Every example uses MIDI channel 1 (`U4::new(0)`).

**Live control bytes.** Defined by [realtime controller](realtime-controller.md) and the
[CC map](../protocol/cc-map.md).

| Call | Wire bytes (channel 1) | Source |
|------|------------------------|--------|
| `play_note(k, v)` | `90 kk vv` | `controller.rs:438-441` |
| `stop_note(k)` | `80 kk 00` | `controller.rs:444-447` |
| `all_notes_off()` | `B0 7B 00` (CC123) | `controller.rs:472-474` |
| 10-bit setter, e.g. `set_cutoff(0.35)` | `v = round(0.35 × 1023) = 358`; `B0 3F 06` (CC63 = v & 7) then `B0 2B 2C` (CC43 = v >> 3) | `controller.rs:29-39`, `:123-127` |
| Stepped setter, e.g. `set_mod_fx_type(Ensemble)` | `B0 cc tx` with the enum's TX value (Ensemble = 38) | `param/enums.rs:267-274` |
| On/off setter | `B0 cc 00` or `B0 cc 7F` | `controller.rs` |

`f32_to_10bit` returns an error, not a clamp, outside `0.0..=1.0`. The examples clamp before
calling (`controller.rs:29-39`).

**SysEx transaction.** Defined by [SysEx transactions](sysex-transactions.md) and
[SysEx framing](../protocol/sysex-framing.md).

- Every SysEx example builds `SysexTransaction::new(&mut out, &mut in, ch0)` with a 5 s
  timeout.
- Requests go out on SOUND:
  - current program: `F0 42 30 00 01 51 10 F7`;
  - stored program: `F0 42 30 00 01 51 1C nl nm F7`, where `nl = n & 0x7F` and
    `nm = n >> 7` (`sysex/program/mod.rs:225-241`).
- Replies arrive on KBD/KNOB as function `0x40` (current) or `0x4C` (stored). The reply
  carries 1,024 data bytes, 7-bit packed to 1,171.
- Before the reply, the XD typically sends `F8` timing clock bytes. The receiver discards
  them (commit `bc6c452`) **[hw]**.

### Per-example detail

#### `list_ports`

1. Prints every output port name (`device::list_output_ports`).
2. Prints the result of `find_output_port()` (`examples/list_ports.rs:5-24`).

It sends no MIDI and does not list input ports.

#### `test_drive`

- **Patch setup** (`examples/test_drive.rs:36-81`): it connects to SOUND and sets a pad patch
  through about 40 `set_*` calls. Each 10-bit knob sends two CCs.
- **Chords** (`:89-94`): `Chord::from_regex` builds Cm9, Fm7, Gm7, and Cm. Each chord's notes
  come from `chord.notes().iter().map(midi_pitch)` (`:100`).
- **Playback per chord:**
  1. Note-ons, 20 ms apart (strum).
  2. Hold for 1,500 ms.
  3. Note-offs.
  4. A 300 ms gap.
- **Ending:** a 3 s tail (`:103-118`).

Cm9 expands to five notes (60 63 67 70 74), more than the XD's four voices, so one note is
stolen or dropped.

#### `berlin_school`

- **Patch setup** (`examples/berlin_school.rs:69-121`): `RealtimeController` on SOUND.
- **Timing** (`:130-131`): a 120 BPM 16th-note grid, so `step = 60 / bpm / 4` = 125 ms.
- **Pattern:** steps of `(note, velocity, gate fraction)` or rest.
- **Passes:** each pass transposes the pattern and repeats it. It sends cutoff at the start of
  each repetition (`:338-343`) and scales velocity (`:348`).
- **Each step:** `play_note`, sleep `gate × step`, `stop_note`, then sleep the rest of the step
  (`:349-360`).
- **Ending:** `all_notes_off` (`:367`), then a 4 s tail.

Timing is host-owned `thread::sleep`, so drift accumulates. See
[wayfinder DSL](../sources/wayfinder-dsl.md) for the planned scheduler contract.

#### `export_midi`

- **No hardware.** A `SynthParams` struct literal (`examples/export_midi.rs:22-39`) describes
  the patch, with raw values such as `vco2_pitch: 527`, `cutoff: 358`, `resonance: 460`.
- `MidiFileBuilder::new(120.0).track_name(…).patch_ccs(0, &synth)` (`:41-43`) puts every patch
  CC at tick 0.
- A 16-step E-minor pattern (`:48-65`) is played in passes of
  `(transpose, repeats, base_cutoff, drift, vel_scale)` (`:68-83`). One pass is transposed −4
  semitones, to C minor.
- Each repetition starts with `set_cutoff(tick, …)` (`:91`). Each step adds
  `note(tick, n, vel, step × gate)` (`:98`). One step is 120 ticks at 480 ticks per quarter.
- **Outputs:**
  - `berlin_school-legend.txt` (`:105-106`): 2 header lines and 50 `CC nnn  Name` lines,
    produced by `MidiFileBuilder::legend` (`src/midi_file.rs:846-855`).
  - `berlin_school.mid` (`:109-110`).

  Both files are committed. The `.mid` file is 4,747 bytes, format 0, one track, 480 tpq.
  Its first events are tempo (`FF 51 03 07 A1 20` = 500,000 µs per quarter), time signature
  (`FF 58 04 04 02 18 08`), and track name (`FF 03`), in that order. See
  [standard MIDI file](../protocol/standard-midi-file.md) and
  [MIDI file export](midi-file-export.md).

#### `exit`

There is one channel and one patch, reconfigured three times: `setup_voice_a` (`:29`),
`setup_voice_b` (`:90`), and `setup_hybrid` (`:134`). The "voices" are sequential patch
changes made with CCs, **not** multitimbral parts. Tempo is 130 BPM (`:205-206`).

| Act | What happens |
|-----|--------------|
| I | A sequence played four times, with cutoff changes |
| II | Chords Dm, Bb, Gm, Dm from `Chord::from_regex` (`:232-237`). Notes are struck 30 ms apart and held 3 s (`:266-279`) |
| III | The hybrid patch. Each chord is played as a pad an octave up (`midi_pitch() + 12`, `:297-301`) under two passes of the sequence |
| Ending | A fade over four passes, `all_notes_off`, and a 4 s tail (`:347-349`) |

#### `quintal`

This example is a small **score model**, and the closest precedent for the planned DSL
adapter.

- **Events** (`examples/quintal.rs:48-53`): `NoteOn`, `NoteOff`, `Cutoff`, and `Label`, each
  with an absolute time in milliseconds.
- **Ordering** (`:65-70`): events at the same time are ordered by priority:
  `Label` 0, `Cutoff` 1, `NoteOff` 2, `NoteOn` 3. So a retriggered note is released before it
  is re-struck. The sort is stable on `(time, priority)` (`:380-385`).
- `build_score(bpm)` (`:145`) runs at 60 BPM.

| Mode | Flow |
|------|------|
| default (play) | Runs `setup_patch` with live CCs (`:395-461`). `play` (`:473-516`) walks the sorted event list, sleeping `event_time - now` on a *logical* millisecond counter (`:488-494`). It never reads a clock, so send time and sleep overshoot still accumulate. Ends with `all_notes_off` and a 5 s tail |
| `--export` | `export` (`:523-583`) converts ms to ticks (`round(ms / ms_per_tick)`), pairs each note-on with its note-off, and closes overlapping retriggers. Writes `quintal.mid` and `quintal-legend.txt` (`:572-575`). No hardware is used, but the example still needs `midi-io` to build |
| `--save [N]` | `save` (`:590-637`): opens SOUND and KBD/KNOB; `request_current_program()` returns the base blob; `apply_patch_params` (`:642-732`) overwrites synth fields. With `N`, it calls `send_program(ProgramNumber::new(N))` and prints `N + 1` as the display number. Without `N`, it calls `send_current_program` (edit buffer only) |

The read-modify-write in `--save` relies on `ProgramData` keeping the raw blob, so that unknown
bytes survive. See [byte fidelity](byte-fidelity.md).

#### `read_patch`

1. Argument 1 is a **0-based** program number, defaulting to 0 (`examples/read_patch.rs:17-22`).
2. Opens SOUND and KBD/KNOB and calls `request_program` (`:43`).
3. Prints oscillators, multi engine, mixer, filter, EGs, LFO, effects, and sequencer. BPM is
   shown as `bpm / 10` (`:143`).

The header prints the 0-based number as `Program {:03}`, so the panel's program 001 prints as
`000`.

#### `read_edit_buffer`

1. Calls `request_current_program()` (`examples/read_edit_buffer.rs:183`).
2. Writes Rust source to stdout in two forms:
   - an `apply_patch_params` body with raw blob values (`:28-84`);
   - a `setup_patch` body of float `RealtimeController` calls, using `raw / 1023` to three
     decimals (`:23-25`, `:86-164`).

This is the tool used to capture a panel-made sound into code.

#### `sysex_roundtrip_diag`

This is the byte-fidelity oracle. See [byte fidelity](byte-fidelity.md).

Default mode (`examples/sysex_roundtrip_diag.rs:91-185`):

1. Sends `build_current_program_request` on SOUND.
2. Receives by hand until a chunk ends in `F7`, within a 5 s deadline. It skips only
   *single-byte* chunks ≥ `F8`, whereas `receive_sysex` filters every byte above `F7`.
3. Runs `parse_sysex` and asserts function `0x40`.
4. Runs `ProgramData::from_bytes`, then `to_bytes`.
5. Diffs all 1,024 bytes and labels each difference as `SynthParams` (offset < 156) or
   `SequencerParams`. Prints "PERFECT ROUND-TRIP" when there is no difference.
6. Also compares the full rebuilt SysEx (`build_current_program_dump`) with the received
   message.

`--compare A B` (`:82-88`, `:18-64`) instead requests two stored programs and diffs them.
That shows which bytes a single panel edit changes.

## Invariants

- Live examples must end with `all_notes_off` (`B0 7B 00`) so no voice is left hanging. All
  four performance examples do.
- SysEx requests go on SOUND and replies are read on KBD/KNOB. Sending SysEx on MIDI OUT
  reaches the DIN jack, not the synth.
- Read-modify-write (`quintal --save`) must start from a blob read from the device, never from
  a default blob.
- `sysex_roundtrip_diag` passes only if decode → encode is the identity on all 1,024 bytes and
  on the full 1,179-byte current-program SysEx message: 7 header bytes + 1,171 data + `F7`.

## Errata & gotchas

- **The SOUND fallback can pick the wrong port.** `find_output(Sound)` falls back to MIDI OUT
  when SOUND is missing (`device.rs:192-203`). Notes then go to the DIN jack and the synth
  stays silent. A port should fail rather than fall back for engine control.
- **Too many notes.** Cm9 in `test_drive` has five notes, and the XD has four voices.
- **Different note-off encodings.** Live note-off is `8n kk 00` (`controller.rs:444-447`).
  The SMF writer uses `8n kk 40` (`src/midi_file.rs:138`).
- **Off-by-one display.** `read_patch` prints the 0-based number. `quintal --save N` writes
  0-based slot `N` and prints `N + 1`.
- **`quintal --export` needs `midi-io`** because the whole example has
  `required-features = ["midi-io"]`.
- **Voice-mode and mod-FX numbering conflicts** in `quintal.rs:650` and `:701`.

  For voice mode, there are four claims:

  | Claim | Mapping | Source |
  |-------|---------|--------|
  | Spec, NRPN | 0 ARP LATCH, 1 ARP, 2 CHORD, 3 UNISON, 4 POLY | Korg MIDI Impl rev 1.01 *3-5 |
  | Spec, TABLE 2 byte 21 | range 1~4; values "1 ARP, 2 CHORD, 2 UNISON, 3 POLY", with a duplicated 2 | *note P3 |
  | Code | stores the raw byte; converts with `raw − 1` against enum program values Poly 0, Unison 1, Chord 2, Arp 3, ArpLatch 4. Raw 3 therefore decodes as Chord | `param/enums.rs:278-285`, `sysex/program/synth.rs:656-670` |
  | Code comments | say 1 = Arp, 2 = Chord, 3 = Poly, 4 = ArpLatch | `sysex/program/synth.rs:658`, `:715` |

  `quintal` writes raw 4, commented "ArpLatch (as saved by hardware)" **[hw?]**.

  For mod-FX type:

  - The spec says 1 CHORUS, 2 ENSEMBLE, 3 PHASER, 4 FLANGER, 5 USER (*note P12).
  - The code's `raw − 1` agrees: raw 1 = Chorus (`sysex/program/synth.rs:1001`).
  - `quintal` writes raw 1, commented "Ensemble (as saved by hardware)", after selecting
    Ensemble live with CC (`quintal.rs:441`).

  The quintal comments are the only hardware claims, and they are unverified. See
  [program blob](../protocol/program-blob.md) and [errata](../protocol/errata.md).
- **The README is incomplete.** Three examples are missing from its list (`README.md:238-253`).

## Test vectors

| Vector | Value | Source |
|--------|-------|--------|
| `set_cutoff(0.35)` | `B0 3F 06 B0 2B 2C` | `controller.rs:29-39`, `:123-127` |
| Current-program request, ch 1 | `F0 42 30 00 01 51 10 F7` | `sysex/program/mod.rs:192-194` |
| Program 499 request, ch 1 | `F0 42 30 00 01 51 1C 73 03 F7` (499 = `0x1F3`) | `sysex/program/mod.rs:225-241` |
| `berlin_school.mid` header chunk | `4D 54 68 64 00 00 00 06 00 00 00 01 01 E0` | `examples/berlin_school.mid` |
| `berlin_school.mid` track header | `4D 54 72 6B 00 00 12 75` (track length 4,725) | same |
| Legend line format | `CC  43  Cutoff` (`"CC {:>3}  {}"`) | `examples/berlin_school-legend.txt`, `src/midi_file.rs:846-855` |
| Note-name helper | `n(C, 4) = 60`, `n(A, 4) = 69`, `n(E, 2) = 40` | `music_comp_mt` 0.5 `midi_pitch` |

## Rust design

- The examples use the **public API only**, and `export_midi` uses only the pure core. They
  double as API documentation, and they gate which APIs must exist.
- Each example is a single `main`. There is no shared harness, and port discovery is copied
  into each file.
- Two timing styles appear, and both accumulate drift:
  - sleep per step, using gate arithmetic (`berlin_school`, `exit`, `test_drive`);
  - an event list sorted by `(time, priority)`, with sleeps between consecutive logical times
    (`quintal::play`, `:484-509`).

  The quintal *event model* is the precedent for the Zig acceptance demos. Its timing loop is
  not.

### Music-theory dependency

`music_comp_mt` 0.5 (feature `midi`) is a **non-optional** dependency
(`crates/minilogue-xd/Cargo.toml`). It is re-exported as `minilogue_xd::theory`
(`src/lib.rs:93-94`), was added in commit `9ea5ecf`, and is used **only by examples**. The
library itself never calls it.

| Use | Examples | API used | What it computes |
|-----|----------|----------|------------------|
| Note name → MIDI number | `berlin_school.rs:25-27`, `export_midi.rs:13-15`, `exit.rs:24-26`, `quintal.rs:39-41` | `Note::new(Pitch::from(PitchSymbol::X), octave).midi_pitch()` | `min(127, 12 × (octave + 1) + semitone)`, so C4 = 60 |
| Chord symbol → notes | `test_drive.rs:89-100`, `exit.rs:232-237`, `:266`, `:297-301` | `Chord::from_regex("Cm9")`, `.notes()`, `.midi_pitch()` | Root at octave 4, intervals stacked upward |
| Chord display | `test_drive.rs:97` | `.format_notes()` | Note names for printing |

Pitch symbols used: C, Cs, D, Eb, E, F, G, Ab, A, Bb, B. Octaves used: 1–5. The crate brings in
`regex`, `strum`, and `strum_macros`.

| Chord | MIDI notes |
|-------|-----------|
| Cm9 | 60 63 67 70 74 |
| Fm7 | 65 68 72 75 |
| Gm7 | 67 70 74 77 |
| Cm | 60 63 67 |
| Dm | 62 65 69 |
| Bb | 70 74 77 |
| Gm | 67 70 74 |

## Zig / TigerStyle notes

**Acceptance demos, in porting order.** These line up with the milestones on
[port overview](../zig/port-overview.md).

| Order | Demo | Proves | Pass criterion |
|------:|------|--------|----------------|
| 1 | `list_ports` | platform MIDI enumeration ([platform MIDI](../zig/platform-midi.md)) | Lists the four XD ports; picks SOUND, and fails if SOUND is absent |
| 2 | `read_edit_buffer` / `read_patch` | request, reassembly, codec, blob decode | Same printout as the Rust tool for the same device state |
| 3 | `sysex_roundtrip_diag` | byte fidelity | "PERFECT ROUND-TRIP" plus full-message match on the edit buffer and on several factory programs |
| 4 | `export_midi` | SMF writer, CC encoding | Output byte-identical to the committed `berlin_school.mid` (after confirming that file matches current Rust output) and to the legend text |
| 5 | `test_drive` / `berlin_school` | live CC/note encoding, timing | Audible result; captured byte log matches the Rust log |
| 6 | `quintal --export` / `--save` | score ordering, SMF, read-modify-write | `.mid` byte-identical to the Rust output; `--save` round trip read back unchanged |

**Zero-dependency music theory.** This is all the examples need. Sketch:

```zig
// sketch
const PitchClass = enum(u4) { c, cs, d, eb, e, f, fs, g, ab, a, bb, b };

fn midi_note(pitch_class: PitchClass, octave: u4) u7 {
    const value: u16 = 12 * (@as(u16, octave) + 1) + @intFromEnum(pitch_class);
    std.debug.assert(value <= 127);
    return @intCast(value);
}

const chord_major = [_]u8{ 0, 4, 7 };
const chord_minor = [_]u8{ 0, 3, 7 };
const chord_minor_7 = [_]u8{ 0, 3, 7, 10 };
const chord_minor_9 = [_]u8{ 0, 3, 7, 10, 14 };
```

- **Assert, don't saturate.** Assert `≤ 127` at the boundary instead of copying
  `music_comp_mt`'s silent `min(127, …)`.
- **No chord parser.** Chord symbols are fixed in the demos. Write chords as
  `(root, octave 4, interval table)` constants. A regex-free parser is unnecessary.
- **Static names.** Note names for printing come from a static `[12][]const u8` table.

**Timing.** Use quintal's sorted event list, but sleep to absolute deadlines measured on a
monotonic clock (`deadline = start + event_time`). Per-step or delta sleeps let drift build up.
Each demo should accept a `--dry-run` that writes bytes to a fixed-capacity log instead of a
port. That makes every demo also a deterministic test: compare its log with a golden log. See
[testing strategy](testing-strategy.md).

**Ports.** Do not copy the SOUND → MIDI OUT fallback. Return an explicit error that names the
missing port.

## Open questions

- Port-name substrings were observed on macOS. Do Windows (WinMM/WinRT) and Linux (ALSA)
  expose the same `"minilogue xd SOUND"` substrings? See [USB ports](../protocol/usb-ports.md).
- Which voice-mode byte-21 mapping is correct? A capture of the edit buffer after selecting
  each voice mode on the panel would settle it. `sysex_roundtrip_diag --compare` can do this.
- Does the committed `berlin_school.mid` still match `export_midi` output byte for byte?
- Should `quintal --export` move to a feature-free example, as `export_midi` did?

## Sources

- `crates/minilogue-xd/Cargo.toml:29-62` — example declarations and features.
- `crates/minilogue-xd/examples/*.rs` — line ranges cited inline.
- `crates/minilogue-xd/examples/berlin_school-legend.txt`, `berlin_school.mid` — committed outputs.
- `crates/minilogue-xd/src/device.rs:1-18`, `:40-82`, `:183-258` — ports and discovery.
- `crates/minilogue-xd/src/controller.rs:29-39`, `:123-127`, `:438-474` — live wire bytes.
- `crates/minilogue-xd/src/sysex/program/mod.rs:192-241` — request builders.
- `crates/minilogue-xd/src/sysex/program/synth.rs:656-685`, `:715`, `:1001` — voice-mode and mod-FX decoding.
- `crates/minilogue-xd/src/param/enums.rs:265-285` — `ModFxType` and `VoiceModeType` tables.
- `crates/minilogue-xd/src/midi_file.rs:138`, `:846-855` — SMF note-off and legend.
- `crates/minilogue-xd/src/lib.rs:93-94` — `theory` re-export.
- `README.md:238-253` — examples list.
- `docs/design/05-active/0002-midi-file-export-implementation-plan.md:97-101` — feature-free export example.
- Korg MIDI Impl rev 1.01 *3-5, TABLE 2 *note P3, *note P12.
- `music_comp_mt` 0.5.0 (crates.io, not in repo) — `Note::midi_pitch`, `Chord::from_regex`, `Chord::notes`.
- Git: `9ea5ecf` (theory dependency), `bc6c452` (ports, F8 skipping), `1c16566` (byte fidelity), `7c1b4ad` (`read_edit_buffer`).
