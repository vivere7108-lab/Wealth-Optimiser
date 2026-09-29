# Status board

Update at the end of every session. Keep it terse: what changed, what the number was, what is next. A
result without its mean ± SE and n does not go in here.

**Now (2026-09-29):** the plan has been rewritten around two deliverables: a forward-walkable `decide()`
and a five-year backtest of it (`PLAN.md`). Nothing is measured yet. Stage 0 (scaffold) is next. Stage 1's
first act is to **register the window below before any return is computed**.

| Stage | State | Notes |
|---|---|---|
| 0 Scaffold | not started | |
| 1 Data | not started | window proposed below, not yet registered |
| 2 Sleeves | not started | |
| 3 Optimiser + `decide()` | not started | |
| 4 Backtest (`bt-v1`) | not started | |
| 5 Forward walk | not started | |

## Decisions taken

- 2026-09-29 — **Two deliverables** (user): a forward-walkable algorithm that optimises across the chosen
  instruments, and a roughly five-year backtest of it. Both call one function, `decide()`
  (`docs/00-problem.md`). The earlier nine-stage plan is replaced by six stages.
- 2026-09-29 — **Three sleeves: equity, duration, short variance** (user). FX and commodity carry are
  dropped; the design is kept in `docs/v2-carry.md`. The straddle has one alternative construction, the
  defined-risk arm. The strangle and VIX-future arms, dealer gamma, the VRP-conditioned equity mean
  and the Kalman reproduction are cut from V1.
- 2026-09-29 — **Window** (user chose the recommended split): fit 2010-06-01 to 2021-06-30; backtest
  2021-07-01 to 2026-06-30; rehearsal 2026-07-01 onward. Parameters are refit each 1 July on an expanding
  window. The design is frozen at fit end.
- 2026-09-29 — **`D` = 10% of NAV per session** (user). `D` bounds leverage; margin is only a feasibility
  cap, `M` = 50% of NAV.
- 2026-09-29 — **Whole contracts are part of the problem.** Integer holdings are chosen by scoring
  candidates on the objective. The straddle's delta is netted into MES. Results are reported at integer
  $50k and fractional sizing, with the rounding drag (`docs/03-optimiser.md`).
- 2026-09-29 — **The forward walk is automated through the IBKR API** (user), reusing MarketMaker's TWS
  plumbing, with the guards in `docs/05-forward-walk.md`.
- 2026-09-29 — **Instruments ruled out at $50k:** SPX options (~$600k a contract); SPY on leverage
  (portfolio margin needs a six-figure minimum, and Reg T gives 2×); front-month VIX futures (have
  doubled in a day with margin near half the notional); ES until the rounding-drag curve says otherwise.
- 2026-09-29 — **Python 3.13 only.** The rebalance is daily.
- 2026-09-29 — **Rules carried over from MarketMaker** (`CLAUDE.md`).

## Pre-registrations

- **Window, proposed, to be registered by stage 1 before any return exists:** fit 2010-06-01 to
  2021-06-30; backtest 2021-07-01 to 2026-06-30; rehearsal 2026-07-01 to the walk's start. Refit each
  1 July on data through 30 June.
- **Backtest looks:** `bt-v1` is the first run on the frozen design. Every later run is numbered and
  logged here with its change and reason.
- **T1 (short-variance mean on VRP):** its threshold comes from a power calculation written here
  *before* the test is run (`docs/03-optimiser.md`).

## Open questions

See `PLAN.md` ("Open questions"): micro yield tracking, option bid–ask at settlement, T1's power, and
the settlement-vs-live-fill gap.

## Backtest looks

_(one line per run: id, date, what changed, why)_

## Result log

_(newest first; one entry per measured result, with n, mean ± SE, t, and the test count)_

- 2026-09-29 — no measured results. The first number will be stage 1's catalogue (series, records, cost).
