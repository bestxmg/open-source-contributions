# openGauss: Flattening the Expression Evaluation Framework

**Status:** Merged into `openGauss/openGauss-server` master, 13 March 2023
([PR #3121](https://gitee.com/opengauss/openGauss-server/pulls/3121)).
Reviewed and approved, CLA signed, CI green. 118 files changed.

This is the public, open-source counterpart to the **Expression Flattening
Computation Framework** project on my résumé from my time on Huawei's
GaussDB team — openGauss is Huawei's open-source community database,
sharing kernel lineage with GaussDB.

## The problem

The original expression evaluator handled operator trees recursively,
walking the tree on every evaluation. For expression-heavy analytical
workloads, that recursive walk repeats per row and per operator, and the
cost adds up across a query.

## The fix

Flatten the expression tree once, during expression-tree initialization,
instead of re-walking a recursive structure on every evaluation. This
moves the cost of traversal out of the hot per-row evaluation path and
into a one-time setup step.

## Outcome

Reviewed and approved (`lgtm` from two reviewers), CI pipeline green, CLA
signed, merged to master. Tracked against issue I6HRGJ, "Master
performance optimization." Built with a 3-person team; isolated A/B
benchmarking (same build, same hardware, only this commit differing)
showed a consistent ~10% speedup on expression-evaluation-heavy queries.

[View the merged PR →](https://gitee.com/opengauss/openGauss-server/pulls/3121)
