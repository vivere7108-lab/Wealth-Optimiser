# V1 plan — Wealth Optimiser

**Goal of V1:** a daily-rebalanced portfolio of futures and index options on a $50k account, sized by a
growth-rate optimiser with a cost model, that (1) reproduces every earlier finding with standard errors on
one common sample, (2) beats buy-and-hold, 60/40 and vol-scaled equity on held-out data, paired by year,
by a margin stated with its t and its test count, and (3) in a forward walk realises the costs, exposures
and per-sleeve returns the replay predicted.

**One-line thesis:** every earlier experiment — buy and hold, 60/40, the Kalman allocation, vol-scaled
Kelly, short vol — computed the same operator, inverse covariance times conditional mean, with a different
choice of what was constant and what was conditional. V1 makes that choice explicit per input, by
identifiability, and adds the two things the experiments lacked: a cost model and a tail-aware sizing rule.

## The problem

```
maximise over w_t:   g = E[ log(1 + w_t' r_{t+1}) ] - cost(w_t - w_{t-1}, s_t)
subject to:          gross leverage <= L,   worst one-interval loss <= D
to second order:     g ~ w' mu(s) - 1/2 w' Sigma(s) w   so   w* = Sigma(s)^-1 mu(s)
```

`r` is the vector of the sleeves' daily excess returns, `s` the state they are conditioned on, `w` the
exposures. The three inputs are `mu(s)`, `Sigma(s)` and the cost of moving. Notation, the ledger contract
and the mapping of every earlier finding onto this problem are in `docs/00-problem.md`.

## The sleeves

| sleeve | premium, and who pays it | V1 instrument at $50k | `mu` treatment | `Sigma` treatment | V1 status |
|---|---|---|---|---|---|
| Equity index | equity premium; investors buying safety | MES, then ES above ~$150k | constant prior from the long history; a VRP-conditioned mean is tested in stage 5 against it | daily forecast from ES intraday realised variance | traded |
| Duration | term premium; the 60/40 diversifier | ZF or ZN | constant prior | daily forecast | traded |
| Short variance | variance premium; hedgers buying insurance | delta-hedged short straddle on MES options, 20–45 DTE, hedged with MES; a defined-risk variant and a back-month VIX future run as ledger arms | VRP-conditioned, gated against constant | own forecast plus joint bootstrap tails | traded if stage 3 passes |
| FX carry | interest differential; crash-risk bearing | micro FX futures, G10, long high carry / short low carry, carry read off the calendar spread | the carry signal is the mean, scaled by a constant fitted on the fit split | daily forecast; joint tails | traded at micro size if stage 4 passes |
| Commodity carry | term-structure premium (backwardation); hedging pressure | micro metals and energy where listed | the carry signal is the mean | daily forecast; joint tails | **measured** in V1; traded only if the breadth gate in `docs/04-carry.md` passes |

Dropped from V1 by decision (2026-09-29): the pairs leg and composition tilts. Execution therefore stays
in futures and index options.

## Pipeline

```
[DATA]  GLBX daily settlements + intraday bars (ES, ZF/ZN, MES options, FX, commodities)
        Ken French daily market factor; FRED rates; Cboe VIX, PUT, BXM
             |
             v
[LEDGER]  one contract per sleeve: daily excess return + cost + margin + exposure map
          equity | duration | short variance | FX carry | commodity carry
             |
             v
[MOMENTS]  mu(s): constant priors; VRP-conditioned where gated; carry signals
           Sigma(s): daily variance forecasts; EWMA correlations on vol-normalised sleeves
           tails: block bootstrap of the joint fit sample
             |
             v
[OPTIMISER]  max E[log(1 + w'r)] - cost;  fractional Kelly;  leverage cap L;  loss cap D
             |
             v
[INSTRUMENTS + EXECUTION]  exposures -> contracts at $50k; roll calendar; post-vs-cross rule
             |
             v
[REPLAY | FORWARD WALK]  the same journal from both
```

## Stack decision (settled 2026-09-29)

- **Python 3.13 only for V1.** The rebalance is daily, the heaviest computation is realised variance
  from 1-minute bars, and a block bootstrap of 20 years of daily returns is seconds. MarketMaker's C++
  book existed for 4 M events/s; nothing here needs it.
- **JSON configs and artifacts, each with a manifest** (git sha, fit period, fitted_at) and a hash of
  its inputs, so a run is reproducible from the catalogue.
- **The live path is decided in stage 8.** MarketMaker's TWS plumbing is the candidate; a daily
  rebalance may not need it.

## Stages

Each stage ends at a stated, checkable deliverable. Do not start a stage before its predecessor's
deliverable is checked in and recorded in `STATUS.md`.

| # | Stage | Deliverable | Doc | Est. |
|---|---|---|---|---|
| 0 | Scaffold + ledger contract | `research/` builds, `make py-test` and `make lint` run, the sleeve record and manifest contracts enforced by tests | `docs/00-problem.md` | 1 session |
| 1 | Data + splits | every series catalogued with sha256, counts and cost; **the fit / validate / sealed split registered in `STATUS.md` before any stream is built**; realised variance from ES bars | `docs/01-data.md` | 1–2 sessions + downloads |
| 2 | Benchmarks with SEs | buy-and-hold, 60/40 and vol-scaled equity on the ledger, paired by year, on the fit split; the earlier findings reproduced or the discrepancy explained | `docs/02-benchmarks.md` | 1 session |
| 3 | Short-variance sleeve | the delta-hedged straddle stream built from option settlements with own costs, cross-checked against Cboe PUT/BXM; the VRP-conditioned mean gated on validate; the tail arms | `docs/03-short-variance.md` | 2–3 sessions |
| 4 | Carry sleeves | FX and commodity carry streams; marginal Sharpe of each against the stage-2 portfolio; the breadth gate decided | `docs/04-carry.md` | 1–2 sessions |
| 5 | Moments + optimiser | `artifacts/moments/v1`, `artifacts/policy/v1`; sanity checks pass; the equity-only case reproduces vol-scaling | `docs/05-optimiser.md` | 2 sessions |
| 6 | Instruments, costs, replay | exposure map to contracts; the instrument table with sources; `wo replay` on the ledger with every cost; journal | `docs/06-instruments-execution.md` | 2 sessions |
| 7 | Validation | the full stack against the three benchmarks on validate, paired by year; marginal contribution of each sleeve; sealed untouched | `docs/07-validation.md` | 1 session |
| 8 | Forward walk | small size, daily report of realised vs predicted cost, exposure and per-sleeve return | `docs/06-instruments-execution.md` | ongoing |

Rough total before the walk: **12–15 working sessions**.

## What V1 deliberately does not include

1. The pairs leg and any composition tilt (user decision, 2026-09-29).
2. Single-name options, and market making of any kind (closed in MarketMaker).
3. Any intraday signal as a strategy. Dealer gamma enters only as a covariate test in stage 5, gated.
4. More than one live short-variance structure. The ledger picks one; the others stay as measured arms.
5. Commodity carry as a traded sleeve unless the breadth gate passes.
6. Anything learned online. Every parameter is fitted offline on the fit split and shipped as an artifact.
7. Tax modelling. Section 1256 treatment is noted in the instrument table, not optimised for.

## Open questions, to resolve in the stage that needs them

- **Is there a published S&P 500 delta-hedged straddle index?** None is known to this plan. Cboe's
  strategy benchmarks are PUT and BXM (unhedged, monthly) and VPD/VPN (VIX futures). Stage 3 builds the
  stream from option settlements and uses PUT/BXM as the cross-check; a vendor series, if found, is a
  second cross-check and never the fit data (rule 6).
- **How deep is MES-option history on GLBX?** MES options list from 2020. Stage 3 builds the *index* from
  ES options (history from mid-2010) with unit sizing and trades it in MES options; stage 1 confirms
  both depths with a priced request.
- **Where do VIX futures settlements come from?** Confirm whether the vendor carries CFE; Cboe's own
  historical files are the fallback. Needed only for the VIX-future arm of stage 3.
- **Constant or VRP-conditioned mean for equity?** The variance premium is one of the few documented
  predictors of index returns at quarterly horizons. Stage 5 tests it against the constant prior under
  the identifiability rule; low prior weight, pre-registered.
- **The loss cap D and the Kelly fraction.** Set in stage 5 from the bootstrap's worst intervals and the
  broker's margin, not chosen by taste. Half Kelly is the starting point.
