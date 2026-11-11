# Wiki schema — minilogue-xd design wiki

This directory is an **LLM Wiki** (after Karpathy's "LLM Wiki" pattern): a persistent,
interlinked set of Markdown pages that compiles everything known about the design of the
`minilogue-xd` Rust library. Its purpose is to let an engineer (or an agent) **re-implement the
library in Zig following TigerStyle, with zero third-party dependencies**, without having to
re-derive the design from the Rust sources.

Read this file before creating, editing, or querying wiki pages.

## Three layers

| Layer | Location | Owner | Mutability |
|-------|----------|-------|------------|
| Raw sources | `crates/`, `docs/design/`, `docs/dev/`, `docs/wayfinder/`, `docs/korg-user-guide/`, the Korg MIDI Implementation PDF (rev 1.01, external) | Humans | Immutable — read, never edit from wiki work |
| Wiki | `docs/wiki/**` | LLM | LLM writes and maintains every page |
| Schema | `docs/wiki/AGENTS.md` (this file) | Human + LLM | Co-evolved; change deliberately |

## Directory layout

```text
docs/wiki/
  AGENTS.md      schema (this file)
  index.md       content catalog — every page, one line each, by category
  log.md         append-only chronological record of ingests, queries, lint passes
  overview.md    top-level synthesis: what the tool is, layers, data flows
  glossary.md    canonical terms; one definition each
  sources/       one summary page per raw source (or source family)
  protocol/      device/wire facts, independent of any implementation language
  design/        how the Rust tool is designed (types, state machines, flows, tests)
  zig/           how to re-implement in Zig with TigerStyle and zero dependencies
```

Category rule of thumb: if a fact would still be true for a C or Python implementation, it
belongs in `protocol/`. If it describes a decision made by the Rust crate, it belongs in
`design/`. If it prescribes what the Zig port should do, it belongs in `zig/` (or in the
"Zig / TigerStyle notes" section of a `protocol/` or `design/` page).

## Page format

Every page starts with YAML front matter, then a single H1, then a one-paragraph summary
blockquote.

```markdown
---
title: Korg 7-bit SysEx codec
category: protocol            # protocol | design | zig | source | meta
sources:                      # raw sources this page was compiled from (repo-relative)
  - crates/minilogue-xd/src/codec.rs
related:                      # wiki pages (wiki-relative paths)
  - protocol/sysex-framing.md
updated: 2026-10-11
---

# Korg 7-bit SysEx codec

> One paragraph: what this is, why it exists, and the single most important fact.
```

Recommended sections (use those that apply, in this order):

1. **Facts / Layout / Behavior** — tables, byte offsets, bit masks, state machines, algorithms.
   Complete tables beat prose. Re-implementation must be possible from the page alone.
2. **Invariants** — bullet list phrased so each bullet can become a Zig `assert`.
3. **Errata & gotchas** — where the Korg spec, the hardware, and the Rust code disagree.
4. **Test vectors** — exact bytes copied from Rust tests or real files, with citations.
5. **Rust design** — types, traits, ownership choices; what to keep and what to drop.
6. **Zig / TigerStyle notes** — static sizes, comptime tables, assertions, error sets,
   no allocation, options structs, bounded loops, naming.
7. **Open questions** — unknowns, suspected bugs, things to verify on hardware.
8. **Sources** — `path:line-range` citations into raw sources.

## Conventions

- **Links**: relative Markdown links between pages, e.g.
  `[7-bit codec](../protocol/korg-7bit-codec.md)`. Every page must link to at least one other
  page and be listed in `index.md`. No orphans.
- **Citations**: cite raw sources as repo-relative `path:start-end` (e.g.
  `crates/minilogue-xd/src/codec.rs:29-68`). Cite the line range that proves the claim.
- **Provenance tags**: when a fact's origin matters, tag it inline: **[spec]** (Korg MIDI
  Implementation rev 1.01), **[hw]** (observed on hardware / real `.mnlgxd*` files),
  **[rust]** (a choice made by this crate), **[community]** (workbench reference code).
  When sources conflict, **[hw]** wins over **[spec]**; record the conflict on
  [errata](protocol/errata.md).
- **Numbers**: hex bytes as `0x42`; byte sequences as `F0 42 30 00 01 51 …` (bare hex,
  space-separated) inside code spans or blocks. Offsets are decimal unless prefixed `0x`.
  Bit ranges are written `bits 7..0` (high..low).
- **Zig code**: short sketches only, marked "sketch"; they illustrate shape, not final API.
  Follow TigerStyle naming (snake_case, units last: `timeout_ms_max`).
- **Tone**: precise, declarative, present tense. Say *why* for every design decision.
- **No duplication**: one canonical page per fact; other pages link to it.

## TigerStyle digest (constraints for every Zig note)

The Zig port follows [TigerStyle](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md).
The constraints that shape this design most:

- **Zero dependencies** beyond the Zig toolchain and OS-provided MIDI APIs.
- **Static allocation**: no allocation after init; every buffer has a compile-time maximum.
- **Put a limit on everything**: every loop, queue, wait, and retry is bounded.
- **Assertions**: ≥2 per function; assert arguments, returns, invariants; pair assertions
  across write/read paths (e.g. assert a blob before encode and after decode); assert
  compile-time constant relationships (`comptime assert(program_size == 1024)`).
- **Explicitly sized integers** (`u7`, `u10`, `u14`, `u32`); avoid `usize` in data formats.
- **Simple control flow**: no recursion, functions ≤70 lines, push `if`s up and `for`s down.
- **Errors**: operating errors are Zig error sets and are always handled; programmer errors
  are assertions.
- **Don't react directly to external events**: poll input at the program's own pace.
- **Naming**: no abbreviations, units/qualifiers last, same-length related names
  (`source`/`target`).
- **Options structs** where arguments could be confused; pass options explicitly.

## Workflows

### Ingest (a new or changed raw source)

1. Read the source fully (for code: include its tests — they hold the test vectors).
2. Create or update its `sources/` summary page.
3. Update every affected `protocol/`, `design/`, and `zig/` page; flag contradictions on
   [errata](protocol/errata.md) instead of silently overwriting.
4. Update `index.md` and `glossary.md` if pages or terms were added.
5. Append a `log.md` entry.

### Query

1. Read `index.md` first, then drill into pages. Answer with citations to wiki pages.
2. If the answer is reusable (a comparison, a derived table, a decision), file it as a new
   page and log it.

### Lint

Check for: pages missing from `index.md`, broken relative links, orphan pages, facts
duplicated across pages, claims without citations, stale `updated` dates after source
changes, and terms used but absent from `glossary.md`. Log the pass.

## Log format

`log.md` is append-only. Each entry starts with a parseable header:

```markdown
## [YYYY-MM-DD] ingest | <source or topic>
## [YYYY-MM-DD] query | <question>
## [YYYY-MM-DD] lint | <scope>
```

`grep "^## \[" docs/wiki/log.md | tail -5` lists the latest entries.
