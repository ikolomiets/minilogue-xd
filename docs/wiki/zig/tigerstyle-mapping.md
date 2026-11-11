---
title: Rust idioms → Zig / TigerStyle mapping
category: zig
sources:
  - crates/minilogue-xd/src/param/mod.rs
  - crates/minilogue-xd/src/param/nrpn.rs
  - crates/minilogue-xd/src/controller.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/src/error.rs
  - https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md
related:
  - design/architecture.md
  - design/typed-values.md
  - design/errors.md
  - zig/module-layout.md
  - zig/memory-budget.md
  - zig/test-plan.md
updated: 2026-11-11
---

# Rust idioms → Zig / TigerStyle mapping

> This page is a translation table. For each pattern the Rust crate uses, it gives the Zig
> pattern to use instead and the TigerStyle rule behind the choice. Most of it is mechanical.
> Three items change the design: (1) transactions become **sans-I/O state machines fed bytes and
> time**; (2) every `Vec<u8>` becomes a **fixed array sized by a comptime maximum**; (3) the
> range-checked newtypes become **native narrow integers plus assertions**, with validation only
> where untrusted bytes enter.

## Facts

### Values and types

| Rust | Zig / TigerStyle | Rule |
|------|------------------|------|
| `U4`, `U7`, `U14`, `TenBitParam` newtypes with `new() -> Result` ([typed values](../design/typed-values.md)) | Native `u4`, `u7`, `u14`, `u10`. Narrow at the trust boundary with `std.math.cast(u7, x) orelse return error.OutOfRange`; inside the core, `@intCast` plus an assert. | Explicitly sized ints; assertions for programmer errors |
| `I14` (pitch bend −8192…8191) | `i14`; wire form `u14 = @bitCast(value) +% 0x2000`, i.e. offset binary. Write paired tests for the bit pattern. | Explicit sizes |
| `ProgramNumber` (0–499) | `const ProgramNumber = enum(u16) { _ }` plus `fn init(n: u16) ?ProgramNumber`. Store `u16` in structs; `comptime assert(program_count == 500)`. | Constants with comptime asserts |
| `ProgramName` (≤12 ASCII printable, null padded) | `[12]u8` inline in the blob, with accessors that read up to the first `0` | Static allocation |
| `f32 0.0..=1.0 → TenBitParam` (`controller.rs:29-39`, rejects NaN because the range check fails) | Offer an integer API first (`set_cutoff(u10)`). Make the float convenience a separate function that asserts `!std.math.isNan(x)`, clamps or errors, and rounds the same way (`@round(x * 1023)`). | Prefer integers; fewer hidden conversions |
| Stepped enums + `trait SteppedParam { to_tx_value, from_rx_value, to_program_value, from_program_value }` (`param/mod.rs:22-42`) | `enum(u8)` whose tag value *is* the program-data value, plus one comptime table per enum: `tx: [n]u7` and `rx_upper_bounds: [n]u7`. Comptime-assert that the bounds are strictly increasing and the last one is 127. See [stepped params](../design/stepped-params.md). | Comptime tables; assert relationships at compile time |
| `CcParam` / `NrpnParam` enums with payloads | `union(enum)` with narrow payloads, plus a comptime `[128]?CcKind` lookup from controller number to kind | Static tables |
| `#[derive(Debug, Clone, PartialEq, Eq, Hash)]` | Nothing needed: structs compare with `std.meta.eql` in tests; formatting via `std.fmt` default | — |
| `impl Display` for names/labels | `fn name(self) []const u8` returning comptime strings | — |

### Memory

| Rust | Zig / TigerStyle | Rule |
|------|------------------|------|
| `fn build_*(...) -> Vec<u8>` (every SysEx/message builder) | `fn build_*(..., out: *[max_len]u8) u16` or `out: []u8` with `assert(out.len >= max_len)`, returning the written length. Each builder has a comptime `max_len`. | Static allocation; put a limit on everything |
| `fn parse_*(&[u8]) -> Result<T>` | Same shape: `fn parse_*(bytes: []const u8) Error!T`. No allocation; results go into fixed structs or slices *into the input*. | — |
| `ProgramData { synth, sequencer, raw: Option<Vec<u8>> }` + `to_bytes_with_base` ([byte fidelity](../design/byte-fidelity.md)) | `const Program = struct { bytes: [1024]u8 }` with offset accessors (`fn cutoff(p) u10`, `fn set_cutoff(p, v)`) that read and write the blob *in place*. Fidelity comes for free and no merge step is needed. | Static allocation; simpler design |
| `MockOutput { messages: Vec<Vec<u8>> }` | Fixed `[max_messages][max_len]u8` + `[max_messages]u16` lengths; assert on overflow (test bug) | Limits everywhere |
| `mpsc::channel` (unbounded) | SPSC ring buffer with an overflow counter ([platform MIDI](platform-midi.md#input-handoff)) | Bounded queues |
| `String` in errors (`Error::Zip(String)`, `MidiIo(String)`) | Error set + optional `*Diagnostics` out-parameter with fixed fields (`os_code: i32`, `offset: u32`, `status: u8`) | No allocation |

### Control flow and time

| Rust | Zig / TigerStyle | Rule |
|------|------------------|------|
| `SysexTransaction::request_*` blocks on `MidiInput::receive(timeout)` in a `loop` until `F7` or the deadline (`transaction.rs:103-140`) | `Transaction` state machine: `begin(kind, out_request) → on_bytes(chunk) → tick(now_ns) → .pending/.done/.failed`. The caller owns the loop and the clock. See [SysEx transactions](../design/sysex-transactions.md). | Don't react directly to external events; bounded waits; deterministic tests |
| `std::time::Instant::now()` inside library code | `now_ns: u64` passed in. Only `io/` reads `std.Io.Clock`. | Determinism |
| `NrpnReceiver::feed(cc) -> Option<NrpnParam>`, a 4-state enum (`nrpn.rs:406-440`) | Keep it as is: `const State = union(enum) { idle, have_msb: u7, addressed: Address, have_data_lsb: struct { address: Address, data_lsb: u7 } }`. `fn feed(self: *Receiver, cc: Cc) ?NrpnParam` with an exhaustive `switch`. | Already sans-I/O |
| `CcParamReceiver` pairing CC63 with the next 10-bit CC | Same push parser; assert that a pending CC63 is consumed or discarded within the next message | Paired assertions |
| `for` over iterators, `.iter().filter().collect()` (e.g. stripping realtime bytes in `receive_sysex`) | An explicit `for (bytes) |b|` with an index into a fixed output buffer; `assert(written <= out.len)` | Simple control flow |
| Builder chains (`PatchBuilder::new().a().b().build()`) | A struct literal with field defaults (`Program.init_default()` then setters), or an options struct for the 3–4 required inputs | Options structs; no hidden state |
| Generics `<O: MidiOutput>` | `anytype` / comptime type parameter for the single backend, or `*const fn` tables if runtime polymorphism is needed. No traits. | Explicit |

### Errors

| Rust | Zig / TigerStyle |
|------|------------------|
| `thiserror` `Error` + `SysexError` enums (see [errors](../design/errors.md)) | Error sets per layer: `CodecError`, `FrameError`, `BlobError`, `TransactionError`, `IoError`. Merge them with `||` only at the API surface. |
| `Result<T>` everywhere, including on values the caller already validated | Programmer errors are assertions. Return error sets only for **operating errors**: bytes from the wire or a file, OS failures, timeouts, and NAKs. |
| `SysexError::NakReceived(SysexStatus)` carrying the status byte | `error.Nak` + `diagnostics.status = 0x24..0x2F` |

## Invariants

- Every function has at least 2 assertions, counting asserts on arguments, results, and
  invariants. For pure encoders a typical pair is `assert(out.len >= max_len)` on the way in
  and `assert(written <= max_len)` on the way out.
- Paired assertions across directions:
  - `encode_7bit` asserts the input length is at most `decoded_len_max`, and `decode_7bit`
    asserts `encoded_len(decoded_len) == input.len`.
  - `Program.validate()` runs before building a dump and again after parsing one.
  - The 10-bit split asserts `(msb << 3) | lsb == value`.
- Functions stay at 70 lines or fewer. In Rust, `ProgramData::to_bytes`/`from_bytes` and the
  sequencer parsers are long, field-by-field functions. In Zig, the in-place accessor design
  removes them completely.
- No recursion appears anywhere, and none exists in the Rust crate either.
- Lines are 100 columns or fewer, and `zig fmt` is canonical.

## Errata & gotchas

- `usize` is tempting for lengths. Prefer `u16` for message lengths (max wire message <
  65536) and `u32` for file offsets, with comptime asserts on the maxima. Use `usize` only
  for indexing.
- Zig `enum(u8)` with gaps: some stepped parameters have non-contiguous program values. Use a
  non-exhaustive enum or an explicit table rather than `@enumFromInt` on untrusted bytes, which
  is illegal behavior if the value is not a valid tag.
- `@round` on `f32` rounds half away from zero, the same as Rust's `f32::round`. Pin this with
  test vectors at 0.5/1023 boundaries.

## Zig / TigerStyle notes

```zig
// sketch — pure, bounded, asserted builder (shape every SysEx builder should follow)
pub const program_request_len = 10;

pub fn build_program_request(
    options: struct { channel: u4, program: ProgramNumber },
    out: *[program_request_len]u8,
) void {
    const number: u16 = @intFromEnum(options.program);
    assert(number < program_count);
    out.* = .{ 0xF0, korg_id, 0x30 | @as(u8, options.channel), 0x00, 0x01, 0x51,
        function_program_request, @truncate(number & 0x7F), @truncate(number >> 7), 0xF7 };
    assert(out[0] == 0xF0 and out[program_request_len - 1] == 0xF7);
}
```

## Open questions

- Should the float API exist at all in the Zig port, or should callers convert themselves? The
  Rust controller exposes only `f32` setters for 10-bit parameters.

## Sources

- `crates/minilogue-xd/src/param/mod.rs:22-42` — `SteppedParam` trait.
- `crates/minilogue-xd/src/param/nrpn.rs:406-440` — `NrpnState` and `NrpnReceiver`.
- `crates/minilogue-xd/src/controller.rs:29-39` — `f32_to_10bit`.
- `crates/minilogue-xd/src/sysex/transaction.rs:103-140` — blocking `receive_sysex` loop.
- `crates/minilogue-xd/src/sysex/program/mod.rs:225-239` — `build_program_request` (10 bytes).
- TigerStyle (TigerBeetle `docs/TIGER_STYLE.md`) — rules quoted in the
  [schema digest](../AGENTS.md#tigerstyle-digest-constraints-for-every-zig-note).
