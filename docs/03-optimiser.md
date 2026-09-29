# Stage 3 — Moments, the optimiser, whole contracts, and `decide()`

**Deliverable:** `artifacts/moments/<refit-date>.json` and `artifacts/policy/<refit-date>.json` for
the fit-end refit (2021-06-30), `decide()` implemented and tested, and every sanity check below passing
on fit.

## Moments

**Means.** Constant, refit each 1 July on the expanding window:

| sleeve | mean | source |
|---|---|---|
| equity | `ERP_prior`: the French daily excess return, 1926 to the refit date, annualised, with SE | reported also from 1963 and 1990 for sensitivity; the full history ships |
| duration | the excess return of a constant-maturity 10-year position from FRED yields, 1962 to the refit date, with SE | |
| short variance | the sleeve's own net mean over its history to the refit date, with SE | shrunk halfway to zero (a stated prior: 11 years of one premium is thin) |

**The one conditional-mean test (T1).** The short-variance mean as `a + b·VRP_t`, where
`VRP_t = IV_t² − E_t[RV²]` over the option's remaining life, fitted on non-overlapping months of fit.
It replaces the constant only if `b > 0` at the threshold a power calculation sets **before the test is
run** (written into `STATUS.md` first), and its out-of-sample R² on the last three fit years is
positive. It is one test, run once, and logged whichever way it goes.

**Variances.** HAR-RV on ES realised variance, with EWMA as the control. The model with the lower QLIKE
on the last three fit years ships, and the choice is logged. Duration and short variance use an EWMA of
their own returns, with the index forecast as a covariate for short variance. Half-lives are fitted
on fit.

**Correlations.** EWMA on vol-normalised sleeve returns, used only in the quadratic check.

**Tails.** A stationary block bootstrap (expected block one month) of the joint daily net returns of the
three sleeves over the window to the refit date, 10,000 one-year paths. The fit window holds 2018-02 and
2020-03, so the solver has seen the fastest vol crashes of the era.

## The solver

```
w* = argmax_w  mean over paths of log(1 + w' r_path)
     subject to  q01(worst session on the path at w) >= -D,   margin(w) <= M · NAV
ship  f · w*,  f = 0.5
```

- `D = 10%` of NAV per session (user decision, 2026-09-29), checked at the 1% quantile of the
  bootstrap's worst session. It bounds leverage; there is no separate gross cap.
- `M = 50%` of NAV in initial margin, so that a one-day doubling of requirements does not force a
  liquidation.
- `D`, `M`, `f` and the quantile are artifact parameters, not constants in code. They are frozen at fit
  end.
- Between refits, `w*` moves daily only through the variance forecasts. The means are constant, so the
  shipped policy is a function of the state, `w*(σ̂_eq, σ̂_dur, σ̂_sv)`, tabulated on a grid at each refit
  and interpolated daily. This keeps `decide()` fast and deterministic.

## Whole contracts

At $50k, one contract carries a large share of NAV. The planning estimates below (index 6,000, ES vol
16%, 10-year yield vol about 90 bp a year) are replaced by measured values in stage 2:

| instrument | notional per contract, × NAV | annual vol per contract, % of NAV |
|---|---|---|
| MES | ~0.6× | ~10% |
| ZN | ~2.2× | ~13% |
| micro 10Y yield | DV01-equivalent ~0.3× | ~2% |
| 1 MES straddle, hedged | stage 2 | stage 2 |

One MES contract is roughly a whole sleeve's risk budget. Four rules turn `w*` into holdings:

1. **Choosing holdings.** Round `w*` to contracts, then score every holding within ±2 contracts of that
   per instrument (at most 125 candidates). The score is bootstrap log growth at the shipped `f`, minus
   the cost of trading from yesterday's holdings, subject to `D` and `M` on the integer book. Hold the
   best candidate. The no-trade band comes from this rule, not from a separate parameter.
2. **Netting.** The MES position is one number: the equity target delta minus the straddle book's delta,
   rounded once.
3. **The option leg is whole straddles.** Straddle count is part of the candidate search. Zero straddles is
   a legal answer, and the backtest reports how often it is chosen.
4. **Rounding is a cost that gets reported.** Stage 4 reports both the fractional and the integer book,
   and the difference per sleeve as rounding drag with SE. It also reports rounding drag at NAV $50k,
   $100k, $250k and $500k, which gives the MES-to-ES switch point and whether a sleeve is worth
   trading at $50k at all.

## `decide()`

```
decide(date, store, holdings, params) -> Decision(targets, orders, diagnostics)
```

Steps, in order: read the state through `date`'s settlement → variance forecasts → `w*` from the policy
grid → candidate search → orders as target minus holdings (option legs as whole straddles, with rolls
when the calendar says) → diagnostics (fractional `w*`, integer holdings, worst-session estimate,
margin, which constraint bound, rounding residual per sleeve).

The trading rate needs no separate `τ`. The integer search already charges the round trip, and with
constant means the only fast input is the variance forecast.

## Sanity checks (all must pass on fit)

1. **Equity only:** with the other sleeves off, `w*` equals the vol-scaled rule `ERP_prior / (γσ̂²)` at
   `γ = 1/f`, day by day, where `D` does not bind.
2. **Monotone in variance:** each sleeve's weight falls as its own variance forecast rises.
3. **Monotone in cost:** raising a sleeve's cost never raises its average holding or its turnover.
4. **Concavity:** *without* `D`, growth on the bootstrap is concave in `f` and peaks at `f = 1`.
5. **Does `D` bind?** Report how often `D` binds on the bootstrap at the shipped weights. If it never
   binds, say so: at 10% per session with half Kelly, it may be loose.
6. **Determinism:** `decide()` twice on the same inputs gives identical output. A store read past `date`
   raises.

## Artifacts

Moments carry the priors with SEs and samples, the variance models and parameters, and the bootstrap seed
and block length. The policy carries `f`, `D`, `M`, the quantile, the `w*` grid, and the candidate
search's radius. `wo check moments|policy PATH` validates both. A heatmap of `w*` over the equity and
short-variance forecasts goes to `runs/policy/<refit>/policy.png` and is inspected before stage 4.

## Next action

Fit HAR-RV against the EWMA control on the stage-1 realised variance and log the QLIKE comparison. Every
sleeve's sizing depends on it.
