# Status board

Update at the end of every session. Keep it terse: what changed, what the number was, what is next. A
result without its mean ± SE and n does not go in here.

**Now (2026-09-29):** the repo is a plan. Nothing is measured. Stage 0 (scaffold + ledger contract) is
next; stage 1's first act is to **register the split below before any stream is built**.

| Stage | State | Notes |
|---|---|---|
| 0 Scaffold + ledger contract | not started | |
| 1 Data + splits | not started | split proposed below, not yet registered |
| 2 Benchmarks with SEs | not started | |
| 3 Short-variance sleeve | not started | |
| 4 Carry sleeves | not started | |
| 5 Moments + optimiser | not started | |
| 6 Instruments, costs, replay | not started | |
| 7 Validation | not started | |
| 8 Forward walk | not started | |

## Decisions taken

- 2026-09-29 — **The project.** One growth-rate objective with costs over five sleeves (equity, duration,
  short variance, FX carry, commodity carry). Every earlier finding is a special case of it
  (`docs/00-problem.md`).
- 2026-09-29 — **Dropped:** the pairs leg and composition tilts. Execution stays in futures and index
  options.
- 2026-09-29 — **Added as risk sources:** the commodity term-structure premium and FX carry, both from
  CME futures already on the data plan. Commodity carry is measured in V1 and traded only on the breadth
  gate (`docs/04-carry.md`), because a $50k account can hold 3–5 commodity names and 3–5 names is
  idiosyncratic risk, not the premium.
- 2026-09-29 — **The short-variance stream is self-built** from option settlements with own costs, and
  cross-checked against Cboe PUT/BXM. Its mean is VRP-conditioned only if the held-out gate passes;
  the constant mean is the control (`docs/03-short-variance.md`).
- 2026-09-29 — **Python only for V1.** No C++; the rebalance is daily.
- 2026-09-29 — **Rules carried over from MarketMaker** (`CLAUDE.md`): real data only, net growth is the
  metric, paired by year with SEs, identifiability, pre-registered splits, one construction per sleeve,
  instruments chosen by the cost model.

## Pre-registrations

- **Proposed split, to be registered by stage 1 before any stream exists** (the GLBX era; the long
  equity and bond histories are used only for constant-mean priors, fixed before validate is opened):
  - **fit:** 2010-06-01 to 2017-12-31 (7.6 years; contains 2011-08, 2015-08, 2016-02).
  - **validate:** 2018-01-01 to 2021-12-31 (4 years; contains 2018-02 and 2020-03, the two vol events
    the short-variance sleeve must survive).
  - **sealed:** 2022-01-01 to the walk's start (contains 2022, the joint equity–bond drawdown 60/40 must
    survive). Opened once, at V1 sign-off.
  - Paired statistics are by calendar year; the validate t has df = 3 and that is stated wherever it is
    reported. **A gate with df = 3 needs a pre-stated effect size, not a t alone** (MarketMaker's
    stage 4b: a perfectly calibrated model fails a per-decile 2-SE rule 78% of the time at df = 3).

## Open questions

See `PLAN.md`. Add to this list as they are found; move them to Decisions when settled.

## Result log

_(newest first; one entry per measured result, with n, mean ± SE, t, and the test count)_

- 2026-09-29 — no measured results. The first number will be stage 1's catalogue (series, records, cost).
