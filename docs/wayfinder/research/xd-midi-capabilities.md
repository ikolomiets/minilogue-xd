# XD MIDI capabilities and playback constraints

Research date: 2026-09-13. Repository baseline: `361d0185dac2b79382788a806a30d3606ee886a7`.
Question: [Establish XD MIDI capabilities and playback constraints](../issues/xd-dsl-0002-xd-midi-capabilities.md).

## Evidence and conclusion

The required control groups are expressible through channel messages and program SysEx, but this library cannot yet be treated as a hardware-conformant oracle. **Documented** below means stated by Korg; **source-verified** means inspected code, not tested hardware. No device measurements or new tests were performed. Discrepancies are evidence gaps, not automatically proven software defects.

Primary references:

- [Korg MIDI Implementation revision 1.01, 2020-02-10](https://cdn.korg.com/us/support/download/files/5227b0b2f7c1da5acd0911ac3e8c4f38.txt) (downloaded and read; [official download page](https://www.korg.com/us/support/download/manual/0/811/4440/)). Section references below identify the relevant tables.
- [Korg owner's manual, edition E8](https://cdn.korg.com/us/support/download/files/a5bba576f00973bf63c78cdeff83a7f7.pdf), especially MIDI configuration and implementation chart, pp. 46–48, 56–58, 68.
- [Korg specifications](https://www.korg.com/us/products/synthesizers/minilogue_xd/specifications.php).
- [MIDI Association electrical specification update](https://midi.org/wp-content/uploads/wpforo/default_attachments/1709416667-ca33-MIDI-10-Electrical-Specification-Update.pdf).

## Capability inventory

| Group | MIDI surface and repository exposure | Qualification |
|---|---|---|
| Played music | Note On/Off, velocity, pitch bend, channel pressure, hold/damper and channel-mode messages; `message/channel.rs`, `controller.rs` | Korg §2-1 documents reception. Controller note calls send immediately. Musical durations, ties, chords and tempo maps require host interpretation. |
| Analog sound | CC controls for VCO pitch/shape/wave/octave, mixer, sync/ring/cross modulation, filter, drive/keytrack, envelopes and LFO; `param/cc.rs` | Group coverage exists in typed `CcParam`; raw values are not automatically physical units such as Hz or milliseconds. |
| Digital and effects | CC multi-engine mode/subtype/shape; modulation-effect type/subtype/time/depth, delay/reverb type/time/depth/mix and enables | Subtype meaning depends on the selected engine/effect. Installed user slots must match the document's intended content. |
| Extended program sound | NRPN names, voice mode, per-engine shape/selection, bend range, joystick/CV assignment/range, tuning selection, LFO synchronization, velocity response, routing, legato/portamento, level, VPM and user parameters; `param/nrpn.rs` | Korg §*3 lists these groups. Several addresses differ from source; see below. Six user-oscillator parameters have module-dependent meaning. Tuning-table administration is outside scope. |
| Patch selection | Bank Select and Program Change; `RealtimeController::program_change`, `ProgramNumber` | Selects an existing program; distinct from transferring a patch or persisting one. Library models slots 0–499. Bank encoding needs verification. |
| Patch transfer and persistence | Korg functions `0x10`/`0x40` request/send current edit buffer; `0x1C`/`0x4C` request/send stored program; `sysex/program/mod.rs`, `sysex/transaction.rs` | Korg §3 (1),(2),(4),(5) explicitly distinguishes edit buffer from internal-memory writes and returns status. Stored dump is itself a write; no separate save command should be assumed. |
| Onboard sequence | Program data contains sequence settings, step notes/velocities/gates, activation masks and four motion lanes; `sysex/program/sequencer.rs`, `builder/sequence.rs` | This is program-data editing, not a stream of arbitrary host score events. No dedicated per-step mutation message was found in the reviewed MIDI command tables. |
| Onboard transport | Timing Clock, Start, Continue, Stop and Song Position Pointer; `message/realtime.rs`, `message/common.rs` | Korg §2-2/2-3 documents reception; SPP uses XD-specific step/resolution interpretation. Do not infer arbitrary host-score seeking from generic MIDI SPP support. |

Inventory sources: Korg MIDI §2-1, *3, §2-2/2-3 and TABLE 2; [CC source](../../../crates/minilogue-xd/src/param/cc.rs), [NRPN source](../../../crates/minilogue-xd/src/param/nrpn.rs), [controller](../../../crates/minilogue-xd/src/controller.rs), [program](../../../crates/minilogue-xd/src/sysex/program/mod.rs), [sequence](../../../crates/minilogue-xd/src/sysex/program/sequencer.rs).

## Encoding and ordering facts

Korg's high-resolution CC protocol sends **CC63 containing the lower three bits, then the parameter CC containing the upper seven**. NRPN selects an address with CC99/98 and commits data through CC6, optionally preceded by CC63. The *3 notes specify 3+7-bit, 3+5-bit and 7+7-bit layouts for 10-, 8- and 14-bit values. This is not generic CC38 data-entry LSB. Source implements those layouts in [encoding.rs](../../../crates/minilogue-xd/src/param/encoding.rs).

**Inference:** competing writers must not interleave parameter/address/data messages on the same device channel; a valid byte encoder alone does not provide this guarantee. The controller performs successive `send` calls, without timestamps or a playback queue. Effects and engine selectors also introduce semantic ordering dependencies. A single documented MIDI setting may require several wire messages; “simultaneous” updates cannot mean simultaneous serial arrival.

## Hard constraints and performance evidence

Korg specifies four voices, a 16-step sequencer and four motion sequences. Its program format/source reserves eight note entries per step; those entries do **not** establish eight-voice hardware polyphony. TABLE 2 specifies onboard length 1–16, resolutions from whole through sixteenth notes, and program tempo 10.0–300.0 BPM. These onboard limits do not directly limit rational timing in a host score. Voice mode changes effective note behavior. [Korg specifications](https://www.korg.com/us/products/synthesizers/minilogue_xd/specifications.php), MIDI TABLE 2, and [sequencer source](../../../crates/minilogue-xd/src/sysex/program/sequencer.rs).

For host clock, configure the XD to receive the chosen MIDI/USB source; reception of CC, program changes, pitch bend and transport has device-setting prerequisites. Installed custom content and a matching MIDI channel are also prerequisites, not new administration features. The manual says Auto clock falls back to internal timing when no external clock arrives. [Owner's manual](https://cdn.korg.com/us/support/download/files/a5bba576f00973bf63c78cdeff83a7f7.pdf), pp. 46–48, 56–58.

DIN MIDI carries 31,250 bits/s with ten serial bits per byte: **calculated** 3,125 bytes/s, or 0.32 ms/byte. Without running status, a three-byte note costs 0.96 ms, paired high-resolution CC costs 1.92 ms, and four-CC NRPN costs 3.84 ms. These are wire occupancy, not device latency guarantees. A library-generated 1,024-byte program expands to 1,171 packed data bytes plus framing, consuming roughly 377 ms over DIN before response/processing. USB does not inherit that DIN wire bound, but device processing capacity remains unmeasured. Sources: electrical specification above; [7-bit codec](../../../crates/minilogue-xd/src/codec.rs) and program encoder. Clock is 24 messages per quarter note in [realtime.rs](../../../crates/minilogue-xd/src/message/realtime.rs).

[SysexTransaction](../../../crates/minilogue-xd/src/sysex/transaction.rs) synchronously waits for responses with a configurable timeout. Successful sending or ACK is not proof of bounded audible onset, uninterrupted playback, or persistence after power cycling. There is no evidence here for safe automation frequency, queue depth, jitter budget, SysEx/clock interleaving by the selected backend, or patch-switch settling time.

## Conformance discrepancies requiring evidence

These are examples sufficient to reject an unqualified completeness claim, not an exhaustive audit:

- **NRPN addresses:** official *3 names occupy `0x01–0x0C`, voice mode is `0x10`, and noise selector is `0x11`; source uses names `0x00–0x0B`, voice mode `0x0C`, noise selector `0x0D`. Some later addresses agree. Resolve address-by-address with captured device traffic; do not apply a universal offset.
- **Bank select:** official §2-1 says CC0=0 and CC32=bank 0–4; controller sends CC0=bank and CC32=slot 0–99. README calls bank encoding an erratum, but this investigation found no hardware capture proving this particular sequence.
- **Conflicting Korg editions:** TXT assigns delay/reverb enables to CC93/94 (matching source), while E8's chart lists 94/95. TXT omits Volume/Expression CC7/11 from the receive table, E8 lists them, and typed `CcParam` does not expose them. TXT documents Song Position Pointer and channel pressure; older charts must not override that silently.
- **Resolution ambiguity:** source treats alternative Voice Mode Depth CC59 as 10-bit, while TXT lists it without the high-resolution footnote.
- **Dump sizes:** TXT dump diagrams say 336 raw bytes, while TABLE 2 extends through byte 1023 and source emits 1024. README additionally reports 448-byte preset-file blobs. File variants do not prove an on-wire size; capture request/reply traffic. Program-number bit annotations also disagree between request/dump diagrams.

Sources: official MIDI sections cited above; E8 p.68; [README errata](../../../README.md#known-documentation-errata); [NRPN](../../../crates/minilogue-xd/src/param/nrpn.rs), [CC](../../../crates/minilogue-xd/src/param/cc.rs), [controller](../../../crates/minilogue-xd/src/controller.rs), [program](../../../crates/minilogue-xd/src/sysex/program/mod.rs).

## Resolution and newly sharp question

The research question is answered at the **documented capability and source-comparison** level; hardware behavior and quantitative performance remain unverified. No syntax, scheduling policy or permissions policy was selected.

A new decision is now precise: **What firmware-specific conformance evidence and compatibility behavior must gate the adapter's claim of full MIDI control when Korg documents and current encoders disagree?** It should establish authoritative parameter mappings, device-capture fixtures and explicit handling of unsupported/ambiguous operations before an implementation-ready contract promises them. Measurement-dependent acceptance criteria should cover patch selection, edit-buffer loading, stored writes/readback, onboard sequence replacement during playback, NRPN ordering, sustained automation with clock, voice stealing and stop/disconnect behavior.
