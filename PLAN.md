# V1 plan — Wealth Optimiser

**V1 delivers two things:**

1. **A forward-walkable algorithm.** One daily function, `decide()`, that takes the data through
   today's settlement and the account's current holdings and returns target whole-contract holdings
   across three sleeves (equity, duration, short variance). It is sized by a growth-rate optimiser with
   a cost model and a per-session loss cap, and it places its own orders through the IBKR API.
2. **A five-year backtest of that same function**, 2021-07-01 to 2026-06-30, against buy-and-hold,
   60/40 and vol-scaled equity. It is reported by year with standard errors, costs, drawdowns and each
   sleeve's marginal contribution.

**Why the two are one piece of code:** the backtest calls `decide()` once a day over history, and the walk
calls it once a day live. A trade that exists in one exists in the other, so the walk's first job is
to confirm that the backtest's costs and exposures were real.

**Thesis:** every earlier experiment (buy and hold, 60/40, the Kalman allocation, vol-scaled Kelly, short
vol) computed the same operator, inverse covariance times mean, with a different choice of what was held
constant. V1 holds the means constant where the data cannot identify them, forecasts the variances
daily where it can, sizes on the joint tails rather than a Gaussian, and pays for every trade.

## The problem

```
maximise over w_t:   g = E[ log(1 + w_t' r_{t+1}) ] - cost(h_t - h_{t-1})
subject to:          worst one-session loss(h_t) <= D = 10% of NAV     (bounds leverage)
                     initial margin(h_t)         <= M = 50% of NAV     (broker feasibility)
                     h_t whole contracts, chosen near w_t by the same objective
to second order:     w* = Sigma(s)^-1 mu(s)      (the sanity check, not the solver)
```

`r` is the vector of the sleeves' daily excess returns, `w` the fractional exposures per dollar of NAV,
`h` the contracts actually held. Notation and the daily record every sleeve publishes are in
`docs/00-problem.md`.

## The sleeves

| sleeve | premium | traded in | mean | variance and tails |
|---|---|---|---|---|
| Equity | equity risk premium | MES | constant prior from the French library (1926 to fit end) | daily HAR-RV forecast from ES 1-minute bars |
| Duration | term premium | micro Treasury yield futures (10Y) if their tracking against ZN passes; else ZN | constant prior from FRED yields (1962 to fit end) | daily EWMA forecast |
| Short variance | variance risk premium | delta-hedged short ATM straddle on MES options, 20–45 DTE; the delta is netted into the MES position | constant; a VRP-conditioned mean replaces it only through one pre-registered test | own EWMA forecast; tails from a block bootstrap of the joint history |

A **defined-risk** variant of the straddle (long 10-delta wings) runs beside it as the one alternative
construction. Its worst session is bounded by the contract terms, which is the cheapest way to honour
`D`. The backtest decides between the two on net growth.

## The window (to be registered in `STATUS.md` by stage 1, before any return is computed)

| period | dates | use |
|---|---|---|
| fit | 2010-06-01 to 2021-06-30 | every design choice and every initial parameter; contains 2011-08, 2015-08, 2018-02, 2020-03 |
| backtest | 2021-07-01 to 2026-06-30 | five years, read with the configuration frozen; contains 2022 (joint equity–bond drawdown), 2024-08, 2025-04 |
| rehearsal | 2026-07-01 to the walk's start | the live code path run on recent days, paper only; the parity check before real orders |

**Refit rule:** parameters (priors, variance models, bootstrap sample) are refit every 1 July on all data
up to that date, the expanding window the live walk will use. The *design* (sleeves, constructions,
objective, `D`, `M`, Kelly fraction) is frozen at the end of fit and is not changed by what the backtest
shows without a logged new run (see "Looks" below).

**Why this split:** the fit period holds the two fastest vol crashes of the era, so the short-variance
sizing has seen them. The backtest holds 2022, which 60/40 must survive. MES options (listed 2020) and
micro yield futures (2021) trade throughout the backtest, so it runs on the contracts the account will
hold, with no hypothetical lot sizes.

**Looks:** the first full backtest run is `bt-v1`. Any later run after a change is `bt-v2`, `bt-v3`
and so on. Every run is logged with what changed, and the report states the number of looks. Five years
gives df = 4 by year; **the backtest can show behaviour (drawdowns, costs, turnover, marginal
contribution) with SEs, but it cannot prove outperformance, and no report claims it does.**

## Architecture

```
[DATA]      GLBX settlements + ES 1-min bars; French; FRED; Cboe VIX, PUT     (catalogued, hashed)
    |
[SLEEVES]   equity | duration | short variance (+ defined-risk arm)           (daily records, unit exposure)
    |
[MOMENTS]   constant means; daily variance forecasts; joint block bootstrap   (refit each 1 July)
    |
[OPTIMISER] half-Kelly log growth on the bootstrap, minus cost, under D and M -> fractional w*
    |
[CONTRACTS] w* -> whole contracts by the same objective; straddle delta netted into MES
    |
decide(date, data_through_close, holdings) -> target holdings      <- one function
    |                                   |
[BACKTEST] loops it, fills at        [LIVE] runs it after the 15:00 CT settlement,
settlement + cost model              places orders via IBKR, reconciles, reports
```

## Stages

Each stage ends at a checkable deliverable recorded in `STATUS.md`. Do not start a stage before its
predecessor's deliverable is checked in.

| # | Stage | Deliverable | Doc | Est. |
|---|---|---|---|---|
| 0 | Scaffold | `research/` builds; `make py-test`, `make lint`; the daily sleeve record, manifests and the date guard enforced by tests | `docs/00-problem.md` | 1 session |
| 1 | Data | every series catalogued with sha256, counts and cost; history depths confirmed; the window registered; ES realised variance | `docs/01-data.md` | 1–2 sessions + downloads |
| 2 | Sleeves | the three streams (and the defined-risk arm) built from settlements with own costs; the PUT rebuild cross-check; MES-vs-ES and micro-yield-vs-ZN tracking; fit-period stats with SEs | `docs/02-sleeves.md` | 2–3 sessions |
| 3 | Optimiser | moments, bootstrap, sizing under `D` and `M`, the whole-contract rule, `decide()`; sanity checks pass on fit | `docs/03-optimiser.md` | 2 sessions |
| 4 | Backtest | `bt-v1`: `decide()` daily over 2021-07 to 2026-06 against the three benchmarks; the report by year | `docs/04-backtest.md` | 1–2 sessions |
| 5 | Forward walk | the IBKR adapter, guards and daily report; rehearsal on paper, then live at reduced size | `docs/05-forward-walk.md` | 1–2 sessions + ongoing |

Rough total before live orders: **8–11 working sessions**.

## Not in V1

1. FX carry and commodity carry (dropped 2026-09-29; design kept in `docs/v2-carry.md`).
2. The pairs leg, composition tilts, single-name options, market making of any kind.
3. Intraday signals, dealer gamma, a VRP-conditioned *equity* mean, the Kalman reproduction.
4. Straddle arms beyond the defined-risk variant (strangle, VIX futures): V2.
5. Anything learned online. Parameters change only at the annual refit.
6. Tax modelling. Section 1256 treatment is noted, not optimised.

## Open questions, resolved in the stage named

- **Micro yield futures (stage 1–2).** Listed 2021, cash-settled on a yield at about $10 a basis point. Stage 1
  confirms the symbols and depth. Stage 2 measures their daily tracking against ZN. If tracking fails,
  duration trades in ZN, whose notional is ~2× NAV, and the whole-contract rule decides whether a ZN
  position is ever worth holding at $50k.
- **Option spreads at settlement (stage 1).** Whether `statistics` carries a bid–ask for ES/MES options.
  If not, a quote schema sampled near 15:00 CT is priced and pulled.
- **Gate power (stage 3).** The one conditional-mean test (VRP on the short-variance mean) has about 130
  non-overlapping months in fit. Its threshold is set from a power calculation written before it is run,
  not a bare t ≥ 3.
- **Settlement vs live fill (stage 5).** The backtest fills at settlement. Live orders go in
  after 15:00 CT, and Treasury futures settle at 14:00 CT. The gap is a cost the walk measures and the
  cost model must cover.
