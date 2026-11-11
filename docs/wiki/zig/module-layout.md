---
title: Zig module layout
category: zig
sources:
  - crates/minilogue-xd/src/lib.rs
  - crates/minilogue-xd/Cargo.toml
related:
  - design/architecture.md
  - sources/rust-crate.md
  - zig/port-overview.md
  - zig/tigerstyle-mapping.md
  - zig/platform-midi.md
  - zig/memory-budget.md
updated: 2026-10-11
---

# Zig module layout

> The proposed source tree for the Zig port. It mirrors the Rust layers but splits **pure
> protocol code** (`wire/`, `param/`, `sysex/`, `file/`) from **I/O** (`io/`). Only `io/` may
> import OS bindings, and only `transaction.zig` and `controller.zig` know about time or
> transports. Names are `snake_case` files with no abbreviations. Each file maps back to the
> Rust module whose behavior it reproduces.

## Facts

```text
minilogue_xd/
  build.zig                 lib + tests + examples; links CoreMIDI/CoreFoundation | winmm by target
  build.zig.zon             name, version, minimum_zig_version — and NO dependencies
  src/
    root.zig                public API surface (re-exports), comptime-asserts global invariants
    constants.zig           every size/limit/ID in one place (see memory-budget.md)
    wire/
      message.zig           channel/realtime/common messages, status → length table, running status
      stream_parser.zig     byte stream → messages (+ SysEx start/data/end events); realtime anywhere
      codec_7bit.zig        Korg NOTE 1 7↔8-bit packing, encoded_len()/decoded_len()
    param/
      encoding.zig          10-bit (CC63 lsb first), 8-bit hi-res, 14-bit splits
      stepped.zig           comptime tables: enum tag ↔ tx value ↔ rx band ↔ program value
      cc.zig                CC number → parameter kind table; CC63 pairing receiver
      nrpn.zig              NRPN address table; 4-state receiver; send sequence builder
    sysex/
      frame.zig             Korg header, function IDs, status (ACK/NAK) codes, parse/build
      assembler.zig         F0…F7 reassembly from fragments, realtime filtering, bounded length
      identity.zig          universal identity request/reply, search device
      poly_chain.zig        poly chain messages (0x60/0x61)
      tuning.zig            user scale (384), user octave (36), MIDI Tuning Standard messages
      global.zig            63-byte GLOB blob, offset accessors
      program.zig           1024-byte program blob container, PROG/PREDSQ magic, dumps & requests
      program_synth.zig     synth field accessors (bytes 0–155)
      program_sequencer.zig sequencer/motion accessors (bytes 156–1023)
      user_module.zig       logue SDK slot messages, CRC-32
    transaction.zig         request/response state machine: begin → on_bytes → tick → done/failed
    controller.zig          typed setters → message bytes into an outbound queue (no I/O itself)
    file/
      zip_reader.zig        end record + central directory parse; entry streaming (store/deflate)
      zip_writer.zig        local headers + central directory; deflate via std.compress.flate
      program_file.zig      .mnlgxdprog / .mnlgxdlib / FileInformation.xml
      smf.zig               Standard MIDI File (format 0) writer
    io/
      transport.zig         backend selection by builtin.os.tag; port name matching
      ring_buffer.zig       SPSC byte ring (OS thread → app thread)
      coremidi.zig          macOS
      alsa_rawmidi.zig      Linux
      winmm.zig             Windows
      mock.zig              scripted input chunks, recorded output (tests, examples)
    time.zig                monotonic now_ns() for io/ and examples only
  test/
    vectors/                byte-exact fixtures (.prog_bin, .mnlgxdprog, SysEx dumps) — see test-plan.md
    simulation.zig          seeded transaction/assembler simulation (VOPR-style)
  examples/                 list_ports, cutoff_sweep, dump_program, write_patch, export_midi, …
```

### Rust → Zig mapping

| Rust module | Zig file(s) | Change |
|-------------|-------------|--------|
| `message/{channel,realtime,common,types}.rs`, `message/mod.rs` (`parse_midi_bytes`) | `wire/message.zig`, `wire/stream_parser.zig` | The parser handles a raw byte *stream* (needed for ALSA rawmidi), not just framed messages |
| `codec.rs` | `wire/codec_7bit.zig` | Same |
| `transport.rs` | `io/transport.zig`, `io/ring_buffer.zig`, `io/mock.zig`, OS files | Blocking `receive(timeout)` becomes non-blocking `drain()` |
| `device.rs` | `io/transport.zig` (port names), `param/cc.zig` (`cc_name`) | Breaks the `transport → device` dependency wart |
| `error.rs` | Error sets declared per file | `error.rs` imported `sysex::frame::SysexStatus`; here status codes live in `sysex/frame.zig` and errors carry them through diagnostics |
| `param/{encoding,enums,cc,nrpn}.rs` + `param/mod.rs` | `param/*.zig` | Enums become comptime tables |
| `sysex/frame.rs` | `sysex/frame.zig` | Same |
| `sysex/transaction.rs` (`receive_sysex`) | `sysex/assembler.zig` + `transaction.zig` | Split reassembly (pure) from request/response (state machine) |
| `sysex/program/{mod,synth,sequencer}.rs` | `sysex/program*.zig` | Struct-of-fields becomes in-place accessors on `[1024]u8` |
| `sysex/program/file.rs` | `file/program_file.zig` + `file/zip_*.zig` | No `zip` crate |
| `sysex/{global,tuning,user_module,identity,poly_chain}.rs` | same names under `sysex/` | Same |
| `controller.rs` | `controller.zig` | Writes into an outbound queue/buffer; the app flushes to `io/` |
| `connection.rs` (`MinilogueXd<O>`: channel + output, `send_message`) | (none) | A thin wrapper. `wire/message.zig` encode plus `io/` write covers it. |
| `builder/{patch,sequence}.rs` | (none) or a thin `builder.zig` | Struct defaults + setters replace fluent builders |
| `midi_file.rs` | `file/smf.zig` | Fixed event capacity, no `Vec` sort; insertion into a bounded array or a caller-supplied buffer |
| `crates/minilogue` umbrella (`minilogue::xd`) | (none) | Zig modules are named in `build.zig` |

### Allowed imports (enforced by review, optionally by a build step)

```text
wire/      → std (assert, mem, math), constants
param/     → wire/, constants
sysex/     → wire/, param/, constants
file/      → sysex/, wire/, std.compress.flate, std.hash.Crc32, constants
transaction.zig, controller.zig → sysex/, param/, wire/
io/        → std.os.*, builtin, OS externs, constants
root.zig   → everything
```

Nothing in `wire/`, `param/`, `sysex/`, or `file/` imports `io/` or `time.zig`. This is the
sans-I/O boundary.

## Invariants

- `constants.zig` is the single source of truth for sizes. Every buffer type is declared from
  it, e.g. `[constants.program_size]u8`, and its relationships are comptime-asserted, e.g.
  `encoded_len(program_size) == 1171`. See [memory budget](memory-budget.md).
- Each file keeps its tests at the bottom (`test "..." {}`), following the Rust convention of
  in-file unit tests. Cross-module and byte-fixture tests live in `test/`.
- Public API names follow TigerStyle: units and qualifiers come last (`timeout_ns`,
  `program_size_max`), and there are no abbreviations (`program`, not `prog`, except in
  on-disk names like `Prog_000.prog_bin`).

## Errata & gotchas

- The two Rust layering warts disappear with this layout. `error → sysex::frame` becomes
  per-layer error sets, and `transport → device` moves port-name patterns into `io/`. See
  [rust crate source](../sources/rust-crate.md#internal-dependency-graph-non-test-code).
- `controller.zig` must not hold a transport. In Rust, `RealtimeController<O>` owns the
  output. In Zig it should fill a caller-provided outbound buffer, so it can be tested without
  mocks and scheduled by the caller, which the [wayfinder DSL](../sources/wayfinder-dsl.md) needs.

## Open questions

- Should `builder.zig` exist at all? It would be a convenience layer for examples, not needed
  by the core.
- Should SMF export live in this library or in a separate tool? In Rust it is in-crate.

## Sources

- `crates/minilogue-xd/src/lib.rs` — module declarations and four-layer diagram.
- `crates/minilogue-xd/Cargo.toml` — features and dependencies being removed.
- [rust crate source](../sources/rust-crate.md) — module map and dependency graph.
