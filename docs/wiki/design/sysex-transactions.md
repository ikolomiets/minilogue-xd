---
title: SysEx transactions — request/response, framing, timeouts
category: design
sources:
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/src/sysex/frame.rs
  - crates/minilogue-xd/src/sysex/program/mod.rs
  - crates/minilogue-xd/src/transport.rs
  - crates/minilogue-xd/src/error.rs
  - crates/minilogue-xd/tests/sysex_flows.rs
  - crates/minilogue-xd/examples/read_patch.rs
  - crates/minilogue-xd/examples/read_edit_buffer.rs
  - crates/minilogue-xd/examples/quintal.rs
  - crates/minilogue-xd/examples/sysex_roundtrip_diag.rs
related:
  - protocol/sysex-framing.md
  - protocol/device-identity.md
  - protocol/tuning.md
  - protocol/program-blob.md
  - protocol/global-blob.md
  - protocol/user-modules.md
  - protocol/usb-ports.md
  - protocol/midi-messages.md
  - design/transport.md
  - design/errors.md
  - design/byte-fidelity.md
  - zig/platform-midi.md
  - zig/memory-budget.md
  - zig/tigerstyle-mapping.md
  - zig/test-plan.md
updated: 2026-11-11
---

# SysEx transactions — request/response, framing, timeouts

> `SysexTransaction` performs one conversation at a time:
> 1. Send a Korg exclusive request or dump.
> 2. Wait, within a single deadline (default **5 s**), for the next complete `F0 … F7` frame on the
>    input port.
> 3. Classify the frame as the expected dump, an ACK (`0x23`), a NAK (`0x24`, `0x26`–`0x2F`) or
>    something unexpected.
>
> Its receive loop, `receive_sysex`, was hardened on real hardware in four commits within 15
> minutes. It now:
> - skips non-SysEx chunks;
> - reassembles fragmented frames;
> - strips realtime bytes `0xF8`–`0xFF`.
>
> It still classifies by **chunk** instead of by byte, has no size cap, and does not match the
> reply's header, channel or function before accepting it. A port should use a byte-level, bounded,
> allocation-free assembler. It should be polled with an explicit nanosecond deadline and a bounded
> poll count.

## Wiring and roles

- Over USB the host **sends on the SOUND port** and **listens on the KBD/KNOB port** [hw]. This was
  found during hardware bring-up (commit `bc6c452`; `crates/minilogue-xd/examples/read_patch.rs:24-37`).
  See [USB ports](../protocol/usb-ports.md).
- Only **one request is outstanding** at a time. There is no pipelining and no request ID: the
  Korg protocol has no correlation field except the function ID and the `g` nibble.
- The device sends MIDI clock (`F8`, 24 ppqn) and active sensing (`FE`) on the same input. These
  bytes arrived before the reply [hw] (commit `bc6c452`). Commit `5e4b4f0` suspects the driver
  can also embed them *inside* a SysEx chunk; that was inferred, not captured.

## `SysexTransaction` structure

| Field | Type | Meaning |
|-------|------|---------|
| `output` | `&'a mut O` (`O: MidiOutput`) | exclusive borrow of the send port |
| `input` | `&'a mut I` (`I: MidiInput`) | exclusive borrow of the receive port |
| `channel` | `U4` | global channel `g` written into every request header |
| `timeout` | `Duration` | per-operation receive budget; default `DEFAULT_TIMEOUT = 5 s` |

- `new(output, input, channel)` sets the default timeout.
- `with_timeout(self, d)` consumes and returns `self`, builder style.
- `timeout()` and `channel()` are accessors.
- See `crates/minilogue-xd/src/sysex/transaction.rs:47-87`.

## Operations

| Method | Sends (fn, bytes) | Helper | Accepts | Decodes with | [hw] verified |
|--------|-------------------|--------|---------|--------------|:-------------:|
| `request_current_program` | `0x10`, 8 | `request_response` | `0x40` | `ProgramData::from_bytes(frame.data)` | yes |
| `send_current_program` | `0x40`, 1179 | `send_and_wait_ack` | any non-error status | — | yes |
| `request_program(n)` | `0x1C lsb msb`, 10 | `raw_request_response` | anything not an 8-byte NAK | `parse_program_dump(raw)`; **returned program number discarded** | yes |
| `send_program(n)` | `0x4C`, 1181 | `send_and_wait_ack` | non-error status | — | yes |
| `request_global` | `0x0E`, 8 | `request_response` | `0x51` | `GlobalParams::from_bytes` | no |
| `send_global` | `0x51`, 80 | `send_and_wait_ack` | non-error status | — | no |
| `request_user_scale` | `0x14`, 8 (spec 9) | `request_response` | `0x44` | `UserScale::from_bytes` | no |
| `send_user_scale` | `0x44`, 447 | `send_and_wait_ack` | non-error status | — | no |
| `request_user_octave` | `0x15`, 8 (spec 9) | `request_response` | `0x45` | `UserOctave::from_bytes` | no |
| `send_user_octave` | `0x45`, 50 | `send_and_wait_ack` | non-error status | — | no |
| `query_identity` | `F0 7E 0g 06 01 F7`, 6 | `receive_sysex` only | any frame | `identity::parse_identity_reply` | no |

Sources: `transaction.rs:218-358`. The read path was verified in `read_patch` and
`read_edit_buffer`; the write path in `quintal` (`examples/quintal.rs:608-632`) and commit
`1c16566`.

Notes on the table:
- The tuning payload shapes are disputed; see [tuning](../protocol/tuning.md).
- Search device, user-module operations (`0x17`–`0x1E`), MTS and poly chain have **no** transaction
  wrapper. See [user modules](../protocol/user-modules.md) and
  [device identity](../protocol/device-identity.md).

## `receive_sysex` (current Rust algorithm)

`transaction.rs:103-142`:

```text
deadline = now() + timeout                          # monotonic Instant, set AFTER send returned
buf = []
loop:
    remaining = saturating(deadline - now())
    if remaining == 0: return Timeout(timeout)      # reports configured, not elapsed, time
    chunk = input.receive(remaining)?               # Ok(None) on timeout; Err(MidiIo) on disconnect
    if chunk is None: return Timeout(timeout)
    if chunk[0] == F0 and buf is empty:             # classified by the chunk's FIRST byte
        clean = [b for b in chunk if b <= F7]
        if clean[-1] == F7: return clean            # whole frame in one chunk
        buf = clean; continue                       # start of fragmented frame
    if buf not empty:
        buf += [b for b in chunk if b <= F7]        # any bytes, including F0 and 80..F6
        if buf[-1] == F7: return buf
        continue
    # buf empty and chunk does not start with F0: drop the whole chunk
```

## Response classification

| Response (first complete frame) | `request_response` (`transaction.rs:144-162`) | `raw_request_response` (`169-188`) | `send_and_wait_ack` (`191-206`) |
|---------------------------------|------------------------------------------------|------------------------------------|---------------------------------|
| Expected dump | `Ok(frame)` (payload NOTE-1 decoded) | `Ok(raw bytes)` | `UnexpectedResponse(fid)` |
| ACK `0x23` | `UnexpectedResponse(0x23)` | `Ok(raw)`, then `parse_program_dump` fails with `InvalidHeader("message too short")` | **`Ok(())`** |
| NAK `0x24`, `0x26`–`0x2F` | `NakReceived(status)` | `NakReceived` **only if `len == 8`** | `NakReceived(status)` |
| Undefined status `0x25` / other fid | `UnexpectedResponse(fid)` | `Ok(raw)`, then a downstream parse error | `UnexpectedResponse(fid)` |
| Non-Korg or other-model SysEx | `InvalidHeader` from `parse_sysex` | `Ok(raw)`, then `InvalidHeader` | `InvalidHeader` |
| Malformed 7-bit payload | `Codec` error | n/a (raw) | `Codec` error |
| No frame before deadline | `Timeout(timeout)` | same | same |
| Input disconnected | `MidiIo("MIDI input disconnected")` | same | same |
| `send` fails | output's error, returned before waiting | same | same |

- The status set and `is_error` are defined in [SysEx framing](../protocol/sysex-framing.md).
- The **first** complete frame decides the outcome. Unrelated SysEx is never skipped.

## Timeouts

- The default is **5 s** (`transaction.rs:48`, test `transaction.rs:870-876`). All examples set
  5 s explicitly (`examples/read_patch.rs:40-41`, `examples/read_edit_buffer.rs:179-180`,
  `examples/quintal.rs:608-609`).
- Tests use 10 ms (`tests/sysex_flows.rs:84-97`) and 500 ms / 10 s for the accessors
  (`transaction.rs:879-885`, `tests/sysex_flows.rs:100-108`).
- There is **one deadline per operation**, measured with the monotonic `Instant` from the start of
  `receive_sysex`, i.e. after `output.send` has returned. Every `receive` call gets the remaining
  time, so N chunks cannot extend the wait.
- `MockMidiInput::receive` ignores its timeout and returns `None` as soon as its queue is empty
  (`crates/minilogue-xd/src/transport.rs:129-132`). Timeout tests therefore finish instantly.
- `MidirInput::receive` uses `mpsc::Receiver::recv_timeout(remaining)`. A timeout gives `None`; a
  disconnect gives `MidiIo` (`transport.rs:309-320`).
- Wire-time budget [spec MIDI 1.0]:
  - 5-pin DIN runs at 31 250 baud with 10 bits per byte, i.e. **0.32 ms per byte**.
  - The largest fixed frame (`0x4C`, 1181 bytes) therefore takes about **378 ms** each way over DIN.
  - USB is far faster.
  - The device's own processing time (for example a flash write before the ACK) is unmeasured.

## Hardening history (lessons learned)

All four commits landed on 2026-04-07 during the first hardware session:

| Commit | Time | Symptom on hardware | Change |
|--------|------|---------------------|--------|
| `bc6c452` | 00:27 | The XD sends `F8` clock at 24 ppqn. Clock arrived **before** the reply and was taken as the response | Added `receive_sysex`: loop until a chunk whose first byte is `F0`. Also found the SOUND-out / KBD/KNOB-in port mapping |
| `685ea64` | 00:38 | Partial fragments (`F0` without `F7`), or chunks with embedded realtime bytes, were accepted and caused codec errors downstream | Require `first == F0 && last == F7`; skip everything else. With fragmented delivery this can only **time out** |
| `d465474` | 00:40 | "1181 bytes for program dumps … delivered … in multiple pieces on some platforms"; intermittent codec errors | Accumulate fragments; see below |
| `5e4b4f0` | 00:42 | Realtime bytes "embedded within SysEx messages by the MIDI driver"; remaining intermittent codec errors | Strip bytes `> 0xF7` in both the single-chunk and the fragment path; this produced the current code |

How `d465474` accumulated fragments:
- A complete `F0 … F7` chunk is returned only when the buffer is empty.
- An `F0` chunk **not** ending in `F7` **resets** the buffer, which is correct restart behavior.
- Continuations are appended **unfiltered**, even though the code comment says realtime is
  filtered.

`5e4b4f0` added the filtering but tied the `F0` test to `buf.is_empty()`. An `F0` chunk arriving
mid-frame is now **appended instead of restarting** the buffer, a regression from `d465474`.

`1c16566` (17:17 the same day) fixed a different failure: writes NAKed with `0x26` DATA FORMAT
ERROR. The cause was **payload content** bugs, not framing; see
[byte fidelity](byte-fidelity.md). A NAK means the frame arrived intact.

Lessons for a port:

1. **Chunk boundaries are meaningless.**
   - A transport chunk may hold part of a frame, several frames, or frame plus other messages.
   - Process the stream one byte at a time.
2. **Realtime bytes (`F8`–`FF`) may appear anywhere,** including inside SysEx (MIDI 1.0 allows
   it). Drop them at byte level and never let them change state.
3. **Any other status byte terminates SysEx** (MIDI 1.0):
   - `F0` restarts the frame.
   - `0x80`–`0xEF` and `0xF1`–`0xF6` abort it.
4. **Bound everything.** That covers buffer size, deadline, number of polls, and number of
   unrelated frames tolerated.
5. **Write the assembler with its test vectors first.** Four fixes in 15 minutes, with no
   fragmentation or realtime test anywhere in the suite, is the failure mode to avoid.

## Known weaknesses of the Rust implementation

| # | Weakness | Citation | Consequence |
|---|----------|----------|-------------|
| 1 | Classification is by first byte of chunk | `transaction.rs:121-129`, `140-141` | `F8 F0 42 …` in one chunk is dropped entirely while idle, so the operation times out |
| 2 | `F0` mid-frame is appended, not a restart | `transaction.rs:121`, `132-133` | Concatenated frames, then `InvalidHeader`/`Codec` errors |
| 3 | `0x80`–`0xF6` mid-frame are appended | `transaction.rs:122`, `133` | Corrupt payload instead of a clean abort |
| 4 | Trailing bytes after `F7` in the same chunk | `transaction.rs:122-128` | `… F7 90 3C 64`: the last byte is not `F7`, so the chunk is treated as a fragment start; the next frame is glued on |
| 5 | No buffer cap | `transaction.rs:105`, `133` | Unbounded growth until the deadline if `F7` is lost |
| 6 | No header, channel or function match before accepting | `transaction.rs:144-162` | A stale dump, a panel-triggered dump, an identity reply or another device's SysEx becomes the response and fails the operation |
| 7 | No input drain before `send`; `mpsc` channel is unbounded | `transaction.rs:145`, `transport.rs:266` | Messages queued between transactions are consumed as replies |
| 8 | `Timeout` carries the configured duration | `transaction.rs:110`, `115` | Elapsed time is not reported |
| 9 | ACK to a request becomes `UnexpectedResponse(0x23)`; NAK detection on the raw path requires `len == 8` | `transaction.rs:151-159`, `177-185` | Asymmetric error mapping |
| 10 | `request_program` discards the program number in the reply | `transaction.rs:242-247` | Cannot detect a reply for the wrong slot |
| 11 | Misplaced doc comment: the `request_response` doc sits above `receive_sysex` | `transaction.rs:93-102` | Documentation only |
| 12 | `sysex_roundtrip_diag` has its own loop: skips single-byte chunks `>= F8`, appends everything else | `examples/sysex_roundtrip_diag.rs:98-121` | Does not filter embedded realtime bytes; diverges from the library |

## Invariants

- At most one outstanding request per port pair.
- A sent request satisfies `req[0] == 0xF0`, `req[req.len-1] == 0xF7`, `req.len` equals the table
  value for its function, and every middle byte is `<= 0x7F`.
- Assembler state:
  - `len <= capacity`.
  - `in_frame ⇒ len >= 1 and buffer[0] == 0xF0`.
  - `!in_frame ⇒ len == 0`.
- An emitted frame satisfies `frame[0] == 0xF0`, `frame[len-1] == 0xF7`,
  `∀ i ∈ 1..len-1: frame[i] <= 0x7F`, and `len >= 2`.
- Feeding a byte `>= 0xF8` never changes assembler state.
- `now_ns()` is monotonic non-decreasing, the deadline is fixed once set, and
  `polls <= poll_count_max`.
- A response is accepted only if:
  - `len >= 8`;
  - the header is `F0 42 3g 00 01 51` with `g == channel` (strict mode);
  - `fid == expected` or `fid` is a status code.

## Test vectors

From the Rust tests:

| Case | Input / setup | Expected | Citation |
|------|---------------|----------|----------|
| Request header uses channel | `request_global` on ch 12 | sent `F0 42 3C 00 01 51 0E F7` | `transaction.rs:898-912` |
| NAK on request | reply `F0 42 30 00 01 51 24 F7` | `NakReceived(DataLoadError)`; message contains "error status" | `transaction.rs:676-689` |
| NAK on write | reply `F0 42 30 00 01 51 26 F7` | `NakReceived(DataFormatError)` | `transaction.rs:691-705` |
| NAK on raw path | `request_program(0)`, reply `F0 42 30 00 01 51 24 F7` | error | `transaction.rs:734-746` |
| NAK display | reply `… 28 F7` / `… 29 F7` | "User Data CRC Error" / "User Target Error" | `transaction.rs:920-946` |
| ACK | reply `F0 42 30 00 01 51 23 F7` to `send_*` | `Ok(())` | `transaction.rs:519-594`, `tests/sysex_flows.rs:52-66`, `112-124` |
| Wrong dump | `request_current_program`, reply = global dump | `UnexpectedResponse(0x51)`; "unexpected response function ID 0x51" | `transaction.rs:781-796`, `1036-1039` |
| Non-status to write | `send_current_program`, reply `F0 42 30 00 01 51 0E F7` | `UnexpectedResponse(0x0E)` | `transaction.rs:815-829` |
| No reply | empty mock queue | `Timeout`; "transaction timed out after 500ms" | `transaction.rs:601-669`, `1021-1027` |
| Two operations | queue global dump, then program dump | both succeed; 2 messages sent | `transaction.rs:954-979` |

Derived vectors for a byte-level assembler (one row = one input stream, `|` = chunk boundary):

| Input | Expected frames / events | Current Rust |
|-------|--------------------------|--------------|
| `F8 F0 42 30 00 01 51 23 F7` | `F0 42 30 00 01 51 23 F7` | dropped, then timeout |
| `F0 42 30 00 \| F8 \| 01 51 23 F7` | `F0 42 30 00 01 51 23 F7` | same frame |
| `F0 42 30 00 01 F8 51 FE 23 F7` | `F0 42 30 00 01 51 23 F7` | same frame |
| `F0 42 30 \| F0 42 30 00 01 51 23 F7` | `F0 42 30 00 01 51 23 F7` (1 frame aborted) | `F0 42 30 F0 42 …`, then `InvalidHeader` |
| `F0 42 30 00 90 3C 64 F7` | no frame (aborted at `90`; lone `F7` ignored) | 8-byte "frame", then `InvalidHeader` |
| `F0 42 30 00 01 51 23 F7 90 3C 64` | frame, then `90 3C 64` passed to the channel parser | glued to the next chunk |
| `F0` + 2048 × `00` | `FrameOverflow` (capacity 2048) | grows until timeout |
| `F7 F7 F0 F7` | frame `F0 F7` (rejected later by header check) | whole chunk dropped (first byte `F7`), then timeout |

## Rust design

- **Borrowing.** The struct holds `&'a mut` to both ports for its lifetime. That guarantees no
  other code reads the input mid-conversation. Generic over `MidiOutput`/`MidiInput`, so mocks
  slot in (see [transport](transport.md)).
- **Allocation.** Every chunk is a fresh `Vec<u8>` (midir callback → `mpsc`). `buf`, the cleaned
  frame and `SysexFrame.data` are `Vec`s too. The parsers return owned domain values.
- **Time.** `std::time::Instant` (monotonic) and `Duration` for the deadline.
- **Threading.** midir runs a callback on its own thread and pushes each message into an
  **unbounded** `std::sync::mpsc` channel (`transport.rs:226-283`). `receive` blocks on
  `recv_timeout`.
- **Essential:**
  - one outstanding request;
  - the expected-function / ACK / NAK classification;
  - a single deadline;
  - realtime filtering;
  - fragment reassembly.
- **Incidental:**
  - generics and lifetimes;
  - `Vec` returns;
  - the `mpsc` thread bridge;
  - the builder-style `with_timeout`;
  - three slightly different helpers (`request_response` / `raw_request_response` /
    `send_and_wait_ack`). These should collapse into one matcher.

## Zig / TigerStyle notes

**Assembler state machine.** Process one byte at a time and keep only a fixed buffer:

| State | Byte | Action | Next |
|-------|------|--------|------|
| any | `F8`–`FF` | ignore; `realtime_dropped += 1` | unchanged |
| idle | `F0` | `len = 0`; push | in_frame |
| idle | anything else | not SysEx; hand to channel/common parser or drop | idle |
| in_frame | `00`–`7F` | push; if `len == capacity`, `overflowed = true` and return to idle | in_frame |
| in_frame | `F7` | push; **emit frame** | idle |
| in_frame | `F0` | `frames_aborted += 1`; `len = 0`; push | in_frame |
| in_frame | `80`–`EF`, `F1`–`F6` | `frames_aborted += 1`; reprocess byte as idle | idle |

```zig
// sketch
pub const frame_size_max = 2048; // >= 1181 (0x4C); see memory-budget for 0x4A user-slot data

pub const SysexAssembler = struct {
    buffer: [frame_size_max]u8 = undefined,
    len: u16 = 0,
    in_frame: bool = false,
    frames_aborted: u32 = 0,
    overflows: u32 = 0,

    pub const Feed = struct { consumed: usize, frame: ?[]const u8 };

    /// Consumes bytes until one frame completes or `chunk` is exhausted.
    /// The returned frame slice is valid until the next call.
    pub fn feed(self: *SysexAssembler, chunk: []const u8) Feed { ... }
};
```

- `feed` returns `consumed`, so a chunk holding *frame + more bytes* is resumed on the next call
  without loss.
- Assert the invariants above at the end of `feed`.
- In tests, assert the "frame or not" result is the same for every split of the stream into
  chunks (exhaustive two-way splits, random n-way splits).

**Transaction loop.** Poll the platform input at the caller's pace:

```zig
// sketch
pub const TransactionOptions = struct {
    timeout_ms: u32 = 5000,
    poll_interval_ms: u32 = 1,
    frames_unrelated_max: u8 = 16,
    channel_strict: bool = true,
};

pub const Expect = union(enum) { dump: FunctionId, ack, identity_reply };
pub const Outcome = union(enum) { frame_len: u16, ack };

pub fn transact(port: *Port, assembler: *SysexAssembler, request: []const u8,
    expect: Expect, options: TransactionOptions, out: *[frame_size_max]u8,
    diagnostics: ?*Diagnostics) Error!Outcome
{
    assert(request.len >= 6 and request[0] == 0xF0 and request[request.len - 1] == 0xF7);
    port.input_drain();                       // bounded: at most ring capacity bytes
    assembler.reset();
    try port.send(request);
    const deadline_ns = port.now_ns() + @as(u64, options.timeout_ms) * 1_000_000;
    const poll_count_max = options.timeout_ms / options.poll_interval_ms + 1;
    var unrelated: u8 = 0;
    var poll: u32 = 0;
    while (poll < poll_count_max) : (poll += 1) {
        // read available bytes into a fixed scratch; feed; match; sleep poll_interval_ms
        if (port.now_ns() >= deadline_ns) break;
    }
    return error.Timeout;
}
```

How to use it:
- **Matching.**
  - Accept the frame if the header is `F0 42 3g 00 01 51`, `g` matches (when
    `channel_strict`), and `fid == expect.dump`.
  - Return `.ack` for `0x23`. For a NAK (`0x24`, `0x26`–`0x2F`), fail with `error.Nak` and
    put the status byte in `diagnostics.status`. Treat ACK to a request as an error by the
    caller's choice, not an accident.
  - Count anything else as unrelated. Fail with `error.FramesUnrelatedExceeded` after
    `frames_unrelated_max`.
  - For `0x4C`, also compare the echoed program number with the request.
  - `identity_reply` matches `F0 7E ?? 06 02 42`; see
    [device identity](../protocol/device-identity.md).
- **Errors vs outcomes.** A NAK is an operating error: the write did not happen. A Zig error
  cannot carry the status byte, so it goes in the caller-owned `Diagnostics` struct, as
  described in [errors](errors.md).
  - Error set: `TransactionError || TransportError` from [errors](errors.md)
    (`Timeout`, `Nak`, `ResponseUnexpected`, `FramesUnrelatedExceeded`, `FrameOverflow`,
    `InputOverrun`, plus `WriteFailed`, `Disconnected`, …).
  - Report `elapsed_ns` in a diagnostics out-param or log line, not the configured timeout.
- **Clock.** Monotonic only (`now_ns()` from the platform layer; see
  [platform MIDI](../zig/platform-midi.md)). Never use wall-clock time.
- **Threading.**
  - Prefer single-threaded polling.
  - If the OS delivers input via callback (CoreMIDI, ALSA seq), copy bytes into a fixed **SPSC
    byte ring** (16384 bytes ≥ 4 × 1181; see [transport](transport.md)).
  - On ring overflow set a sticky flag; the next transaction fails with `error.InputOverrun`.
    Do not grow the ring.
- **Sizes.**

  | Item | Bytes |
  |------|------:|
  | Largest request (`0x1E`) | 11 |
  | Status frame | 8 |
  | Largest fixed reply/dump (`0x4C`) | 1181 |
  | Program `0x40` | 1179 |
  | Global `0x51` | 80 |
  | Assembler | 2048 |
  | Input ring | 16384 |
  | Scratch read buffer | 256 |

  See [memory budget](../zig/memory-budget.md) and [user modules](../protocol/user-modules.md) for
  `0x4A`.
- **Per-operation options.** A write (`0x4C`) may need a longer ACK timeout than a read. Keep one
  `TransactionOptions` per call site; do not use global state.
- **Naming.** `timeout_ms`, `deadline_ns`, `poll_count_max`, `frames_unrelated_max`: units and
  qualifiers last.

## Open questions

- [hw] What is the latency between the last request byte and the first reply byte for `0x10`,
  `0x1C`, `0x0E`? What is the ACK latency after `0x4C`/`0x40` (flash write)?
- [hw] Which platform fragmented 1181-byte replies? Does the XD itself interleave `F8` inside
  SysEx, or does the host driver merge streams? The commits say "some platforms" and "the driver".
- [hw] Does the device answer requests with `0x24` when busy (for example during sequencer
  playback)? Does it ever send ACK to a request?
- [hw] Does the reply always carry the request's `g` nibble, so that strict channel matching is
  safe?
- [hw] Does the `0x4C` reply echo exactly the requested program number? See
  [program blob](../protocol/program-blob.md) for bank/program errata.

## Sources

- `crates/minilogue-xd/src/sysex/transaction.rs:1-31` — module doc; `47-87` — struct and accessors;
  `93-142` — `receive_sysex`; `144-206` — helpers; `218-358` — operations; `364-378` —
  `FailingOutput`; `395-1039` — tests.
- `crates/minilogue-xd/src/sysex/frame.rs:195-202` — `SysexFrame`; `256` — `build_status`; `269` —
  `parse_sysex`.
- `crates/minilogue-xd/src/sysex/program/mod.rs:272-290` — `parse_program_dump` header checks.
- `crates/minilogue-xd/src/transport.rs:16-34` — traits; `103-132` — `MockMidiInput`; `226-320` —
  `MidirInput`.
- `crates/minilogue-xd/src/error.rs:77-87` — `Timeout`, `NakReceived`, `UnexpectedResponse`.
- `crates/minilogue-xd/tests/sysex_flows.rs:20-146` — integration flows.
- `crates/minilogue-xd/examples/read_patch.rs:24-43`, `read_edit_buffer.rs:175-183`,
  `quintal.rs:592-632`, `sysex_roundtrip_diag.rs:26-35`, `90-121` — hardware usage.
- Commits `bc6c452`, `685ea64`, `d465474`, `5e4b4f0`, `1c16566` (`git --no-pager show <sha>`).
- Korg MIDI Impl rev 1.01 §1-2, §1-4 (*4 E), §2-3, NOTE 2.
