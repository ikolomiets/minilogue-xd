---
title: USB MIDI ports and port discovery
category: protocol
sources:
  - crates/minilogue-xd/src/device.rs
  - crates/minilogue-xd/src/transport.rs
  - crates/minilogue-xd/examples/list_ports.rs
  - crates/minilogue-xd/examples/test_drive.rs
  - crates/minilogue-xd/examples/read_patch.rs
  - crates/minilogue-xd/examples/sysex_roundtrip_diag.rs
  - docs/korg-user-guide/book.md
related:
  - design/transport.md
  - protocol/midi-messages.md
  - protocol/sysex-framing.md
  - design/sysex-transactions.md
  - protocol/global-blob.md
  - zig/platform-midi.md
  - protocol/errata.md
updated: 2026-10-11
---

# USB MIDI ports and port discovery

> Over its single USB-B connector, the minilogue xd appears as **four** USB MIDI ports: two
> destinations, **SOUND** and **MIDI OUT**, and two sources, **KBD/KNOB** and **MIDI IN**. Use
> **SOUND** for real-time control. For SysEx transactions, **send on SOUND and listen on
> KBD/KNOB**. The Rust crate finds ports with a **case-sensitive substring match** on fixed
> name patterns. It tries the preferred port first, then the other port in the same
> direction.

## Facts

### The four ports

The names are fixed strings that the crate searches for. No Korg document in the repo lists
them. They are device-reported names observed by the crate authors **[hw]/[rust]**. Direction
is given from the host's point of view.

| Host direction | Pattern (substring) | Display | Rust enum | Carries | Use for |
|---|---|---|---|---|---|
| Output (host → XD) | `minilogue xd SOUND` | `SOUND` | `OutputPort::Sound` | Input to the synth engine: notes, CC, NRPN, program change, SysEx requests/dumps | Real-time control, SysEx requests |
| Output (host → XD) | `minilogue xd MIDI OUT` | `MIDI OUT` | `OutputPort::MidiOut` | Forwarded to the 5-pin DIN MIDI OUT jack (current docs) | Driving external gear through the XD |
| Input (XD → host) | `minilogue xd KBD/KNOB` | `KBD/KNOB` | `InputPort::KbdKnob` | Keyboard and knob events from the XD, **SysEx replies** (dumps, ACK/NAK), clock when master | SysEx responses, recording performance |
| Input (XD → host) | `minilogue xd MIDI IN` | `MIDI IN` | `InputPort::MidiIn` | Data received on the 5-pin DIN MIDI IN jack | Listening to external gear |

Preference order is `OutputPort::ALL = [Sound, MidiOut]` and `InputPort::ALL = [KbdKnob, MidiIn]`
(`device.rs:50, 84`).

### Recommended port pairs [rust, hw]

| Task | Send on | Receive on | Evidence |
|---|---|---|---|
| Real-time control (notes, CC, NRPN, pitch bend, program change) | SOUND | — | `examples/test_drive.rs:21-29`; module doc `device.rs:13` |
| SysEx request/response (program, global, tuning, user module) | SOUND | KBD/KNOB | `examples/read_patch.rs:25-41`; `examples/sysex_roundtrip_diag.rs:68-78`; commit `1c16566` reports byte-identical hardware writes using this pair **[hw]** |
| External DIN routing | MIDI OUT | MIDI IN | module doc `device.rs:15` |

Commit `213020c` changed the default input from `MIDI IN` to `KBD/KNOB`. The reason given:
"KbdKnob (synth responses, SysEx)". The original `find_input_port` searched "MIDI IN"
(commit `cdc0a73`).

### Device-side settings that affect what flows

These are described in [global blob](global-blob.md). The user guide documents them, but not
the port names.

| Setting | Values | Effect on USB |
|---|---|---|
| MIDI Route (TABLE 1 offset 15) | `USB+MIDI` / `USB` | `USB+MIDI`: receives from USB and DIN IN, transmits to USB and DIN OUT. `USB`: USB only (`book.md:1897-1903`) |
| MIDI Ch (offset 16) | 0..15 | Channel used for notes, CCs and dumps (`book.md:1905-1909`) |
| Clock Source (offset 17) | Auto(USB) / Auto(MIDI) / Internal | Selects which connector's clock is followed. Under Auto, the internal clock is used until external clock arrives (`book.md:1911-1917`, `:2353`) |
| Rx/Tx Prog Chg, CC, PitchBend (offsets 19-24) | Off/On | Gate the channel messages; see [MIDI messages](midi-messages.md) |

The user guide also says [hw/spec]:

- A front-panel "All Dump" sends SysEx over USB-B and takes about 20 s.
- The dump is not transmitted unless the host's MIDI input port is open (`book.md:2065-2067`).
- A front-panel "Program Dump" is sent on both MIDI OUT and USB-B (`book.md:1640`).

### Discovery algorithm (Rust behaviour)

```text
list_output_ports():                 ; also list_input_ports(), same shape
  client = OS MIDI client named "minilogue-xd"
  return [port_name(p) for p in client.ports() if port_name(p) succeeds]   ; failures skipped

find_output(prefer):                 ; find_input(prefer) is identical over InputPort
  names = list_output_ports()
  if some n in names (first in OS order) contains pattern(prefer): return n
  for variant in OutputPort.ALL where variant != prefer:
    if some n in names contains pattern(variant): return n
  return None

find_output_port() = find_output(Sound)
find_input_port()  = find_input(KbdKnob)
```

- Matching uses `str::contains`, which is case-sensitive. There is no normalisation.
- The first match in OS enumeration order wins. With two XDs attached, the result depends on
  enumeration order, and nothing selects a particular unit.
- **Fallback crosses roles.** If SOUND is missing, `find_output(Sound)` returns MIDI OUT. If
  KBD/KNOB is missing, `find_input(KbdKnob)` returns MIDI IN. SysEx replies never arrive on
  MIDI IN, so that fallback turns a missing port into a later transaction timeout.
- `MidirOutput::connect(name)` and `MidirInput::connect(name)` match `name` by substring again
  (`transport.rs:149-178`, `transport.rs:241-284`). Passing just the pattern works too.
- Client and connection names that the crate registers with the OS: client `minilogue-xd`,
  output connection `minilogue-xd-out`, input connection `minilogue-xd-in`
  (`device.rs:116-130`).
- `PRODUCT_NAME = "minilogue xd"` is used for file metadata, not for discovery
  (`device.rs:113-114`).

### Platform naming

The crate has **no platform-specific handling**. It relies on substring matching to tolerate
OS decoration:

- ALSA (midir) builds names from the client name, the port name and the numeric
  `client:port` ids.
- CoreMIDI reports endpoint display names.
- WinMM truncates names to 31 characters (`MAXPNAMELEN` = 32 including the NUL).

The longest pattern, `minilogue xd MIDI OUT` (21 characters), fits within the WinMM limit.
**The patterns are probably verified only on macOS.** This is inferred: the [hw]-verified
examples were developed on a Mac. The Windows class driver or the KORG USB-MIDI driver may
expose different names altogether. See [platform MIDI](../zig/platform-midi.md) and Open
questions.

## Invariants

- The XD exposes exactly 2 destinations and 2 sources. No pattern is a substring of another
  pattern.
- A SysEx transaction's output port is SOUND and its input port is KBD/KNOB. A Zig port should
  assert this pairing by role (an enum), not by string.
- Discovery is a pure function of the enumerated name list and the preference. Given the same
  list, it returns the same port.
- Name patterns are ASCII and at most 21 bytes long.

## Errata & gotchas

- **Conflicting descriptions of SOUND vs MIDI OUT** [rust history]:
  - The first version (commit `cdc0a73`) described SOUND as "direct to sound engine
    (bypasses internal MIDI router)". It described MIDI OUT as routing "through the
    Minilogue XD's MIDI implementation (channel filtering, local on/off)".
  - The current docs say MIDI OUT is "forwarded to the physical MIDI OUT jack"
    (`device.rs:6-11, 35-37`).
  - Only the SOUND path has [hw] evidence (commit `1c16566`).
- **Silent fallback** can pick MIDI IN for SysEx replies, as described above.
- Messages that the XD transmits on KBD/KNOB are gated by the "MIDI Tx …" settings. Clock
  is gated by "En Rx Transport" (see [MIDI messages](midi-messages.md)). If these are Off, the
  host sees no CC or clock traffic, but SysEx replies still arrive.
- CV IN CC 118/119 are never sent over USB [spec] (Korg MIDI Impl rev 1.01 §1-1 `*1-5`).

## Test vectors

Discovery needs real OS ports, so the unit tests check only the patterns, display strings and
`ALL` ordering:

| Check | Expected | Citation |
|---|---|---|
| `OutputPort::Sound.port_name_pattern()` | `minilogue xd SOUND` | `crates/minilogue-xd/src/device.rs:345-452` (tests module) |
| `OutputPort::MidiOut.port_name_pattern()` | `minilogue xd MIDI OUT` | same |
| `InputPort::KbdKnob.port_name_pattern()` | `minilogue xd KBD/KNOB` | same |
| `InputPort::MidiIn.port_name_pattern()` | `minilogue xd MIDI IN` | same |
| `Display` | `SOUND`, `MIDI OUT`, `KBD/KNOB`, `MIDI IN` | `crates/minilogue-xd/src/device.rs:53-60, 87-94` |

A suggested Zig table test for `find` over a fixed name list (derived, not from Rust):

| Enumerated names | `find_output(sound)` | `find_input(kbd_knob)` |
|---|---|---|
| `["IAC Bus 1", "minilogue xd SOUND", "minilogue xd MIDI OUT"]` | `minilogue xd SOUND` | none |
| `["minilogue xd MIDI OUT"]` | `minilogue xd MIDI OUT` (fallback) | — |
| `["minilogue xd:minilogue xd KBD/KNOB 24:1", "minilogue xd:minilogue xd MIDI IN 24:2"]` | — | first name |
| `["Minilogue XD SOUND"]` | none (case-sensitive) | — |

## Rust design

- `OutputPort` and `InputPort` are `#[non_exhaustive]` `Copy` enums. Each has
  `port_name_pattern()`, `Display` and a preference-ordered `ALL` slice (`device.rs:29-95`).
- String constants (`OUTPUT_PORT_SOUND`, …) remain for backward compatibility
  (`device.rs:97-114`).
- Discovery functions (`list_*`, `find_*`) are behind `cfg(feature = "midi-io")`. They return
  `Result<Option<String>>`. `Err(MidiIo)` means the OS MIDI client could not be created.
  `Ok(None)` means no XD was found (`device.rs:132-258`).
- Each call creates a fresh midir client. Discovery is stateless and allocates a `String` per
  port.
- `device::cc_name` lives in the same module. It is covered in
  [transport](../design/transport.md) and [CC map](cc-map.md).

## Zig / TigerStyle notes

- Store patterns in a comptime table, `[4][]const u8`, indexed by an `enum(u2)` port role. Keep
  direction as a separate field so the type system blocks an output role from being opened as
  an input.
- Enumerate into a caller-provided fixed array: `port_count_max = 64` names, each
  `port_name_bytes_max = 128`. Truncate longer names and flag truncation in a counter. Never
  allocate.
- Make fallback **explicit**: an options struct
  `.{ .role = .sound, .fallback = .none }`. The default for SysEx is no fallback. A missing
  KBD/KNOB port should fail at discovery with `error.PortNotFound`, not later as a timeout.
- Support picking among several XDs by returning all matches, up to a bound, and letting the
  caller choose by index.
- Platform enumeration (CoreMIDI, ALSA sequencer or raw MIDI, WinMM/WinRT) is described in
  [platform MIDI](../zig/platform-midi.md). The match logic stays platform-independent.

```zig
// sketch
pub const PortRole = enum(u2) { sound, midi_out, kbd_knob, midi_in };
pub const port_patterns = [_][]const u8{
    "minilogue xd SOUND", "minilogue xd MIDI OUT", "minilogue xd KBD/KNOB", "minilogue xd MIDI IN",
};
pub fn find(names: []const []const u8, role: PortRole) ?u8 { // index into names
    for (names, 0..) |name, index| {
        if (std.mem.indexOf(u8, name, port_patterns[@intFromEnum(role)]) != null) return @intCast(index);
    }
    return null;
}
```

## Open questions

- What exact port names does each OS report? This needs Linux ALSA, Windows (Korg USB-MIDI
  driver vs the class driver) and macOS captures. Does the Windows class driver expose all four
  ports with these names?
- Does SOUND bypass the MIDI channel filter or Local setting, as the first docs claimed? Does
  MIDI OUT route through the XD's MIDI implementation or go straight to the DIN jack?
- Does the desktop module expose the same four ports with the same names? The README claims
  both variants are supported, but no evidence is recorded.
- Does the XD echo incoming SOUND traffic on KBD/KNOB? A Zig input queue sized for SysEx
  replies must also absorb that traffic.

## Sources

- `crates/minilogue-xd/src/device.rs:1-18` — four-port table and usage advice.
- `crates/minilogue-xd/src/device.rs:29-95` — `OutputPort` and `InputPort`.
- `crates/minilogue-xd/src/device.rs:97-130` — legacy constants and client/connection names.
- `crates/minilogue-xd/src/device.rs:132-258` — list and find functions.
- `crates/minilogue-xd/src/device.rs:345-452` — tests.
- `crates/minilogue-xd/src/transport.rs:139-178, 225-284` — `connect` substring match.
- `crates/minilogue-xd/examples/list_ports.rs:5-24`
- `crates/minilogue-xd/examples/test_drive.rs:21-29`
- `crates/minilogue-xd/examples/read_patch.rs:25-41`
- `crates/minilogue-xd/examples/sysex_roundtrip_diag.rs:68-78`
- `docs/korg-user-guide/book.md:1897-1917, 1640, 2065-2067, 2353` — MIDI Route, MIDI Ch,
  Clock Source, dumps.
- Git commits `cdc0a73` (device module, original MIDI IN default), `213020c` (enums,
  KBD/KNOB default), `1c16566` ([hw] SysEx round trip).
- Korg MIDI Impl rev 1.01 §1-1 `*1-5`, TABLE 1 offsets 15-24.
