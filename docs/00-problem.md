# The problem, the notation, and the sleeve record

Every stage reads this first. It states the objective once, defines the words the other docs use, and
fixes the record a sleeve must publish. Change the contract here before changing any consumer.

## The objective

Maximise the expected long-run growth rate of wealth, net of trading cost, over positions that depend
only on what is observable when they are taken:

```
g(w) = E[ log(1 + w_t' r_{t+1}) ]  -  E[ cost(h_t - h_{t-1}) ]

subject to   worst one-session loss(h_t) <= D · NAV      D = 10%; this is what bounds leverage
             sum of initial margin(h_t)  <= M · NAV      M = 50%; broker feasibility with a buffer
             h_t whole contracts
```

- `r_{t+1}`: the sleeves' daily excess returns over the next session, per dollar of sleeve notional.
- `w_t`: fractional exposures, dollars of sleeve notional per dollar of NAV. The optimiser's output.
- `h_t`: whole contracts held. `E` maps holdings to exposures (constant for a future, the Greeks for an
  option). `h_t` is chosen by scoring integer candidates near `w*` on this same objective
  (`docs/03-optimiser.md`).
- `s_t`: the state the moments are conditioned on (variance forecasts; the variance premium if its test
  passes).
- `cost`: half-spread + all-in fee + the execution term from the MarketMaker record, per contract.

To second order, `g ≈ w'μ − ½ w'Σw`, maximised at `w* = Σ⁻¹μ`. **That is the sanity check, not the
solver.** The short-variance sleeve's left tail is invisible to it, so the solver maximises the log
expectation on a block bootstrap of the joint history.

V1 ships **half Kelly**: the solution at full log utility, scaled by `f = 0.5`. Overbetting is
asymmetric (at 2× Kelly growth is zero; at 0.5× it is three quarters of the maximum) and every `μ` is
uncertain.

## Where each input is identifiable

| input | identifiable at | estimated from | rule |
|---|---|---|---|
| mean of equity, duration | decades | French daily market factor (1926–); FRED yields (1962–) | constant; refit annually on the expanding window |
| mean of short variance | years, noisily | the sleeve's own fit-period net return | constant; a VRP-conditioned mean only through the stage-3 test |
| variance of anything | days | realised variance from intraday bars; EWMA / HAR-RV | conditional, daily |
| correlations | months | EWMA on vol-normalised sleeves | conditional, slow; the quadratic check only |
| joint tails | the sample's worst blocks | stationary block bootstrap of the joint history | non-parametric, never Gaussian |

The Kalman allocation (hourly allocation swung; daily and weekly converged to 60/40) and MarketMaker's
stage-3 filter (rejected on held-out R²) are the same lesson: returns identify variance, not means. Do
not ask a model for a conditional mean the data cannot supply at that horizon.

## The sleeve record

A **sleeve** is a daily series plus a fixed description, built by one code path from raw prices
(`CLAUDE.md`, rule 6). Per day `t`:

| field | type | meaning |
|---|---|---|
| `date` | date | the session the return accrues on |
| `ret_gross` | float | excess return per dollar of sleeve notional, before costs |
| `ret_net` | float | after the sleeve's own roll and hedge costs at unit exposure |
| `cost_turnover` | float | cost in return units of a one-notional round trip that day |
| `margin` | float | initial margin per dollar of notional that day |
| `exposure` | vector | index delta, duration (DV01), vega |
| `state` | vector | what the sleeve publishes for conditioning: its variance forecast, its signal if any |
| `flags` | bits | roll, missing settlement, holiday, short session, halted, any imputation |

The manifest fixes the construction (instrument, roll, hedge, sizing convention), the series read (by
catalogue id and sha256), the cost parameters with sources, the cross-check passed, and the git sha.

Rules:

1. **Unit exposure.** A sleeve does not size itself; the optimiser applies `w`.
2. **Excess returns.** Cash earns the risk-free rate outside the ledger.
3. **Point in time.** `state` on day `t` uses data through `t`'s settlement and nothing later. A stage-0
   test shifts every input by one day and asserts `state` does not change.
4. **No look-ahead in the roll.** Rolls happen on the manifest's calendar, at that day's settlement, and
   pay that day's cost.
5. **Determinism.** Rebuilding from the catalogue reproduces the file byte for byte.

## `decide()`: the one function both deliverables call

```
decide(date, store, holdings, params) -> Decision(targets, orders, diagnostics)
```

- `store` exposes data **through `date`'s settlement only**; a read past it raises. The backtest and the
  live walk pass the same store type, filled from history or from today's pull.
- `params` is the artifact in force on `date` (the most recent 1 July refit).
- `diagnostics` carries the fractional `w*`, the worst-session estimate, margin, and which constraint
  bound. The journal writes all of it.
- `decide()` does no I/O beyond `store`, has no clock, and is deterministic given its inputs. The same
  inputs give the same orders in the backtest, the rehearsal and the walk.

## Statistics

Every comparison reports, per arm: n years, mean ± SE across years, t, win rate against the control,
and the number of arms × metrics in the table. The backtest has five years (df = 4, 5% two-sided critical
t = 2.78). Monthly statistics (60 months) are reported beside the yearly ones and labelled as
autocorrelation-sensitive.
