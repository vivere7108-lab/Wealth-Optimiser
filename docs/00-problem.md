# The problem, the notation, and the ledger contract

This is the document every stage reads first. It states the objective once, defines the words the
other docs use, and fixes the contract a sleeve must honour to enter the ledger. Change the contract
here before changing any consumer.

## The objective

Maximise the expected long-run growth rate of wealth, net of the cost of trading, over positions that
depend only on what is observable when they are taken:

```
g(w) = E[ log(1 + w_t' r_{t+1}) ]  -  E[ cost(w_t - w_{t-1}, s_t) ]

subject to   sum_i |w_{t,i}| * gross_i  <= L        (leverage, from margin with a buffer)
             worst one-interval loss(w_t) <= D       (ruin constraint, from the bootstrap)
```

- `r_{t+1}` is the vector of the sleeves' daily excess returns over the next interval, in units of "per
  dollar of the sleeve's notional".
- `w_t` is the vector of exposures, in dollars of sleeve notional per dollar of net asset value.
- `s_t` is the state: everything the moments are conditioned on (variance forecasts, the variance
  premium, carry signals, and any gated covariate).
- `cost` is the cost of moving from `w_{t-1}` to `w_t`: half-spreads, fees, and the market-impact and
  adverse-selection terms the MarketMaker record measured.

To second order in `w`, `g ≈ w'μ(s) − ½ w'Σ(s) w`, whose maximiser is `w* = Σ(s)⁻¹ μ(s)`. **That
quadratic is the sanity check, not the solver.** Two sleeves (short variance, FX carry) have left tails
the quadratic cannot see; the solver maximises the log expectation on a block bootstrap of the joint
history (`docs/05-optimiser.md`).

Log utility is full Kelly. V1 ships **fractional Kelly**, starting at one half, because the penalty for
overbetting is asymmetric (twice the Kelly fraction gives zero growth) and every `μ` is uncertain.

## The three inputs, and where each is identifiable

| input | identifiable at | estimated from | rule |
|---|---|---|---|
| mean of a broad asset class (equity, duration) | decades, or a structural prior | the long history (Ken French daily market factor from 1926; FRED yields from 1962) | constant within V1; fixed before validate is opened |
| variance of anything | days | realised variance from intraday bars, EWMA / HAR forecasts | conditional, daily |
| variance premium | the day it is quoted | implied variance minus a realised-variance forecast | conditional; gated against constant |
| carry (FX, commodity) | the day it is quoted | the futures calendar spread | the signal *is* the mean, scaled by one fitted constant |
| correlations | months | EWMA on vol-normalised sleeves | conditional, slow |
| joint tails | the sample's worst blocks | block bootstrap of the joint fit history | non-parametric; never Gaussian |

The Kalman allocation experiment (hourly allocation swung, daily and weekly converged to 60/40) and the
MarketMaker stage-3 filter (rejected on held-out R²) are the same lesson: a filter returns what its
observable identifies, and returns identify variance, not means. Do not ask a model for a conditional
mean the data cannot supply at that horizon.

## Every earlier finding as a special case

| finding | what it held fixed or conditional | reading in this frame |
|---|---|---|
| Buy and hold | `μ`, `Σ` constant, `w = 1` | the one-asset baseline, not optimised |
| 60/40 | `μ`, `Σ` constant, two assets | the tangency portfolio; leverage on it beats leverage on equity (two-fund separation) |
| Kalman allocation | tried `μ_t`, `Σ_t` from returns | `μ_t` unidentifiable, so it collapsed to 60/40: correct |
| Vol-scaled Kelly | `μ` constant, `σ_t` forecast | `w_t = μ / (γ σ_t²)`: the Kalman with the unidentifiable half removed |
| Short vol | a new asset with an observable conditional mean | needs the log expectation on the joint tail, not the quadratic |
| Pairs | a new asset with a state-dependent mean | dropped from V1 |
| Momentum vs reversal | conditions `μ` on past returns | monthly horizon only; sub-second closed |
| Dealer gamma | conditions autocorrelation and `σ_t` | a covariate test in stage 5, not a sleeve |
| Microprice, volume | a mean identifiable at 100 ms to 1 s | below the cost of acting; enters the cost model |
| Market making | being the cost term for others | closed; at retail latency you pay the adverse selection |

## The ledger contract

A **sleeve** is a daily series plus a fixed description. Nothing enters the optimiser that is not a
sleeve, and every sleeve is built by one code path from raw prices (`CLAUDE.md`, rule 6).

Per day `t`, one record:

| field | type | meaning |
|---|---|---|
| `date` | date | the trading day the return accrues on |
| `ret_gross` | float | excess return per dollar of sleeve notional, before costs |
| `ret_net` | float | after the sleeve's own trading and roll costs at unit exposure |
| `cost_turnover` | float | cost in return units of a one-notional round trip that day (half-spread + fees + the execution-model term) |
| `margin` | float | initial margin per dollar of notional, from the instrument table, on that day |
| `exposure` | vector | the sleeve's map to risk factors: index delta, duration, vega, carry basket weights |
| `state` | vector | the conditioning variables the sleeve publishes: its variance forecast, and its signal where it has one |
| `flags` | bits | roll day, missing settlement, holiday, halted, any imputation |

Fixed per sleeve, in its manifest:

- the construction (instrument, roll rule, hedge rule, sizing convention, expressed as a parameter set);
- the data series it reads, by catalogue id and sha256;
- the cost parameters, each with a source;
- the cross-check it passed (index name, overlap period, tolerance, result);
- the git sha and `fitted_at` of the build.

Rules:

1. **Unit exposure.** Returns are per dollar of notional; the optimiser applies `w`. A sleeve does not
   size itself.
2. **Excess returns.** Cash earns the risk-free rate outside the ledger; every sleeve is over cash.
3. **Point-in-time state.** `state` on day `t` uses data through `t`'s close and nothing later.
   A test in stage 0 shifts every series by one day and asserts the state does not change.
4. **No look-ahead in the roll.** A roll happens on the calendar the manifest states, at that day's
   settlement, and pays that day's `cost_turnover`.
5. **Determinism.** Rebuilding a sleeve from the catalogue reproduces its file byte for byte, and the
   manifest records the hash.

## Paired statistics

Every comparison reports, per arm: n years, mean ± SE across years, t, win rate against the control,
and the number of arms × metrics in the table. The validate period has 4 years, so `df = 3` and the 5%
two-sided critical value is 3.18. A gate on validate states its effect size in advance; a t alone is
not a gate at df = 3.
