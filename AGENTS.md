# Repository Guidelines

## Project Structure & Module Organization

This Rust 2021 workspace provides MIDI control and patch manipulation for the Korg Minilogue XD.

- `crates/minilogue-xd/src/` contains the implementation: `message`, `codec`, and `transport` form the foundation; `param` handles CC/NRPN; `sysex` handles binary data and transactions; `builder` and `controller` expose higher-level APIs.
- `crates/minilogue/` re-exports the library as `minilogue::xd`.
- Unit tests live beside source code; integration tests are in `crates/minilogue-xd/tests/`. Runnable utilities and patches are in `crates/minilogue-xd/examples/`.
- `docs/design/` and `docs/dev/` hold plans; `docs/korg-user-guide/` contains reference material. `assets/` contains images and contributor guidance.

## Build, Test, and Development Commands

Use stable Rust with `rustfmt` and Clippy. Linux MIDI builds require `libasound2-dev`.

- `make build` / `make build-release`: compile debug / optimized builds.
- `make format`: format the workspace; run after Rust changes.
- `make lint`: run Clippy with warnings denied and check formatting; run before tests.
- `make test`: run workspace tests with all features.
- `cargo test -p minilogue-xd --no-default-features`: check the minimal feature configuration.
- `make coverage` / `make coverage-html`: generate library coverage using `cargo-llvm-cov`.
- `make docs`: build workspace API documentation.
- `cargo run -p minilogue-xd --example list_ports`: list local MIDI ports.

## Coding Style & Naming Conventions

Follow rustfmt defaults: four-space indentation, `snake_case` modules/functions, `UpperCamelCase` types, and `SCREAMING_SNAKE_CASE` constants. Document public APIs with Rustdoc. Preserve typed parameter enums and range-checked MIDI values. Gate optional dependencies appropriately; declare `required-features` for examples needing `midi-io` or `file-formats`.

## Testing Guidelines

Use Rust's built-in tests and `proptest` for encoding invariants. Name tests descriptively, such as `default_patch_serialization_round_trip`; run a subset with `cargo test -p minilogue-xd round_trip`. Use `MockOutput` and `MockMidiInput` for hardware-independent flows. Target 95%+ line coverage; never hide failures with ignored tests. For binary-format changes, verify byte-exact round trips against real Korg files when available, accounting for documented spec errata.

## Commit & Pull Request Guidelines

Use concise, imperative commit subjects following history, such as `Fix SysEx program dump round-trip`; no mandatory prefix is established. Keep changes focused. PRs should explain behavior changes, link relevant issues or design plans, and report validation commands and results. Update examples/docs for API changes and ensure CI passes across supported platforms and feature configurations.
