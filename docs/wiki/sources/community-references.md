---
title: Source — community reference implementations
category: source
sources:
  - README.md
  - CLAUDE.md
  - .gitignore
related:
  - sources/korg-midi-implementation.md
  - protocol/program-blob.md
  - protocol/file-formats.md
  - protocol/user-modules.md
  - design/testing-strategy.md
updated: 2026-10-11
---

# Source — community reference implementations

> Before writing the crate, the authors cross-checked the Korg spec against four community
> projects that had already been validated against real hardware and real `.mnlgxd*` files.
> The projects are cloned into a gitignored `workbench/` directory, which is **absent from this
> checkout**. They count as **[community]** evidence: stronger than the spec text, weaker than
> captures from a real device.

## The four references

| Project | Language | Contributes | Wiki pages it backs |
|---------|----------|-------------|---------------------|
| [korginc/logue-sdk](https://github.com/korginc/logue-sdk) | C/C++ | Official Korg SDK: user-module binary format, CRC32, unit packaging | [user modules](../protocol/user-modules.md) |
| [gekart/mnlgxd.py](https://gist.github.com/gekart/b187d3c16e6160571ccfcf6c597fea3f) | Python | Program blob parser; "ground truth for TABLE 2 byte offsets" | [program blob](../protocol/program-blob.md) |
| [isnotinvain/minilogue-xd-util](https://github.com/isnotinvain/minilogue-xd-util) | Python | Patch object model; sequencer and motion byte layout | [sequencer blob](../protocol/sequencer-blob.md) |
| [gazzar/loguetools](https://github.com/gazzar/loguetools) | Python | Cross-synth patch tools; `.mnlgxd*` zip container formats | [file formats](../protocol/file-formats.md) |

## How they were used

The project rules (`CLAUDE.md`, "Key Conventions") require validating implementations against
real `.mnlgxdlib`/`.mnlgxdprog` files *and* the workbench Python implementations, not just the
spec. Where the spec and the references disagreed, the crate followed the references and the
files. The two errata the README records are bank/program-number encoding and 448-byte preset
blobs. Both came from this cross-checking.

## For the Zig port

- Clone the four repos for cross-validation only. Do not vendor them. The Zig port takes no
  code dependency on them.
- When a [community] source and the Rust crate disagree, record the conflict on
  [errata](../protocol/errata.md) and settle it with a hardware capture or a real factory file.
  Do not pick a side by majority.
- Real Korg files (factory `.mnlgxdlib`, librarian exports) make the best golden test vectors.
  This repo ships none. Collect a few and store them as fixtures under the Zig project's
  `testdata/`. See [test plan](../zig/test-plan.md).

## Sources

- `README.md:207-228` — "Community references" table and "Known documentation errata".
- `CLAUDE.md:85-86` — validate against real files and workbench implementations.
- `.gitignore:23` — `/workbench` is ignored.
