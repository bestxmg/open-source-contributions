# PostgreSQL: Hashing a Parameterised ScalarArrayOpExpr

**Status:** Under review on `pgsql-hackers@lists.postgresql.org`. Posted
9 September 2026; reviewed by David Geier on 7 October; v2, split into a
refactoring patch and the feature as he suggested, on 11 October; **v3**
on 20 October, now awaiting further review.

## Motivation

Since PostgreSQL 14, the executor can evaluate `col = ANY (array)` with a
hash table instead of a linear scan — but only when the array is a
`Const`. Every parameterised form stayed on the O(rows × N) path:

- `col = ANY ($1)` — a `Param`, the shape of a cached/generic plan
- `col IN ($1, ..., $N)` — an `ArrayExpr` of `Params`
- `col = ANY ($1::int[])` — an `ArrayCoerceExpr`
- `col = ANY (string_to_array($1, ','))` — a `FuncExpr`

A prepared statement runs either a custom plan, built for each execution
with the actual values (so `$1` folds to a `Const` and gets hashed), or a
generic plan, built once and reused (so the array stays a `Param` and is
scanned linearly for every row). pgJDBC (after `prepareThreshold`), pgx,
asyncpg, psycopg 3 and PL/pgSQL all reuse prepared statements, and by
default PostgreSQL switches to the generic plan at the 6th execution if
it looks cheaper — which, with a long list, it often does, because the
planner assumes a `$1` array has 10 elements. That's why the slowdown
tends to appear only after a statement has run a few times. The original
PG 14 thread already wanted this case: Heikki Linnakangas wrote that "it
would be nice to handle queries like `WHERE column = ANY ($1)`"
([link](https://www.postgresql.org/message-id/557e8853-7cb8-9c01-6193-1d4109345401%40iki.fi)),
but it was left for later.

## The patch

v3 is two patches:

- **0001 — refactor, no behaviour change.** `finfo` becomes
  `fcinfo->flinfo`, `MIN_ARRAY_SIZE_FOR_HASHED_SAOP` moves to
  `primnodes.h`, and hash-table construction moves unchanged into a new
  `saop_build_hashtable()`. Split out at the reviewer's request so the
  feature diff stays readable. 5 files, +143/−129.
- **0002 — the feature and its tests.** Hashes any array argument the
  planner can prove is fixed for the whole execution: built only from
  `Const`s, external `Param`s and an allowlist of node types known to be
  stable. Anything else stays linear, including rare nodes that change
  from row to row (`merge_action()`, `RETURNING OLD/NEW`, set-returning
  functions). The executor evaluates such an array once on the first row,
  builds the hash table, and reuses it. `MIN_ARRAY_SIZE_FOR_HASHED_SAOP`
  is enforced against the run-time element count, and a `NULL`, empty or
  short array behaves exactly as on the linear path. PL/pgSQL simple
  expressions (reused with new parameter values) and JSON
  `ON ERROR`/`ON EMPTY` (where errors must be caught softly) keep the
  linear path. 6 source files plus ~370 lines of regression tests.

Not covered, because the array genuinely isn't fixed for the execution:
a `Var` (new array per row), a volatile function, an
aggregate/grouping/window value, a sub-select, or a `PARAM_EXEC`
(correlated / nestloop-inner). No new GUC, syntax, or catalog change.

## Measuring it

**A realistic query** — a 1M-row table filtered by a 1000-element list
and joined to a second table. Release build, default settings, median of
5:

| | custom plan | generic plan |
|---|---|---|
| master | 33 ms | 972 ms |
| patched | 31 ms | 30 ms |

On master the generic plan is ~30× slower than the custom plan; with the
patch the two are on par.

**`= ANY ($1)`** — `SELECT count(*) FROM t WHERE v = ANY ($1)`, 1M rows.
Release build, no LLVM, generic plan forced, warm cache, one Ryzen 7
4800H laptop, median of 9:

| N | master | patched |
|---|---|---|
| 6 | 59 ms | 57 ms (below threshold: linear both) |
| 9 | 65 ms | 50 ms |
| 40 | 150 ms | 52 ms |
| 100 | 297 ms | 52 ms |
| 1000 | 2494 ms | 51 ms |

**`IN ($1, ..., $N)`** — every value is its own `Param`, so master's
linear path is slower still. 1M rows, each value matching one row,
generic plan, no parallel workers, release build, median of 7:

| N | master | patched |
|---|---|---|
| 6 | 154 ms | 155 ms (below threshold: linear both) |
| 9 | 190 ms | 52 ms |
| 16 | 250 ms | 49 ms |
| 64 | 632 ms | 50 ms |
| 256 | 2078 ms | 51 ms |
| 1000 | 7734 ms | 50 ms |

Patched time is flat in N once hashed: **~49× faster** for `= ANY ($1)`
and **~155× faster** for `IN ($1, ..., $N)` at N=1000. No change on paths
that stay linear (N < 9, literal `IN`, pgbench).

**Single-execution overhead**, against what PG 14 already accepts for
`Const` arrays. `x = ANY (array)` run once, with `x` the first element
(the best case for a linear scan), generic plan, median of 3:

| N | literal array (master, already hashed) | `$1` on master (linear) | `$1` patched (hashed) |
|---|---|---|---|
| 9 | 2.8 µs | 2.8 µs | 3.0 µs |
| 100 | 5.1 µs | 2.8 µs | 4.8 µs |
| 1000 | 12.9 µs | 3.0 µs | 14.0 µs |
| 10000 | 159.2 µs | 5.1 µs | 171.3 µs |

A `$1` array costs slightly more than a literal one because it isn't
stored in the plan, so it's copied once before hashing. The worst case
stays just above what PG 14 accepted for `Const` arrays.

## Testing

- v2 fixed four bugs found in v1, each now covered by a regression test:
  `NOT IN` inside a PL/pgSQL simple expression compared with `=` instead
  of `<>`; arrays containing `merge_action()`, a set-returning function
  or `RETURNING OLD/NEW` of a view column were hashed once despite
  changing per row; `NULL = ANY ('{}')` returned `NULL` instead of
  `false`; and an error inside a JSON `DEFAULT ... ON ERROR` array wasn't
  handled softly.
- A debug build that checks every hashed result against a linear search,
  run over the whole regression suite.
- Differential testing against unpatched master — about 60,000 random
  query executions for v2 and 48,000 random queries for v3 — found no
  differences.
- The full test suite passes for 0001 alone and for both patches.

## Review

**David Geier** reviewed v1 on-list: "a pretty severe limitation that
would be great to remove." He asked for the refactorings to be split out
(done in v2), for `IN ($1, ..., $N)` numbers, and for evidence that
single-execution overhead is no worse than what PG 14 accepted for
`Const` arrays (both above). He also suggested that, as for `Const`
arrays, the choice to hash shouldn't depend on row estimates — agreed,
and kept that way.

On costing: `cost_qual_eval` can't know a bare `$1` array's length at
plan time, so it uses `estimate_array_length()`'s default of 10. For
`IN ($1, ..., $N)` the length is known, since it's an `ArrayExpr`; only a
bare `Param` or a function result falls back to the default. I proposed
leaving the costing as it is and documenting it.

## Alternative considered

Fixing this in the plan cache instead: preferring a custom plan in
`choose_custom_plan()` when the generic plan would search a long
parameter array. Rejected, because it only helps in auto mode (not with
`force_generic_plan` or for stable expressions) and it re-plans on every
execution, so the executor fix is the better one.

## Possible follow-ups (out of scope, noted for later)

- An uncorrelated `ARRAY(SELECT ...)` becomes an `InitPlan` whose
  `PARAM_EXEC` never changes — execution-stable in principle, only the
  planner needs to recognise it.
- A `PARAM_EXEC` array on a nestloop inner side is fixed per rescan, so it
  could be hashed once per rescan given a cost gate and rescan-aware
  invalidation of the cached table.

[Full post and thread →](https://www.mail-archive.com/pgsql-hackers@lists.postgresql.org/msg238384.html)
