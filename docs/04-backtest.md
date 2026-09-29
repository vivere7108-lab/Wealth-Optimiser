# Stage 4 — The five-year backtest

**Deliverable:** `bt-v1`. `decide()` runs daily from 2021-07-01 to 2026-06-30 on the frozen design,
with refits on 2021-06-30 (the fit-end artifact), 2022-07-01, 2023-07-01, 2024-07-01 and 2025-07-01. The
report is committed under `runs/reports/` and its table is pasted into `STATUS.md`.

## How it runs

```
wo backtest --config configs/bt-v1.json --out runs/backtest/bt-v1
```

- A loop over sessions: `decide(date, store, holdings, params)` → fills → the next day's holdings. The
  store and `decide()` are the ones the live walk uses. The backtest adds only a fill model and a clock.
- **Fill model:** orders fill at that day's settlement, plus the cost model (half-spread + fee + the
  MarketMaker posting term; option legs at the stage-1 half-spread). Rolls pay the calendar spread's
  width. Stage 5 measures how far live fills drift from this. The backtest also reports a second
  column with every cost doubled, so it is visible how sensitive each result is to the cost model.
- **Account:** starts at $50k. NAV compounds; cash earns the French risk-free rate. Margin is checked
  daily against the broker's schedule, as recorded on a stated date.
- **Refits** happen on 1 July, reading data through 30 June, and each writes a dated artifact. The
  journal records which artifact was in force every day.
- The journal (`journal.jsonl`) has one line per day: state, fractional `w*`, integer holdings before
  and after, orders, fills, costs, margin, worst-session estimate, which constraint bound, and rounding
  residual.

## Arms (every one on the same store, costs, account and whole-contract rule)

1. **Buy and hold:** one MES per $30k of NAV, rounded; rolled quarterly.
2. **60/40:** 0.6 of NAV in MES and 0.4 in duration, rebalanced monthly. It is reported by dollar weight
   and by risk weight.
3. **Vol-scaled equity:** `decide()` with duration and short variance switched off.
4. **Full stack:** `decide()` as shipped.
5. **Full stack minus one:** each sleeve removed in turn. The difference from the full stack is that
   sleeve's marginal contribution.
6. **Straddle vs defined-risk:** the full stack with each short-variance construction.

Each arm is reported at **integer $50k** (the result) and **fractional** (the ledger's book). The gap is
the rounding drag.

## The report

Per arm, by calendar-year segment (2021H2 to 2026H1, five 12-month blocks from each 1 July), the table
shows mean ± SE, t and win rate against buy and hold, for:

- net growth rate (the metric);
- Sharpe, max drawdown and its dates, worst session, worst month;
- turnover in contracts per month, cost share of gross return, and net growth with costs doubled;
- how often `D` and `M` bound, and how often the straddle count was zero.

State the test count for the table. Five years is df = 4, so the 5% critical t is 2.78. **The report
describes behaviour; it does not claim outperformance.** Beside it: the monthly statistics, labelled as
autocorrelation-sensitive; an equity curve per arm; the event windows 2022, 2024-08 and 2025-04 shown day
by day; and the per-sleeve P&L attribution.

## Discipline

1. **Real data only.** Bootstrap paths size positions; they never appear in a performance number.
2. **Net growth first.** A change that raises gross return or Sharpe and lowers net growth is rejected.
   A change that raises turnover must show it pays for itself in the cost model's own numbers.
3. **Looks are counted.** `bt-v1` is the first run on the frozen design. A later run after any change is
   `bt-v2` and so on. Each is logged in `STATUS.md` with the change and its reason, and every report
   states the number of looks. Changing the design because of a backtest result is allowed, but it
   is logged, and the new run is no longer out of sample for that change.
4. **Reproducible.** A run is fixed by the config, the catalogue hashes and the artifact hashes, and
   re-running it gives an identical journal.

## Gate to stage 5

Stage 5 does not start until all of these hold:

- `bt-v1` is committed with its report;
- the full stack's worst session in the backtest was inside `D`, or every breach is explained;
- the cost share of gross return is stated for every sleeve;
- the rehearsal replay (stage 5's first act) reproduces the backtest's last day exactly.

Whether the full stack beat the benchmarks is reported with its t, and it is not part of the gate. Five
years cannot decide it.
