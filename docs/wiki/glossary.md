---
title: Glossary
category: meta
sources:
  - CONTEXT.md
  - crates/minilogue-xd/src/lib.rs
related:
  - index.md
  - overview.md
updated: 2026-11-11
---

# Glossary

> These are the canonical terms used across this wiki, one definition each. A term's linked page
> holds the full facts; the definition here only fixes the meaning. Terms from the project's
> performance vocabulary (`CONTEXT.md`) are marked *(domain)*. When a page uses a term in
> another sense, the page is wrong. Fix it there.

## A–C

- **Acceptance demo** — a runnable example used as an end-to-end check against real hardware,
  not as a unit test. See [examples](design/examples.md) and [test plan](zig/test-plan.md).
- **ACK / NAK** — the Korg status frames that answer a write: `0x23` (DATA LOAD COMPLETED) and
  `0x24` (DATA LOAD ERROR) to `0x2F` (user-module errors). In Zig a NAK is the operating error
  `error.Nak`, with the status byte in `Diagnostics`; it is not a transport failure. See
  [SysEx framing](protocol/sysex-framing.md) and [errors](design/errors.md).
- **Active step** — a sequencer step that the playback cursor visits (header bits at absolute
  162–163, firmware 2.0+). Not the same as *step on*. See
  [sequencer blob](protocol/sequencer-blob.md).
- **AFK / HITL** — Wayfinder ticket modes. AFK means an agent can resolve the ticket unattended;
  HITL (human in the loop) means it needs the user live. See [Wayfinder DSL](sources/wayfinder-dsl.md).
- **Band value** — see *TX value / RX band*.
- **Byte fidelity** — the requirement that parsing then serializing a real program reproduces
  every byte, including bytes the typed model does not understand. See
  [byte fidelity](design/byte-fidelity.md).
- **CC63 low part** — the low bits of a 10-bit value, sent first on CC63 ("LSB VALUE"). The high
  part follows on the parameter's own CC. See [parameter encodings](protocol/parameter-encodings.md).
- **CC legend** — the text file listing which CC numbers an exported Standard MIDI File uses. See
  [MIDI file export](design/midi-file-export.md).
- **Canonical encoding** — the one 7-bit encoding that `encode_7bit` produces for a given input.
  A decoder may also accept non-canonical trailing MSB bits. See [7-bit codec](protocol/korg-7bit-codec.md).
- **CentOffset** — one tuning entry: a semitone byte plus a 14-bit fraction of a semitone
  (1/16384 semitone ≈ 0.0061 cent). See [tuning](protocol/tuning.md).
- **Channel (`g`)** — the global MIDI channel, 0–15, carried in the low nibble of byte 2 (`3g`)
  of a Korg exclusive header. See [SysEx framing](protocol/sysex-framing.md).
- **Channel voice / mode message** — MIDI messages with status `0x80`–`0xEF`. Mode messages are
  control changes 120–127. See [MIDI messages](protocol/midi-messages.md).
- **Context-dependent CC** — a CC whose value meaning depends on another parameter. CC96 depends
  on the mod FX type and CC103 on the multi engine type. See
  [CC map](protocol/cc-map.md#context-dependent-ccs-96-103).
- **CRC-32 (IEEE)** — the reflected polynomial `0xEDB88320` checksum used by user-module slot
  data and by zip entries. See [user modules](protocol/user-modules.md) and
  [file formats](protocol/file-formats.md).

## D–F

- **Data byte** — a MIDI byte with bit 7 clear (`0x00`–`0x7F`). See [MIDI messages](protocol/midi-messages.md).
- **Deadline** — the monotonic time (`now_ns + timeout`) after which a transaction fails with
  `Timeout`. See [SysEx transactions](design/sysex-transactions.md).
- **Delta time** — the tick distance from the previous event in a Standard MIDI File, written as
  a VLQ. See [standard MIDI file](protocol/standard-midi-file.md).
- **Echo-back ID** — the byte that a Korg Search Device request carries and the reply copies
  back, so the host can match replies. See [device identity](protocol/device-identity.md).
- **Edit buffer** — the XD's current, unsaved program, also called "current program". It is read
  with `0x10`→`0x40` and loaded with `0x40`. Not the same as *internal memory*. See
  [program blob](protocol/program-blob.md).
- **Encoded length** — the wire length of 7-bit-encoded data: `n + ceil(n / 7)`. See
  [7-bit codec](protocol/korg-7bit-codec.md) and [memory budget](zig/memory-budget.md).
- **Errata register** — the single table of every spec/hardware/Rust disagreement, with row IDs
  (`M1`, `C5`, `S1`, …). See [errata](protocol/errata.md).
- **Exhaustive test** — a test that enumerates a whole small input domain, for example all 128
  RX values. The Zig port prefers it to random property tests. See
  [testing strategy](design/testing-strategy.md).
- **Family / member ID** — the device identifiers in an identity reply: family `51 01` (XD),
  member `00 00`. See [device identity](protocol/device-identity.md).
- **Favorite entry** — one of 16 global slots that each point at a program (0–499). See
  [global blob](protocol/global-blob.md).
- **Frontier** — in a Wayfinder map, the open tickets that are unassigned and unblocked. See
  [Wayfinder DSL](sources/wayfinder-dsl.md).
- **Function ID** — byte 6 of a Korg exclusive frame. It names the request, dump or status
  (`0x0E`, `0x40`, `0x4C`, `0x23`, …). See [SysEx framing](protocol/sysex-framing.md).

## G–M

- **GLOB blob** — the 63-byte global-parameter blob (TABLE 1); it starts with the magic `GLOB`.
  See [global blob](protocol/global-blob.md).
- **Golden vector** — exact input and output bytes captured from Rust tests or real files and
  committed for the Zig tests to compare against. See [test plan](zig/test-plan.md).
- **Host clock** *(domain)* — the computer's timing authority for score playback. It may drive
  the XD with MIDI clock. See [Wayfinder DSL](sources/wayfinder-dsl.md).
- **Internal memory** — the XD's 500 stored program slots (and its tuning slots), written with
  `0x4C`. Not the same as the *edit buffer*. See [program blob](protocol/program-blob.md).
- **KBD/KNOB port** — the XD's USB MIDI source that carries knob moves and SysEx replies. See
  [USB ports](protocol/usb-ports.md).
- **Korg exclusive header** — the 7 bytes `F0 42 3g 00 01 51 fid` that start every XD-specific
  SysEx frame. See [SysEx framing](protocol/sysex-framing.md).
- **Korg 7-bit packing** — NOTE 1 of the spec. Each group of up to 7 data bytes is preceded by
  one *MSB byte* holding their bit 7s. See [7-bit codec](protocol/korg-7bit-codec.md).
- **Live parameter change** *(domain)* — a sound-parameter adjustment supplied during score
  playback; it does not rewrite the score. See [realtime controller](design/realtime-controller.md).
- **Load size** — the byte length of a user module's program code, from its header. See
  [user modules](protocol/user-modules.md).
- **Magic** — fixed ASCII at a blob's start that identifies it: `PROG` (program), `PREDSQ`
  (sequencer section), `GLOB` (global). See [program blob](protocol/program-blob.md).
- **Memory budget** — the list of comptime maxima that every Zig buffer is declared from. See
  [memory budget](zig/memory-budget.md).
- **Meta event** — a non-MIDI SMF event (`FF type len data`), for example track name, tempo and
  end of track. See [standard MIDI file](protocol/standard-midi-file.md).
- **MIDI Route** — the global setting choosing `USB+MIDI` or `USB` I/O. See
  [USB ports](protocol/usb-ports.md).
- **MockOutput / MockMidiInput** — Rust's in-memory test doubles for MIDI output and input. See
  [transport](design/transport.md).
- **Motion slot** — one of 4 sequencer lanes that record a parameter's motion across steps. See
  [sequencer blob](protocol/sequencer-blob.md#motion-slot-parameter-s2).
- **MSB byte** — the first byte of each 7-bit codec group, carrying the stripped bit 7s. See
  [7-bit codec](protocol/korg-7bit-codec.md).
- **MThd / MTrk** — the Standard MIDI File header chunk and track chunk. See
  [standard MIDI file](protocol/standard-midi-file.md).
- **MTS bulk dump / single note tuning change** — the MIDI Tuning Standard messages
  `F0 7E dd 08 01 …` (408 bytes) and `F0 7F dd 08 02 …`. See [tuning](protocol/tuning.md).

## N–R

- **NRPN** — Non-Registered Parameter Number. It is addressed with CC99/CC98, and the value goes
  on CC6 (and CC63 for the low part). See [NRPN map](protocol/nrpn-map.md).
- **ODM** — the tool that manages numbered design documents in `docs/design/`. See
  [design docs](sources/design-docs.md).
- **Onboard sequence** *(domain)* — a sequence held and played by the XD's built-in sequencer,
  not a host-performed score. See [sequencer blob](protocol/sequencer-blob.md).
- **Operating error / programmer error** — TigerStyle's split. Operating errors (bad input,
  timeouts) are Zig error-set values and are always handled. Programmer errors are assertions.
  See [errors](design/errors.md).
- **Patch** *(domain)* — a configuration of the synth's sound parameters, distinct from the music
  played with it. On the wire it is a program blob. See [program blob](protocol/program-blob.md).
- **Patch snapshot** — `MidiFileBuilder::patch_ccs`, which writes a program's parameters as CCs
  at the start of an SMF. See [MIDI file export](design/midi-file-export.md).
- **Platform ID** — the logue SDK target platform: prologue 1, minilogue xd 2, NTS-1 3. See
  [user modules](protocol/user-modules.md).
- **Poly chain Master / Slave** — two linked units. The Master sends note SysEx (`0x60`/`0x61`)
  for the voices it hands to the Slave. See [poly chain](protocol/poly-chain.md).
- **PREDSQ / PREDSEQD** — the sequencer-section magic. Firmware 1.xx data has `SEQD` and no
  active-step field, and the device converts it to `SQ` on load. Rust accepts only `PREDSQ`. See [sequencer blob](protocol/sequencer-blob.md#magic-pred--sq-or-seqd).
- **Profile (build)** — the Zig build option that sizes buffers either for core use or for
  user-module slot data. See [memory budget](zig/memory-budget.md#build-profiles).
- **Program value** — the byte stored for a stepped parameter in the program blob. Often 0-based
  where the TX value is a band value. See [stepped params](design/stepped-params.md).
- **Range-checked newtype** — Rust's `U4`/`U7`/`U14`/`TenBitParam`, which reject out-of-range
  values on construction. In Zig these become native `u4`/`u7`/`u14`/`u10`. See
  [typed values](design/typed-values.md).
- **Raw base / overlay** — byte-fidelity technique. The parser keeps the original 1024-byte blob
  (raw base), and serialization writes typed fields on top of it (overlay), so unknown bits
  survive. See [byte fidelity](design/byte-fidelity.md).
- **Raw prefix** — the unencoded leading bytes of some user-module replies, before the 7-bit
  body. See [user modules](protocol/user-modules.md).
- **Raw vs encoded payload** — whether a SysEx body is sent as plain 7-bit data (raw) or with
  Korg 7-bit packing (encoded). See [SysEx framing](protocol/sysex-framing.md).
- **Read-modify-write** — request a program, change fields, send it back. With a raw base this is
  the only safe way to edit a device program. See [byte fidelity](design/byte-fidelity.md).
- **Real-time byte** — a system real-time message (`0xF8`–`0xFF`). It may appear inside SysEx and
  must not break the frame. See [MIDI messages](protocol/midi-messages.md).
- **Running status** — omitting a repeated status byte. The Rust MIDI path never sees it, and
  the Rust SMF writer never emits it. See [MIDI messages](protocol/midi-messages.md#errata--gotchas).

## S–Z

- **Sans-I/O** — the design rule that the protocol core takes bytes in and returns bytes out,
  with no OS calls, so it is fully testable. See [architecture](design/architecture.md).
- **Score** *(domain)* — a prepared description of music to perform, including timing and planned
  sound changes. Not a patch. See [Wayfinder DSL](sources/wayfinder-dsl.md).
- **Score automation** *(domain)* — sound-parameter changes planned at musical times inside a
  score, as opposed to live parameter changes. See [Wayfinder DSL](sources/wayfinder-dsl.md).
- **Signature area** — the 132 bytes after a user module's code, zero-filled by community tools.
  See [user modules](protocol/user-modules.md).
- **Simulated device** — a deterministic model of the XD used in Zig tests, in the spirit of
  TigerBeetle's VOPR. See [test plan](zig/test-plan.md).
- **Slot** — one user-module storage position. Counts per module type: mod FX 16, delay 8,
  reverb 8, oscillator 16. See
  [user modules](protocol/user-modules.md).
- **SMF Format 0** — a Standard MIDI File with one track. It is what the crate exports. See
  [standard MIDI file](protocol/standard-midi-file.md).
- **SOUND port** — the XD's USB MIDI destination that drives the synth engine. See
  [USB ports](protocol/usb-ports.md).
- **Sound parameter** *(domain)* — any setting that shapes the XD's sound, analog or digital.
  See [CC map](protocol/cc-map.md).
- **SPSC ring buffer** — a fixed-capacity single-producer/single-consumer byte queue between the
  OS MIDI thread and the application loop. See [platform MIDI](zig/platform-midi.md).
- **Status byte** — a MIDI byte with bit 7 set that starts a message. See
  [MIDI messages](protocol/midi-messages.md).
- **Step on** — a sequencer step that sounds (bits at absolute 170–171). A visited step can still
  be off. See [sequencer blob](protocol/sequencer-blob.md).
- **Step event** — the 52-byte per-step record: 8 notes, 8 velocities, 8 gates with trigger
  bits, and 4 × 7 bytes of motion data. See
  [sequencer blob](protocol/sequencer-blob.md#step-event-s3--52-bytes).
- **Stepped parameter** — a discrete parameter (an enum) sent as a CC or NRPN band value. See
  [stepped params](design/stepped-params.md).
- **Stored write** — writing a program into internal memory with `0x4C`. See
  [program blob](protocol/program-blob.md).
- **Stream assembler** — the state machine that rebuilds whole SysEx frames from fragmented
  input bytes. See [SysEx transactions](design/sysex-transactions.md).
- **SysEx frame** — one complete `F0 … F7` message. See [SysEx framing](protocol/sysex-framing.md).
- **TABLE 1 / 2 / 5 / 6 / 7** — the spec's tables: global data, program data, user-module info,
  slot status, slot data. See [global blob](protocol/global-blob.md),
  [program blob](protocol/program-blob.md) and [user modules](protocol/user-modules.md).
- **Template blob** — a known-good default 1024-byte program used as the base for programs built
  from scratch. See [byte fidelity](design/byte-fidelity.md).
- **TenBitParam / TenBitSysex** — Rust's two 10-bit value types (CC path and NRPN path). They
  encode identically. See [parameter encodings](protocol/parameter-encodings.md).
- **TIE** — a sequencer gate value of 73–127 that joins a note to the next step. See
  [sequencer blob](protocol/sequencer-blob.md).
- **TPQ** — ticks per quarter note, the SMF time division. See
  [standard MIDI file](protocol/standard-midi-file.md).
- **Transaction** — one request/response exchange: send a request, then wait for the matching
  dump or status frame before the deadline. See [SysEx transactions](design/sysex-transactions.md).
- **Trust boundary** — the point where external bytes become typed values. Range checks happen
  there; inside it, asserts take over. See [TigerStyle mapping](zig/tigerstyle-mapping.md).
- **TX value / RX band** — for a stepped CC, the exact byte sent per value (TX) and the input
  ranges accepted per value on receive (RX). See [CC map](protocol/cc-map.md).
- **User module** — a logue SDK oscillator or effect (mod FX, delay, reverb) loaded into a slot.
  See [user modules](protocol/user-modules.md).
- **User scale / user octave** — custom tunings: 128 notes × 3 bytes (384) and 12 notes × 3
  bytes (36). See [tuning](protocol/tuning.md).
- **`user_prg_header_t`** — the logue SDK's 1024-byte module header (platform, module, load
  size, name). See [user modules](protocol/user-modules.md).
- **VLQ** — variable-length quantity: 7 bits per byte, MSB set on all but the last. It is used
  for SMF delta times. See [standard MIDI file](protocol/standard-midi-file.md).
- **Voice slot** — the 0–3 voice index in a poly-chain note message. See
  [poly chain](protocol/poly-chain.md).
- **Wayfinder map** — the ticket tree (`docs/wayfinder/`) that plans the score DSL work. See
  [Wayfinder DSL](sources/wayfinder-dsl.md).
- **Wire byte** — a byte as sent on MIDI, always ≤ `0x7F` inside SysEx. Not the same as a decoded
  blob byte. See [7-bit codec](protocol/korg-7bit-codec.md).
