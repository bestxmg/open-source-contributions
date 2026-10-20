# GCC: Reducing Needless Value Copies in the Analyzer and Middle-End

**Status:** Posted to `gcc-patches@gcc.gnu.org`, 23 September 2026.
**Merged into GCC trunk as 9 commits**, the last three on 19 October
2026, with authorship preserved throughout
([all 9 on the GitHub mirror](https://github.com/gcc-mirror/gcc/commits?author=bestxmg)).
I don't have commit access, so GCC analyzer maintainer David Malcolm and
GCC maintainers Martin Jambor and Richard Biener pushed them on my
behalf. One small clarity fix to `gcov.cc`, split out of the original
patch 4, is under review as its own patch.

## Motivation

Move semantics are a part of C++ that experienced engineers still get
subtly wrong in large, long-lived codebases. Rather than review by hand, I
audited the GCC tree with clang-tidy's `performance-*` checks —
`performance-unnecessary-value-param`, `performance-move-const-arg`, and
related checks that trace whether a by-value parameter is ever mutated, or
whether a `std::move` target is actually reachable at the call site. That's
an exhaustive, per-parameter check that neither compiler warnings nor human
review are built to do consistently across a codebase this size.

## The series

Five patches — four are correctness/consistency fixes with no individual
performance claim, and one has a measured win:

1. **`analyzer: avoid deep-copying program_state when creating an
   exploded_node`** — `program_state` owns a whole `region_model`. Every
   time the analyzer's exploded-graph engine created a new `exploded_node`,
   it was deep-copying that `program_state` by value. Gave `program_state`
   a move-assignment operator and threaded the move through
   `point_and_state` and `exploded_node`, so the copy only happens where a
   real copy is actually needed.
2. **`analyzer, diagnostics: add missing std::move for by-value sinks`**
3. **`diagnostics: take HTML tag names as const char * in source-printing`**
4. **`gcc, analyzer: drop std::move calls that have no effect`**
5. **`range-op, tree-ssa-ccp, fold-const, ipa-cp: bind wide_int / widest_int
   by reference`**

17 files changed, 101 insertions, 48 deletions across `gcc/analyzer`,
`gcc/diagnostics`, and several middle-end passes (`range-op`,
`tree-ssa-ccp`, `fold-const`, `ipa-cp`).

## Measuring it properly

For patch 1, a single timing run isn't trustworthy. `cc1 -fanalyzer` was run
on `libiberty/cp-demangle.c` 8 times, alternating between the patched and
unpatched binary on every run — not all of one binary's trials first — to
cancel out drift from thermal throttling, cache state, and background load.
Median time dropped from 63.98s to 60.10s, a **6.06% improvement**, with no
overlap between the two sets across all 8 pairs.

The middle-end patches (`range-op`, `tree-ssa-ccp`, `fold-const`) have no
single headline number, so I checked that they sit on a hot path instead:
I wrote a test file heavy in division, xor and min/max comparisons — the
operations that reach these functions — and traced a real build compiling
it. That one file hit them tens of thousands of times.

## Testing

- `x86_64-pc-linux-gnu`, `--enable-languages=c,c++,lto`, `--disable-bootstrap`
- `check-gcc analyzer.exp/tree-ssa.exp/ipa.exp` on trunk (`492fbdbf99d`):
  16820+10537 pass, with the same 2 pre-existing failures as unpatched
  trunk — an unrelated `-Wanalyzer-symbol-too-complex` depth boundary and a
  known `ivopts` regression — i.e. the series introduces no regressions.
- A later full bootstrap (`--enable-checking=yes`) also completed cleanly.
- For the revised middle-end patches: bootstrapped on x86_64, and every
  `gcc.dg/tree-ssa` and `g++.dg/tree-ssa` test compiles to byte-identical
  assembly at `-O2` with and without them — no change in generated code.

## Review, and iterating on it

- **David Malcolm** (GCC analyzer maintainer): "nice to see a measurable
  performance win in the analyzer for this." Patches 1–3: "looks good to
  me." On patch 4's `gcc/text-art/canvas.cc` hunk, he suggested a cleaner
  fix — add an `array2::set` overload taking an rvalue `element_t&&`
  instead of relying on a `std::move` that wasn't actually moving anything,
  since `styled_unichar` is non-trivial to copy.
- **Martin Jambor** (GCC, SUSE): approved the `ipa-cp` hunks and offered to
  push patches on my behalf, since I don't have commit access.
- **Follow-up (26 Sept):** implemented David's suggested `array2::set`
  overload as a new patch, dropped the hunk he couldn't speak to
  (`gcov.cc`), and split the approved `ipa-cp` change into its own patch
  for Martin to land separately. Re-bootstrapped clean.
- **Merged (29 Sept):** Martin committed the `ipa-cp` patch —
  [`fe236f5bef7`](https://github.com/gcc-mirror/gcc/commit/fe236f5bef799694f693f4eb004634738cb1c059),
  authorship preserved (`Author: linden <bestxmg@gmail.com>`), 2 files
  changed (`gcc/ipa-cp.cc`, `gcc/ipa-cp.h`).
- **Merged (5 Oct):** David Malcolm committed patches 1–4 (minus the
  `gcov.cc` hunk) and the `array2::set` follow-up:
  - [`7532151`](https://github.com/gcc-mirror/gcc/commit/7532151) ---
    `analyzer: avoid deep-copying program_state when creating an
    exploded_node` --- the measured 6.06% `-fanalyzer` speedup patch.
  - [`89cb8a7`](https://github.com/gcc-mirror/gcc/commit/89cb8a7) ---
    `analyzer, diagnostics: add missing std::move for by-value sinks`
  - [`b3b1b80`](https://github.com/gcc-mirror/gcc/commit/b3b1b80) ---
    `diagnostics: take HTML tag names as const char * in source-printing`
  - [`d5cdcaa`](https://github.com/gcc-mirror/gcc/commit/d5cdcaa) ---
    `gcc, analyzer: drop std::move calls that have no effect`
  - [`2633f49`](https://github.com/gcc-mirror/gcc/commit/2633f49) ---
    `text-art: add array2::set overload taking an rvalue element` --- the
    cleaner fix David suggested for patch 4's `canvas.cc` hunk.
- **Splitting out the rest (9–13 Oct):** what remained — the `range-op`,
  `tree-ssa-ccp` and `fold-const` half of patch 5, plus the `gcov.cc`
  hunk — is middle-end code outside David's area. He and Martin both
  suggested re-sending each change as its own patch to that area's
  maintainers. On the combined version, Richard Biener asked why it kept
  const-reference aliases rather than using the originals directly, and
  Andrew MacLeod suggested simply renaming the parameters.
- **Revised and re-sent (18 Oct):** four standalone patches taking both
  suggestions — `range-op` renames the parameters, and `tree-ssa-ccp`
  drops the local entirely.
- **Merged (19 Oct):** Richard Biener approved three of them ("OK and
  will do") and pushed them the same day:
  - [`91e9cee`](https://github.com/gcc-mirror/gcc/commit/91e9ceecaa124cb2b38ca8db798055e461f4aa5a) ---
    `range-op: avoid copying the bounds in operator_div::wi_fold`
  - [`3ecf5b7`](https://github.com/gcc-mirror/gcc/commit/3ecf5b73c7f1104441dee2d591e1cb47785f3fa7) ---
    `tree-ssa-ccp: avoid copying lattice values and shift counts`
  - [`3a05514`](https://github.com/gcc-mirror/gcc/commit/3a055144c656454b75fda197d0bd5fd86e401a52) ---
    `fold-const: take wide_int arguments by const reference`

  The `gcov` patch is still awaiting review.

[Cover letter and full review thread →](https://gcc.gnu.org/pipermail/gcc-patches/2026-September/732435.html)
