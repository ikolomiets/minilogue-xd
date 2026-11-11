---
title: Zig port — test plan
category: zig
sources:
  - crates/minilogue-xd/src/codec.rs
  - crates/minilogue-xd/src/param/encoding.rs
  - crates/minilogue-xd/src/sysex/transaction.rs
  - crates/minilogue-xd/tests/
  - .github/workflows/ci.yml
  - /opt/homebrew/Cellar/zig/0.16.0_1/lib/zig/std/testing.zig (fuzz, Smith)
related:
  - design/testing-strategy.md
  - zig/port-overview.md
  - zig/module-layout.md
  - zig/memory-budget.md
  - protocol/errata.md
updated: 2026-10-11
---

# Zig port — test plan

> How the Zig port proves it matches the Rust crate and the hardware. Testing has five tiers:
> **exhaustive** tests for small domains, **golden vectors** from the Rust oracle and hardware
> captures, **seeded property loops**, a **deterministic simulated XD** for transactions, and
> **hardware acceptance** runs. Each milestone on [port overview](port-overview.md) is gated by
> named tiers. Every disputed fact from [errata](../protocol/errata.md) gets a vector tagged
> with its provenance, so a later hardware capture only needs to flip a flag. The Rust test
> inventory, the patterns, and the "port first" ranking are on
> [testing strategy](../design/testing-strategy.md); this page doesn't repeat them.

## Facts

### What we are matching

- Rust has 973 `#[test]` functions (946 in `src/` and 27 in `tests/`). Macro-generated enum
  tests bring the run count to about 1,133. There are also 12 doctests and 9 proptest properties
  (256 cases each, unseeded). See [testing strategy](../design/testing-strategy.md#inventory).
- Rust has **no automated test** for:
  - fragmented SysEx, realtime bytes inside SysEx, or noise before `F0`;
  - bytes → struct → bytes identity on captured blobs;
  - the committed `berlin_school.mid`.
  The Zig port adds these as first-class tests rather than porting gaps.

### Test tiers

| Tier | What | Where | Runs |
|------|------|-------|------|
| T1 Exhaustive | Every value of a small domain: 7-bit, 10-bit, 8-bit hi-res, 14-bit, program numbers, stepped `rx` bands, receiver input pairs | `test` blocks in each `src/**.zig` | every `zig build test` |
| T2 Golden vectors | Byte-exact expected output from the Rust oracle (`[rust]`) and captures (`[hw]`) | `test/vectors/*.hex`, `*.bin`, `*.mid` loaded with `@embedFile` | every build |
| T3 Seeded properties | Round trips and "error or re-encodable" decoders over random input; seed printed on failure | `test` blocks + `test/properties.zig` | every build (bounded); longer runs nightly |
| T4 Simulation | Mock XD with injected chunking, realtime insertion, noise, NAK, drop, duplicate, and wrong function; simulated `now_ns` | `test/simulation.zig` | every build (fixed seeds); nightly random seeds |
| T5 Hardware acceptance | Examples against a real XD: list ports, read edit buffer, write edit buffer, read/write stored program, CC sweep | `examples/` + a checklist | manual, per release and per OS |

Optional: `std.testing.fuzz(context, testOne, options)` with `std.testing.Smith` exists in Zig
0.16 for coverage-guided fuzzing. Use it on the same entry points as T3. It is not portable
across Zig versions, so keep T3 as the baseline.

### Milestone gates

| Milestone | Required tiers | Key vectors / properties |
|-----------|----------------|--------------------------|
| M1 wire | T1, T2, T3 | Codec lengths 0…1024 (`n + ⌈n/7⌉`), every byte at every group position, high-bit rejection; message round trips; stream parser with running status and realtime bytes anywhere |
| M2 params | T1, T2 | 10-bit CC split (CC63 low 3 bits first); every stepped enum × 128 rx values; NRPN receiver FSM never outputs >1023 and resets on unrelated CC; NRPN addresses **both** `[rust]` and `[spec]` tables, selected by a build option until resolved |
| M3 SysEx core | T1, T2, T3, T4 (assembler only) | Frame build/parse per function ID; status codes `0x23`–`0x2F` (`0x25` unassigned); assembler: `F0` restarts, other status aborts, `F8`–`FF` dropped, length cap |
| M4 program | T2, T3 | Byte identity on captured `.prog_bin` blobs; 10-bit little-endian writes preserve the upper 6 flag bits; `PROG`/`PREDSQ` magic; sync/ring inversion |
| M5 transaction | T4 | Every operation × {ok, NAK, timeout, wrong function, duplicate, chunked + realtime}; exact request bytes asserted |
| M6/M7 I/O | T5 | Port names recorded per OS; edit buffer round trip on hardware |
| M8 files | T2 | Read every fixture `.mnlgxdprog`/`.mnlgxdlib`; Rust reads Zig-written files (cross-check job) |
| M9 user modules | T2 | CRC-32 check value `0xCBF43926`; TABLE 5/6/7 layouts vs community captures |
| M10 SMF | T2 | Byte-equal SMF for the `berlin_school` event list |

### Vector file format

Use one line per case, with hex bytes and no dependencies:

```text
# provenance: rust | spec | hw ; source: <file:line or capture id>
encode_7bit  01 02 03 81 ...  =>  00 01 02 03 01 ...
```

- Generate `[rust]` vectors with a small oracle binary in the Rust repo; see
  [port overview](port-overview.md#the-rust-crate-as-oracle).
- `[hw]` vectors come from captures. Commit raw `.syx` dumps next to the 1024-byte `.bin` blob.
- Where `[rust]` and `[spec]` disagree, commit both and mark which one the port implements by
  default. See [errata](../protocol/errata.md).

### CI matrix

| Axis | Values |
|------|--------|
| OS | macOS (arm64), Linux (x86_64), Windows (x86_64) |
| Optimize | Debug, ReleaseSafe (assertions on); ReleaseFast build-only |
| Cross-compile check | `zig build -Dtarget=x86_64-windows`, `aarch64-linux`, `x86_64-macos` from Linux, to prove no host-only deps |
| Lint | `zig fmt --check`; a script that fails on non-empty `build.zig.zon` `.dependencies`; a line-length check (≤ 100) |
| Nightly | T3/T4 with random seeds (seed logged), the Rust differential job (needs `cargo`) |

The Rust CI runs `cargo test` across three OSes and feature sets, but does not enforce coverage.
See [testing strategy](../design/testing-strategy.md#ci-matrix-githubworkflowsciyml). The Zig port
should *not* chase a coverage percentage. Assertion density, plus T1 and T4, is the TigerStyle
substitute.

## Invariants

- Tests use the same comptime tables as the code (stepped enums, CC/NRPN maps). Generate the
  expected *structure* from the table, but check *values* against independent vectors, so a
  table typo cannot pass its own test.
- Every loop in tests is bounded by a named constant (e.g. `codec_iterations_max`), as in
  production code.
- No test touches real MIDI ports or the wall clock. T5 lives in `examples/` only.
- No test allocates, apart from `std.testing.allocator` in the few file tests that load whole
  archives. Prefer fixed buffers there too.

## Errata & gotchas

- The Rust `berlin_school.mid` is committed but untested. Regenerate it from Rust once before
  trusting it as a golden file.
- `quintal.rs` iterates a `HashMap`, so its output order is nondeterministic. Don't use it as a
  golden source.
- MTS checksum: Rust is off by `0x7E` versus the spec. Vectors must follow the spec, `[spec]`,
  and say so. See [tuning](../protocol/tuning.md).

## Open questions

- Which hardware captures can be committed (factory programs may be copyrighted content)?
- Should T4 also model DIN timing (0.32 ms/byte) to test the "deadline starts after write"
  rule? See [SysEx transactions](../design/sysex-transactions.md).

## Sources

- `crates/minilogue-xd/src/codec.rs`, `src/param/encoding.rs` — the proptest suites being
  replaced by T1/T3.
- `crates/minilogue-xd/src/sysex/transaction.rs` — mock-driven transaction tests replaced by T4.
- `crates/minilogue-xd/tests/` — integration tests (`cc_coverage`, `nrpn_coverage`,
  `sysex_flows`, …).
- `.github/workflows/ci.yml` — the Rust CI matrix.
- Zig 0.16 `std/testing.zig:1220-1232` — `Smith` and `fuzz`.
