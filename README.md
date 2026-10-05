### Hi, I'm Linden 👋

Software engineer specialising in **C++ and database systems**, with 3+
years of professional experience.

![C++](https://img.shields.io/badge/C++-00599C?style=flat-square&logo=c%2B%2B&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![GCC](https://img.shields.io/badge/GCC-A42E2B?style=flat-square&logo=gnu&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)
![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white)

- 🔧 2 years building database kernel components — query processing, HTAP,
  columnar storage, and performance optimisation (Huawei, GaussDB)
- 🎓 Completing a Master of Information Technology at the University of
  Waikato, New Zealand
- 🧠 Interested in query engines, storage engines, and systems programming in C++
- 📍 Based in the Waikato, New Zealand — open to software engineering roles

#### Impact, at a glance

| | |
|---|---|
| **6.06%** faster `-fanalyzer`, 5-patch series merged | GCC, reviewed by GCC's analyzer maintainer → [details](./gcc-analyzer-move-semantics.md) |
| **Merged**, 118 files | openGauss kernel — public proof of the Expression Flattening project on my résumé → [details](./opengauss-expression-flattening.md) |
| **156 → 0** warnings, **~2.7×** faster CI | mealie, 3 PRs merged → [details](./mealie-testing.md) |
| **~49×** faster at N=1000 | PostgreSQL, parameterised `IN` / `= ANY` queries → [details](./postgresql-hashed-saop.md) |

#### Selected Open Source Contributions

**GCC — Reduce needless value copies in the analyzer and a few hot middle-end paths** · *merged*

5-patch series to `gcc-patches@gcc.gnu.org`, found by auditing the tree
with clang-tidy's `performance-*` checks for missed moves and needless
by-value parameters. Patch 1 gives `program_state` a move-assignment
operator and threads the move through `point_and_state` and
`exploded_node`, removing a deep copy of the whole region model on every
exploded-graph node — a measured **6.06% speedup** on `-fanalyzer`
(63.98s → 60.10s median, 8 interleaved runs, no overlap). Reviewed
positively by GCC analyzer maintainer **David Malcolm** and GCC
maintainer **Martin Jambor**, who pushed the full series on my behalf
(no commit access) — [`fe236f5bef7`](https://github.com/gcc-mirror/gcc/commit/fe236f5bef799694f693f4eb004634738cb1c059),
[`7532151`](https://github.com/gcc-mirror/gcc/commit/7532151),
[`89cb8a7`](https://github.com/gcc-mirror/gcc/commit/89cb8a7),
[`b3b1b80`](https://github.com/gcc-mirror/gcc/commit/b3b1b80),
[`d5cdcaa`](https://github.com/gcc-mirror/gcc/commit/d5cdcaa),
[`2633f49`](https://github.com/gcc-mirror/gcc/commit/2633f49).

→ [Full write-up](./gcc-analyzer-move-semantics.md)

**openGauss — [Flattening the expression evaluation framework](https://gitee.com/opengauss/openGauss-server/pulls/3121)** · *merged*

The public counterpart to the Expression Flattening Computation Framework
on my résumé from Huawei's GaussDB team. Flattens the expression tree once
at init instead of re-walking it recursively on every evaluation. Merged
to `openGauss/openGauss-server` master, March 2023, 118 files.

→ [Full write-up](./opengauss-expression-flattening.md)

**mealie — [Test suite reliability and speed](https://github.com/mealie-recipes/mealie)** · *3 PRs merged*

Eliminated all 156 pytest warnings by root-causing each one (not
suppressing them), locked that in by making warnings fail CI, then cut
the test suite from ~4 min to ~1.5 min with isolated parallel workers.

→ [Full write-up](./mealie-testing.md)

**PostgreSQL — [Hashing a parameterised ScalarArrayOpExpr](https://www.mail-archive.com/pgsql-hackers@lists.postgresql.org/msg238384.html)** · *work in progress*

Extended PG14's `col = ANY (array)` hash-table optimisation — previously
limited to constant arrays — to cover the parameterised forms real client
drivers actually emit (JDBC, psycopg, asyncpg bind parameters), which had
stayed on the linear-scan path since PG14. Up to **~49× faster** at
N=1000 (2494ms → 51ms), with the single-execution overhead the original
PG14 discussion worried about measured at worst +12µs. Posted to
`pgsql-hackers@lists.postgresql.org`, 9 September 2026.

→ [Full write-up](./postgresql-hashed-saop.md)

---

💼 Looking for **software engineering roles** (most experience in C++) — in New Zealand or remote.
📫 linden.lance.developer@gmail.com
