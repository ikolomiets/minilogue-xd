---
title: Platform MIDI backends without midir
category: zig
sources:
  - crates/minilogue-xd/src/transport.rs
  - crates/minilogue-xd/src/device.rs
  - /opt/homebrew/Cellar/zig/0.16.0_1/lib/zig/libc/include/any-linux-any/sound/asound.h (Linux uapi, bundled with Zig)
  - /opt/homebrew/Cellar/zig/0.16.0_1/lib/zig/libc/mingw/lib-common/winmm.def (bundled import lib)
related:
  - design/transport.md
  - protocol/usb-ports.md
  - zig/dependency-elimination.md
  - zig/module-layout.md
  - zig/memory-budget.md
updated: 2026-10-11
---

# Platform MIDI backends without midir

> Replace `midir` with three thin, hand-declared OS bindings: **CoreMIDI** (macOS), **ALSA
> rawmidi via `/dev/snd` ioctls** (Linux, no libasound), and **WinMM** (Windows). Each backend
> does only four things: enumerate ports, open by name, write bytes, and push received bytes
> into a fixed **SPSC ring buffer** from the OS callback/reader thread. All parsing, SysEx
> reassembly, and timeouts stay in the sans-I/O core. Following TigerStyle, the program polls
> input at its own pace and never reacts directly to the OS callback.

## Facts

### What the Rust transport needs (the contract to reproduce)

| Rust | Behavior | Zig equivalent |
|------|----------|----------------|
| `MidiOutput::send(&[u8])` | One complete message per call (3 bytes for CC, about 1.2 KB for a dump) | `fn write(self, bytes: []const u8) Error!void`. It must deliver a full SysEx as one unit, or as ordered chunks. |
| `MidiInput::receive(timeout) → Option<Vec<u8>>` | Blocks on an `mpsc` channel fed by the midir callback; one `Vec` per callback | `fn drain(self, out: []u8) usize`. Non-blocking: copies whatever bytes are queued. The caller decides when to poll and how long to wait. |
| `device::list_*_ports`, `find_output/find_input` | Substring match on `"minilogue xd SOUND"`, `"… MIDI OUT"`, `"… KBD/KNOB"`, `"… MIDI IN"`, falling back across variants | `fn enumerate(dir, out: *[max_ports]PortName) u8` plus the same substring matcher, kept *outside* the backend (pure, testable) |
| client name `"minilogue-xd"`, port names `"minilogue-xd-out"/"-in"` | Shown in other apps' port lists | Same constants |

`device.rs:43-114` defines the port name patterns. `transport.rs:136-330` is the midir backend.
`transport.rs:266-276` forwards each callback buffer with `tx.send(message.to_vec())`. That is an
**unbounded** channel with one heap allocation per message, so the Zig port must replace it. See
[transport](../design/transport.md) and [USB ports](../protocol/usb-ports.md).

### macOS: CoreMIDI

Link the system frameworks `CoreMIDI` and `CoreFoundation`. Declare roughly 15 `extern "c"`
functions by hand instead of using `@cImport`. That keeps cross-compilation free of the macOS SDK
and makes the ABI surface explicit.

| Step | API |
|------|-----|
| Client | `MIDIClientCreate(name: CFStringRef, notify: ?MIDINotifyProc, ref_con, *MIDIClientRef)` |
| Enumerate | `MIDIGetNumberOfDestinations()` / `MIDIGetDestination(i)`; `MIDIGetNumberOfSources()` / `MIDIGetSource(i)` |
| Names | `MIDIObjectGetStringProperty(obj, kMIDIPropertyDisplayName, *CFStringRef)`, then `CFStringGetCString(..., kCFStringEncodingUTF8)`, then `CFRelease` |
| Output | `MIDIOutputPortCreate`; build a `MIDIPacketList` (`MIDIPacketListInit` / `MIDIPacketListAdd`, timestamp 0 = now) and call `MIDISend(port, dest, list)` |
| Input | `MIDIInputPortCreate(client, name, read_proc, ref_con, *port)` + `MIDIPortConnectSource(port, src, conn_ref)`. `read_proc` runs on a CoreMIDI real-time thread with a `*const MIDIPacketList`. |

Notes:
- `MIDISend`/`MIDIInputPortCreate` are the legacy MIDI 1.0 byte API, deprecated since macOS 11
  in favor of `MIDISendEventList` / `MIDIInputPortCreateWithProtocol` (UMP, block-based). The
  legacy API still works and needs no Objective-C blocks. Treat migrating as an open question.
- **Walking `MIDIPacketList`:** `MIDIPacketNext` is a header macro, not a symbol. On ARM/arm64
  the next packet starts at `align_forward(&pkt.data[pkt.length], 4)`; on x86 it is unaligned.
  `MIDIPacket` is `{ timeStamp: u64, length: u16, data: [256]u8 }` under `#pragma pack(4)`.
  Model it as an `extern struct` with `align(4)` and walk at most `numPackets` (a bounded loop).
- A SysEx may arrive split over several packets and several callbacks. The core's assembler
  handles this. The backend just copies bytes.
- `MIDIPacketList` capacity for one ~1.2 KB dump: allocate a fixed `[2048]u8` buffer per send.
  `MIDIPacket.length` is a `u16`, so one packet can hold a whole program dump.

### Linux: ALSA rawmidi, no libasound

Everything goes through `std.os.linux` syscalls on `/dev/snd/*`. Zig ships the uapi header
(`sound/asound.h`) and `std.os.linux.IOCTL.{IOR,IOW,IOWR}` to compute request numbers at comptime.

| Step | Mechanism |
|------|-----------|
| Cards | Try `open("/dev/snd/controlC{0..31}")`. `SNDRV_CTL_IOCTL_CARD_INFO = _IOR('U', 0x01, snd_ctl_card_info)` gives the card name. |
| Devices | `SNDRV_CTL_IOCTL_RAWMIDI_NEXT_DEVICE = _IOWR('U', 0x40, int)`, starting at -1 and looping until it returns -1 |
| Port names | `SNDRV_CTL_IOCTL_RAWMIDI_INFO = _IOWR('U', 0x41, snd_rawmidi_info)` per (device, subdevice, stream), where stream is `OUTPUT=0` or `INPUT=1`. Read `name[80]`, `subname[32]`, and `subdevices_count`. |
| Select subdevice | On the **still-open** control fd, call `SNDRV_CTL_IOCTL_RAWMIDI_PREFER_SUBDEVICE = _IOW('U', 0x42, int)`, then `open("/dev/snd/midiC{card}D{dev}", O_RDWR or O_WRONLY or O_RDONLY, O_NONBLOCK)`. The kernel matches the preference by the calling process, which is how alsa-lib's `hw:C,D,S` works. Verify with `SNDRV_RAWMIDI_IOCTL_INFO = _IOR('W', 0x01, …)`. |
| I/O | Plain `write()` / `read()` of raw MIDI bytes. Use `poll()` on the fd from a dedicated reader thread, or from the main loop if you prefer no threads at all. |

`struct snd_rawmidi_info` (uapi as bundled with Zig 0.16): `device u32, subdevice u32, stream
i32, card i32, flags u32, id[64], name[80], subname[32], subdevices_count u32, subdevices_avail
u32, tied_device i32, reserved[60]`. Mirror it as an `extern struct` and `comptime` assert
`@sizeOf`. Older kernels had `reserved[64]` and no `tied_device`; the size is the same.

Notes:
- rawmidi is a **byte stream**, not a message stream. The core's stream parser must handle
  running status, realtime bytes interleaved anywhere, and SysEx split across reads. The Rust
  crate never faced this because midir/ALSA-seq delivered framed messages.
- A rawmidi subdevice is usually opened exclusively per direction. PipeWire, JACK, or a DAW
  holding the port gives `EBUSY`. Report that as a distinct error.
- An alternative is the ALSA **sequencer** (`/dev/snd/seq` ioctls; this is what midir uses
  through libasound). It shares ports with other apps and offers kernel-side scheduled delivery,
  but its uapi is much larger. Choose it only if multi-client or scheduled output is required.

### Windows: WinMM

Zig ships `winmm.def` (mingw import lib), so `linkSystemLibrary("winmm")` cross-compiles from
any host. `std.os.windows` has no WinMM bindings; declare them as `extern "winmm"` with
`callconv(.winapi)`.

| Step | API |
|------|-----|
| Enumerate | `midiOutGetNumDevs` / `midiOutGetDevCapsW`; `midiInGetNumDevs` / `midiInGetDevCapsW`. `szPname` is `[32]u16`, so names are **truncated to 31 UTF-16 units**. |
| Output | `midiOutOpen(&h, id, 0, 0, CALLBACK_NULL)`. Use `midiOutShortMsg(h, packed_u32)` for 1–3-byte messages (status in the low byte). For SysEx: `midiOutPrepareHeader` → `midiOutLongMsg` → wait for `MHDR_DONE` → `midiOutUnprepareHeader`. |
| Input | `midiInOpen(&h, id, callback, instance, CALLBACK_FUNCTION)`; `midiInPrepareHeader` + `midiInAddBuffer` for N SysEx buffers; `midiInStart`. Callback messages: `MIM_DATA` (packed short message in `dwParam1`), `MIM_LONGDATA` (`*MIDIHDR`, `dwBytesRecorded`), `MIM_LONGERROR`. |

Notes:
- A SysEx longer than one input buffer arrives as several `MIM_LONGDATA` callbacks. Size each
  buffer at 2 KiB or more so a program dump (about 1.2 KB) usually fits in one. Still rely on
  the assembler for correctness.
- MS docs forbid calling most multimedia functions from the callback. Push bytes into the ring
  buffer and recycle `MIDIHDR`s (`midiInAddBuffer`) from the polling thread, not the callback.
- `MIM_DATA` short messages must be unpacked back to bytes, and their length derived from the
  status byte. Reuse the core's `message_length(status)` table.
- WinMM ports are typically single-client. A port already held by a DAW fails with
  `MMSYSERR_ALLOCATED`.

### Input handoff

```zig
// sketch — single-producer (OS thread) / single-consumer (app thread) byte ring
pub fn RingBuffer(comptime capacity: u32) type {
    comptime std.debug.assert(std.math.isPowerOfTwo(capacity));
    return struct {
        bytes: [capacity]u8 = undefined,
        head: std.atomic.Value(u32) = .init(0), // written by producer only
        tail: std.atomic.Value(u32) = .init(0), // written by consumer only
        overflow_count: std.atomic.Value(u32) = .init(0),
        // push: never blocks, never allocates; on full → drop chunk, bump overflow_count.
        // drain: copies min(available, out.len) bytes, returns count.
    };
}
```

- Capacity: at least 4× the largest expected inbound message, which is a user-slot data dump if
  supported (see [memory budget](memory-budget.md)), otherwise the about 1.2 KB program dump.
  16 KiB is a safe default.
- Drop whole chunks, never partial ones. A dropped chunk mid-SysEx then shows up as a framing
  error in the assembler (missing `F7` or a new `F0`), which reports it, instead of silently
  corrupting data.
- The consumer runs on the application's loop: `drain()` → `assembler.feed()` →
  `transaction.on_bytes()`. Timeouts are evaluated from `tick(now_ns)`, not from blocking reads.

## Invariants

- The backend never parses MIDI. It only moves bytes. (The WinMM `MIM_DATA` unpacking is the one
  exception, and it uses the core's length table.)
- No allocation after `open`. Port name buffers, packet lists, `MIDIHDR`s, and the ring are all
  fixed-size.
- Every OS loop is bounded: at most 32 cards, at most 64 ports per direction, and at most
  `numPackets` packets per CoreMIDI callback.
- `@sizeOf` of every mirrored OS struct is `comptime`-asserted against the header value.

## Errata & gotchas

- Port names differ per OS and driver. The Rust matcher only knows the macOS-style names
  (`"minilogue xd SOUND"`, …). On Windows the class-compliant driver may present generic names
  such as `"minilogue xd"` / `"MIDIIN2 (minilogue xd)"`, while the KORG USB-MIDI driver uses
  others. On Linux the rawmidi `subname` comes from snd-usb-audio and depends on the kernel
  version. **None of this is verified in the repo.** Keep the matcher table-driven and log all
  port names on failure.
- midir's input path allocates per message and its channel is unbounded. A flood of clock
  (`F8`) bytes at 24 PPQN grows memory without limit if nobody calls `receive`. The bounded
  ring removes this failure mode.
- SysEx output on DIN MIDI OUT takes about 0.32 ms/byte (31250 baud), so a 1171-byte payload
  takes roughly 377 ms. USB is much faster. Do not start a transaction timeout until the write
  completes. On WinMM that means `MHDR_DONE`.

## Zig / TigerStyle notes

- One file per backend: `io/coremidi.zig`, `io/alsa_rawmidi.zig`, `io/winmm.zig`, plus
  `io/mock.zig`. Select with `switch (builtin.os.tag)` at comptime. See
  [module layout](module-layout.md).
- The backend interface is a plain struct with `open`, `close`, `write`, `drain`, and
  `enumerate`. Use comptime duck typing instead of a vtable, since only one backend exists per
  build. The mock records writes into a fixed array and scripts input chunks with exact
  fragmentation boundaries. See [test plan](test-plan.md).
- Keep threads to a minimum. CoreMIDI and WinMM already own their callback threads. Linux needs
  one reader thread, or none if the app's loop polls the fd with timeout 0.

## Open questions

- Exact XD port names on Linux (rawmidi `subname`) and on Windows (class driver vs KORG driver).
- Is the legacy CoreMIDI byte API enough long-term, or should the port target UMP
  (`MIDIEventList`)?
- Should Linux use the sequencer (multi-client) instead of rawmidi (exclusive)? This depends on
  whether a DAW will run alongside the tool.

## Sources

- `crates/minilogue-xd/src/transport.rs:136-330` — midir output and input backends, `mpsc` handoff.
- `crates/minilogue-xd/src/device.rs:43-130` — port name patterns, client and port names.
- `crates/minilogue-xd/src/device.rs:141-260` — enumeration and `find_*` fallback logic.
- Linux uapi `sound/asound.h` (bundled with Zig 0.16): `snd_rawmidi_info`,
  `SNDRV_CTL_IOCTL_RAWMIDI_*`, `SNDRV_RAWMIDI_IOCTL_INFO`.
- Zig 0.16 `std/os/linux/ioctl.zig` — `IOR`/`IOW`/`IOWR` helpers.
