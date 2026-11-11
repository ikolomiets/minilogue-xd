---
title: Transport — MidiOutput/MidiInput, mocks, midir backend, connection
category: design
sources:
  - crates/minilogue-xd/src/transport.rs
  - crates/minilogue-xd/src/connection.rs
  - crates/minilogue-xd/src/device.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/src/lib.rs
  - crates/minilogue-xd/Cargo.toml
  - crates/minilogue-xd/examples/list_ports.rs
  - crates/minilogue-xd/examples/test_drive.rs
related:
  - protocol/usb-ports.md
  - protocol/midi-messages.md
  - design/sysex-transactions.md
  - design/realtime-controller.md
  - design/errors.md
  - design/testing-strategy.md
  - protocol/cc-map.md
  - zig/platform-midi.md
  - zig/memory-budget.md
  - zig/dependency-elimination.md
updated: 2026-11-11
---

# Transport — MidiOutput/MidiInput, mocks, midir backend, connection

> The crate uses two minimal traits for MIDI I/O:
>
> - `MidiOutput::send(&[u8])`
> - `MidiInput::receive(timeout) -> Option<Vec<u8>>`
>
> Two in-memory test doubles implement them (`MockOutput`, `MockMidiInput`). Two
> `midir`-backed types also implement them (`MidirOutput`, `MidirInput`). On input, midir's
> callback thread pushes each message into an **unbounded** `mpsc` channel, and `receive`
> blocks on that channel until a timeout. `MinilogueXd` is a thin wrapper that sends typed
> messages. A Zig port keeps the "send bytes" and "poll for bytes" contract. It replaces the
> callback and channel with a fixed-capacity ring buffer, which the program drains at its own
> pace into a byte-stream assembler.

## Facts

### Trait contract (`transport.rs:13-35`) [rust]

| Trait | Method | Semantics |
|---|---|---|
| `MidiOutput: Send` | `fn send(&mut self, bytes: &[u8]) -> Result<()>` | Send **one complete MIDI message**: a 1–3 byte channel or system message, or a whole `F0 … F7` SysEx. `Err` means the backend failed. There is no flush or completion signal, and no ordering guarantee beyond the backend's. |
| `MidiInput: Send` | `fn receive(&mut self, timeout: Duration) -> Result<Option<Vec<u8>>>` | Wait up to `timeout` for the next message. `Ok(Some(bytes))` is one backend-delivered message. `Ok(None)` means timeout. `Err` is an I/O failure, such as disconnection. |

- Both traits are `Send` but not `Sync`. `&mut self` gives one user at a time. They are always
  compiled, with no feature gate, and use `std::time::Duration`.
- "One complete message per `receive`" is the **intended** contract. Real backends break it.
  See the backend table below.
- Consumers are all statically generic. None uses `dyn`.

| Consumer | Bound | Ownership |
|---|---|---|
| `MinilogueXd<O>` (`connection.rs:16-19`) | `O: MidiOutput` | Owns `O` |
| `RealtimeController<O>` (`controller.rs:84-89`) | `O: MidiOutput` | Owns `O` |
| `SysexTransaction<'a, O, I>` (`sysex/transaction.rs:55-60`) | `O: MidiOutput, I: MidiInput` | `&'a mut` borrows of both, plus `channel: U4` and `timeout: Duration` (default 5 s, line 48) |

### Test doubles (always compiled, public) [rust]

| Type | State | API | Behaviour |
|---|---|---|---|
| `MockOutput` (`transport.rs:37-81`) | `messages: Vec<Vec<u8>>` | `new`, `default`, `messages() -> &[Vec<u8>]`, `last_message() -> Option<&[u8]>`, `clear()` | `send` copies `bytes` into a new `Vec` and appends it. It never fails. |
| `MockMidiInput` (`transport.rs:83-133`) | `responses: VecDeque<Vec<u8>>` | `new`, `default`, `queue_response(Vec<u8>)`, `queue_responses(impl IntoIterator)`, `remaining() -> usize` | `receive` **ignores `timeout`** and pops the front. An empty queue returns `Ok(None)` immediately. It never fails. |

A test-only `FailingOutput` returns `Err(InvalidMessage("simulated send failure"))` from every
`send` (`sysex/transaction.rs:362-376`).

### midir backend (`cfg(feature = "midi-io")`) [rust]

**`MidirOutput`** (`transport.rs:136-219`)

```text
connect(pattern):
  client = midir::MidiOutput::new("minilogue-xd")          ; Err → MidiIo(os message)
  port   = first port whose name contains pattern            ; none → MidiIo("no MIDI output port matching '<pattern>'")
  conn   = client.connect(port, "minilogue-xd-out")          ; Err → MidiIo
send(bytes):     conn.send(bytes)  ; Err → MidiIo
available_ports: names of all output ports (name errors skipped)
```

**`MidirInput`** (`transport.rs:221-330`)

```text
connect(pattern):
  client = midir::MidiInput::new("minilogue-xd")
  port   = first port whose name contains pattern            ; none → MidiIo("no MIDI input port matching '<pattern>'")
  (tx, rx) = std::sync::mpsc::channel()                      ; UNBOUNDED
  conn = client.connect(port, "minilogue-xd-in",
           callback(timestamp, message) = { tx.send(message.to_vec()) ; error ignored }, ())
  keep conn (dropping it closes the port), keep rx
receive(timeout):
  rx.recv_timeout(timeout):
    message      → Ok(Some(bytes))
    Timeout      → Ok(None)
    Disconnected → Err(MidiIo("MIDI input disconnected"))
```

- The timestamp is discarded.
- `ignore()` is never called. midir's default is `Ignore::None`, so **SysEx, timing clock
  (`F8`), MTC and active sensing (`FE`) are all delivered**.
- Each callback allocates one `Vec` on the OS MIDI thread.

### How bytes arrive, per midir 0.10.3 backend [rust, from midir source]

| OS backend | Short messages | SysEx | Real-time inside SysEx |
|---|---|---|---|
| CoreMIDI (macOS) | Each packet is split into messages by status length. No running-status reconstruction. | Segmented packets are reassembled until the last byte is `F7`. Then one callback runs. | Can be **embedded** in the reassembled buffer. The packet tail is copied verbatim. |
| ALSA seq (Linux) | One decoded event per callback | The sequencer delivers 256-byte chunks. midir concatenates them until `F7`, then calls back once. | Delivered as separate events |
| WinMM (Windows) | Unpacked from `MIM_DATA`. Length is derived from the status byte. | 4 buffers × **1024 bytes**. A SysEx longer than 1024 bytes arrives as **several callbacks**. midir's own source has a TODO about this. | Separate `MIM_DATA` |

Consequences:

- A current-program dump (1179 bytes) reaches `receive` as **two pieces on Windows**.
- On macOS it may contain stray `F8`/`FE` bytes.
- `SysexTransaction::receive_sysex` handles both cases. It accumulates from `F0` until `F7`,
  drops bytes `> 0xF7` and skips non-SysEx messages while idle (`sysex/transaction.rs:103-142`).
  See [SysEx transactions](sysex-transactions.md).

### `MinilogueXd` wrapper (`connection.rs`) [rust]

| Item | Signature | Notes |
|---|---|---|
| struct | `MinilogueXd<O: MidiOutput> { output: O, channel: U4 }` | `connection.rs:16-19` |
| `new` | `(output: O, channel: U4) -> Self` | |
| `send_message` | `(&mut self, msg: &MidiMessage) -> Result<()>` | `output.send(&msg.to_midi_bytes())`. **The stored `channel` is not used.** Each message carries its own channel. |
| `channel` | `(&self) -> U4` | |
| `output` / `output_mut` | `&O` / `&mut O` | Lets tests inspect a `MockOutput` |
| `connect` (midi-io) | `(port_name: &str, channel: U4) -> Result<MinilogueXd<MidirOutput>>` | `MidirOutput::connect(port_name)` then `new` (`connection.rs:53-67`) |

The phase-1 plan had a callback-driven input,
`MidiInput::listen<F: FnMut(&[u8]) + Send + 'static>(callback)`. It also had a
`MinilogueXd<O = Box<dyn MidiOutput>>` default (phase-1 plan lines 175, 191). The final code
uses static generics and poll-with-timeout instead.

### Device discovery and `cc_name` (`device.rs`)

- `list_output_ports`, `list_input_ports`, `find_output(OutputPort)`, `find_input(InputPort)`,
  `find_output_port()` (SOUND) and `find_input_port()` (KBD/KNOB) live in `device.rs`. They
  are described in [USB ports](../protocol/usb-ports.md). Each call creates a fresh midir
  client.
- `cc_name(cc: u8) -> Option<&'static str>` (`device.rs:260-339`) maps 64 CC numbers to
  display names. The doc comment says 63. It is used for track and text labels in MIDI-file
  export (`midi_file.rs:851`). The authoritative CC table is in [CC map](../protocol/cc-map.md).

### Throughput and timing facts

| Path | Rate | Program dump (1179 bytes) |
|---|---|---|
| 5-pin DIN, 31250 baud, 10 bits/byte | 3125 bytes/s, 0.32 ms/byte | ≈ 377 ms [spec MIDI 1.0] |
| USB MIDI 1.0 class (4-byte event packets, 3 data bytes each) | Much faster than DIN, but host- and device-dependent | Not measured in the repo |
| Default transaction timeout | 5 s (`sysex/transaction.rs:48`) | — |
| Timing clock from XD (24 PPQN) | 48 msg/s at 120 BPM, 120 msg/s at 300 BPM | — |

## Invariants

- `send` receives exactly one complete message. Its first byte is a status byte
  (`>= 0x80`). For SysEx, `bytes[0] == 0xF0` and `bytes[len-1] == 0xF7`.
- Callers never pass an empty slice to `send`. Nothing enforces this in Rust; a Zig port
  should assert it.
- `receive` returns `Ok(None)` only after waiting at least `timeout` (midir). The mock returns
  `Ok(None)` immediately, so code must not rely on `receive` taking time.
- After `Err(MidiIo("MIDI input disconnected"))`, every later `receive` fails the same way.
  That happens once the sender is dropped, i.e. when the connection closes.
- Messages from one input port are delivered in arrival order. `mpsc` and `VecDeque` are both
  FIFO.

## Errata & gotchas

- **Unbounded input queue** [rust]. Nobody filters `F8` or `FE`. If an application holds a
  `MidirInput` and doesn't call `receive`, memory grows without limit, at a rate of up to
  120 clock messages/s plus active sensing. There is also one heap allocation per message on
  the OS real-time thread.
- **The contract is broken on Windows** for SysEx longer than 1024 bytes, which arrives in
  fragments. On macOS, embedded real-time bytes can appear. Any consumer other than
  `SysexTransaction` must reassemble and filter itself.
- **The mock cannot model timing or fragmentation boundaries** beyond what a test scripts by
  hand. Fragmentation tests exist only because the transaction tests queue split pieces
  manually. See [testing strategy](testing-strategy.md).
- **`MinilogueXd.channel` is unused** by `send_message`. It is purely informational.
- **Substring port matching takes the first hit** (`transport.rs:163-171, 255-264`). With two
  XDs attached, the first one wins.
- **Error detail is lost.** midir errors are flattened into `MidiIo(String)`. See
  [errors](errors.md).
- **The `std` feature does nothing.** `transport.rs` unconditionally uses `std::collections`,
  `std::time` and `std::sync::mpsc`, so `no_std` is not possible (`Cargo.toml:12-16`).

## Test vectors

From `crates/minilogue-xd/src/transport.rs:332-414` and
`crates/minilogue-xd/src/connection.rs:69-121`:

| Scenario | Input | Expected |
|---|---|---|
| Mock output records in order | `send(90 3C 64)`, `send(80 3C 00)` | `messages() == [[90 3C 64], [80 3C 00]]` |
| Last message | empty, then `send(F8)` | `None`, then `Some([F8])` |
| Clear | `send(90 3C 64)`, `clear()` | `messages()` empty, `last_message() == None` |
| Mock input FIFO | queue `F0 01 F7`, `F0 02 F7` | `receive → Some(F0 01 F7)`, then `Some(F0 02 F7)` |
| Mock input empty | — | `receive(1 s) → Ok(None)` (immediately) |
| Batch queue | `queue_responses([01],[02],[03])` | `remaining() == 3`, after one `receive` it is `2` |
| `MinilogueXd::send_message` | NoteOn ch 0, key 60, vel 100 | `90 3C 64` |
| Two messages | NoteOn as above + `TimingClock` | `[90 3C 64]`, `[F8]` |
| Channel getter | `new(mock, U4(5))` | `channel().value() == 5` |

## Rust design

- These parts are **essential**:
  - The byte-level seam: `send(bytes)` and `receive(timeout) → bytes?`.
  - Keeping protocol logic (framing, reassembly, parsing) out of the backend.
  - Being able to substitute a mock for hardware.
- These parts are **incidental**:
  - Trait objects vs generics.
  - `Vec<u8>` per message.
  - The `mpsc` channel.
  - Blocking `recv_timeout`.
  - Having midir own a callback thread.
- Allocation: one `Vec` per message received, one per mock send, and `String`s on errors.
  Output does not allocate in the crate. midir may copy internally, e.g. into CoreMIDI packet
  lists.
- Concurrency: the only thread is midir's input callback thread. Everything else is
  single-threaded and blocking. Library code uses no async, locks or atomics.
- Timeouts appear in two places. `receive` takes a per-call timeout. `SysexTransaction` keeps a
  per-transaction deadline using `Instant::now()` (`sysex/transaction.rs:104-110`). These are
  the only clock reads in the library.

## Zig / TigerStyle notes

**Interface (sketch).** Use comptime duck typing with one backend per build, as described in
[platform MIDI](../zig/platform-midi.md):

```zig
// sketch
pub const TransportError = error{ PortNotFound, PortBusy, PortOpenFailed, WriteFailed, Disconnected };
// Output: whole message per call; caller asserts framing.
pub fn write(self: *Output, bytes: []const u8) TransportError!void;
// Input: non-blocking; copies up to out.len queued bytes, returns count.
pub fn drain(self: *Input, out: []u8) u32;
```

**Input pipeline.** Use no callbacks and no blocking reads. The program polls on its own loop:

1. The OS callback thread (CoreMIDI or WinMM) or a Linux reader pushes raw bytes into an SPSC
   ring buffer. The ring is fixed capacity, a power of two, and uses only atomics: no locks
   and no allocation. When full, it drops the whole chunk and increments `overflow_count`.
   Default capacity is **16 KiB**, which is ≥ 4× the largest expected inbound SysEx. See
   [memory budget](../zig/memory-budget.md).
2. The application loop calls `drain(&scratch)` and feeds the bytes to a **stream
   assembler**, which is part of the sans-I/O core.
3. The assembler emits complete messages into fixed buffers. The transaction state machine
   consumes them, and `tick(now_ns)` evaluates deadlines.

**Stream assembler (sketch of the state machine).** It replaces both midir's reassembly and
`receive_sysex`:

```text
state: idle | channel(expected_len, have) | sysex(len)
feed(byte):
  if byte >= 0xF8:                        ; real-time: emit immediately, any state, never stored
      emit_realtime(byte); return
  if byte == 0xF0:
      if state == sysex: count sysex_unterminated (or error)   ; restart
      state = sysex(0); push F0
  elif byte == 0xF7:
      if state == sysex: push F7; emit_sysex(buf[0..len]); state = idle
      else: count stray_eox
  elif byte >= 0x80:                      ; 0x80..0xEF, 0xF1..0xF6
      if state == sysex: count sysex_unterminated; state = idle (then handle as a new status)
      start channel/common message with expected_len = length_table[byte]
  else:                                    ; data byte
      if state == sysex: if len == frame_size_max: count sysex_overflow, state = idle; else push
      elif state == channel: push; if complete: emit; state = idle (or keep status for running status)
      else: count orphan_data_byte           ; Rust never sees running status (midir CoreMIDI/WinMM don't produce it)
```

- Size the SysEx buffer `frame_size_max` to at least the largest accepted SysEx. That is **1181**
  bytes for a stored program dump (1171 encoded + 7 header + 2 program-number bytes + `F7`), or
  the user-slot maximum if [user modules](../protocol/user-modules.md) are supported. The
  [memory budget](../zig/memory-budget.md) rounds this up to 2048 (or 48 KiB).
- Every loop is bounded by `out.len` or the ring capacity.
- Assert that `len <= frame_size_max` on every push.
- The "drop real-time inside SysEx" rule is the same as Rust's `filter(|b| b <= 0xF7)`. The
  difference is that real-time bytes are surfaced as events instead of being discarded.

**Time.** Read the platform monotonic clock (`mach_absolute_time` on macOS, `CLOCK_MONOTONIC`
on Linux, `QueryPerformanceCounter` on Windows). Pass `now_ns: u64` into the core, and never
read the clock inside parsing code.

- Name values with units last: `timeout_ns`, `deadline_ns`.
- Default `transaction_timeout_ns = 5 * std.time.ns_per_s`.
- Start a transaction's deadline only **after** `write` returns. DIN output of a dump takes
  ≈ 377 ms.

**Mocks.** Use fixed arrays, not `Vec`s:

- `MockOutput` keeps `messages: [messages_max][message_bytes_max]u8` and `lengths:
  [messages_max]u16`. Suggested sizes are `messages_max = 64` and `message_bytes_max = 1181`.
- `MockInput` replays scripted **chunks**, given as a slice of slices, so tests can split a
  SysEx at any byte and insert `F8`. Rust's mock can't model that without hand-written
  pieces.

**Threads.** Write from a single thread. On input, the producer is the OS thread and the
consumer is the app thread. They share the ring's `head` and `tail` atomics only, with
release on publish and acquire on consume. Don't call back into user code from the OS
thread.

**Wrapper.** `MinilogueXd` disappears. Callers encode a message into a stack `[3]u8` (see
[MIDI messages](../protocol/midi-messages.md)) and call `write`. If a "default channel"
convenience is kept, use it. Don't store a field that nothing reads.

**`cc_name`.** Use a comptime `[128]?[]const u8` table generated from the
[CC map](../protocol/cc-map.md). Assert its count (64) in a test.

## Open questions

- How does the XD pace SysEx on its USB IN endpoint? Can a 16 KiB ring overflow during an
  "All Dump" (≈ 20 s, `book.md:2065`) if the app polls slowly? The answer depends on the poll
  period. Measure it.
- Does CoreMIDI really embed `F8`/`FE` inside reassembled SysEx from the XD, or is it only
  theoretically possible? The Rust filter was added defensively.
- Should the Zig port support running status on input (`orphan_data_byte`)? No backend that
  Rust uses produces it, but raw ALSA rawmidi or DIN-sourced streams might.

## Sources

- `crates/minilogue-xd/src/transport.rs:13-35` — traits.
- `crates/minilogue-xd/src/transport.rs:37-133` — mocks.
- `crates/minilogue-xd/src/transport.rs:136-219` — `MidirOutput`.
- `crates/minilogue-xd/src/transport.rs:221-330` — `MidirInput`.
- `crates/minilogue-xd/src/transport.rs:332-414` — tests.
- `crates/minilogue-xd/src/connection.rs:1-121` — `MinilogueXd` and its tests.
- `crates/minilogue-xd/src/sysex/transaction.rs:48, 55-60, 103-142, 362-376` — timeout,
  borrowing, reassembly, `FailingOutput`.
- `crates/minilogue-xd/src/controller.rs:84-89` — controller generic over `MidiOutput`.
- `crates/minilogue-xd/src/device.rs:116-130, 260-339` — client and connection names,
  `cc_name`.
- `crates/minilogue-xd/src/midi_file.rs:851` — `cc_name` use.
- `crates/minilogue-xd/Cargo.toml:12-23`, `crates/minilogue-xd/src/lib.rs:65-94` — features.
- midir 0.10.3 source (`backend/coremidi/mod.rs:84-180`, `backend/alsa/mod.rs:761-912`,
  `backend/winmm/mod.rs:42-43`, `backend/winmm/handler.rs:40-120`) — reassembly and
  fragmentation behaviour, default `Ignore::None`.
- `docs/dev/0001-phase-1-implementation-plan-minilogue-xd-foundation.md` — the original
  `Box<dyn>` / `listen()` design.
- `docs/korg-user-guide/book.md:2065-2067` — All Dump over USB.
