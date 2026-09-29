# Stage 5 — Moments and the optimiser

**Deliverable:** `artifacts/moments/v1` (the constant priors, the variance model, the correlation
model, the bootstrap specification) and `artifacts/policy/v1` (the sizing rule with its Kelly fraction,
leverage cap and loss cap), with the sanity checks passing and the equity-only case reproducing the
stage-2 vol-scaled arm.

## Moments

**Means, `μ(s)`.** By the identifiability rule (`docs/00-problem.md`):

| sleeve | shipped mean | control | conditioning tested |
|---|---|---|---|
| equity | `ERP_prior` (constant) | — | `a + b·VRP_t` at the quarterly horizon, pre-registered below |
| duration | constant prior | — | none in V1 |
| short variance | the arm stage 3's gate chose | constant | (decided in stage 3) |
| FX carry | `β · carry_t` | — | none: the signal is the mean |
| commodity carry | `β · carry_t` | — | none |

**Variances.** One realised-variance model for the index, shared by every sleeve that needs it:
HAR-RV (daily, weekly, monthly realised-variance averages) fitted on the fit split, with an EWMA as the
control. The chosen model is the one with the lower out-of-sample QLIKE loss on the last two years of
the fit split, and the choice is logged. Each sleeve's own variance is an EWMA of its net returns with
the index forecast as a covariate (`docs/03-short-variance.md` for the short-variance case).

**Correlations.** EWMA (half-life fitted on fit, reported at 3, 6 and 12 months) on vol-normalised
sleeve returns. Used in the quadratic sanity check and as the second-order term in the trading-rate
rule below. Not used to size tails.

**Tails.** A stationary block bootstrap (expected block length one month) of the joint daily net
returns of all sleeves on the fit split, 10,000 paths of one year. This is the distribution the solver
maximises over. It preserves the conditional correlation in drawdowns without a parametric model, and
its worst blocks are the fit split's own, which are milder than validate's (`docs/01-data.md`).

## The objective and the solver

```
w* = argmax_w  mean over bootstrap paths of  log(1 + w' r_path)  -  cost(w - w_prev, s)
     subject to   sum_i |w_i| * gross_i <= L,   quantile_q(worst day of the path) >= -D
```

- **Fractional Kelly.** Solve at full Kelly, ship `f · w*` with `f = 0.5`; report `f ∈ {0.25, 0.5, 0.75, 1}`.
  The reason is asymmetric: at `2 × w*` growth is zero, at `0.5 × w*` it is three quarters of the maximum.
- **Leverage cap `L`:** the broker's initial margin per contract on the day, summed, at or below 50% of
  NAV, so a one-day doubling of margin requirements does not force a liquidation.
- **Loss cap `D`:** the 1% quantile of the bootstrap's worst one-day loss must be no worse than 10% of NAV
  at the shipped weights. Both numbers are parameters of the artifact, not constants in code, and both
  are revisited when validate is opened, once.
- **Costs and the trading rate.** With quadratic costs the solution moves *toward* the target, not to it:
  `w_t = w_{t-1} + τ (aim_t − w_{t-1})`, where `aim_t` weights the current target against the expected
  future target by the signals' persistence (Garleanu and Pedersen 2013). A slow signal (vol scaling,
  carry) gets a large `τ`; a fast one gets a small `τ`, which is the formal statement of why intraday
  signals are not in V1. `τ` is fitted on the fit split from the cost model and the signals' measured
  autocorrelation, not chosen.
- **The quadratic as the check.** `Σ⁻¹μ` at the same `μ` and `Σ` is computed beside the bootstrap
  solution; where the two differ by more than the tails explain (the difference should be concentrated
  in short variance and FX carry), something is wrong.

## Sanity checks (all must pass)

1. **Equity only:** with the other sleeves switched off, the policy reproduces stage 2's vol-scaled arm
   to within the bootstrap's sampling error, day by day.
2. **Monotone in variance:** every sleeve's weight falls as its own variance forecast rises, holding
   the rest fixed.
3. **Monotone in cost:** raising a sleeve's `cost_turnover` lowers its `τ` and never raises its weight.
4. **Concavity:** the growth rate is concave in `f` with its maximum at `f = 1` on the bootstrap.
5. **The tails bind:** at the shipped weights the loss cap is the active constraint on at least one
   sleeve in the bootstrap, or the cap is loose enough to be meaningless and that is stated.

## Pre-registered tests in this stage (fit split; validate is opened once, in stage 7)

- **T1, equity mean:** constant `ERP_prior` against `a + b·VRP_t` on non-overlapping quarters of the fit
  split. The conditioned mean is adopted only if `b > 0`, t ≥ 3, and the out-of-sample R² on the last
  two fit years is positive. Low prior; one test.
- **T2, dealer gamma as a covariate:** the sign of daily dealer gamma (published, daily, from listed
  index option open interest) as a conditioning variable on (a) the index variance forecast's residual
  and (b) the short-variance sleeve's mean. Adopted for either only at t ≥ 3 on fit and a positive
  out-of-sample improvement on the last two fit years. Two tests. This is the whole of V1's use of gamma.
- **T3, the Kelly fraction:** growth rate of `f ∈ {0.25, 0.5, 0.75, 1}` on the fit bootstrap, with the
  worst-year drawdown of each. `f = 0.5` ships unless 0.75 beats it by more than one SE *and* its
  worst-year drawdown is inside `D`.

## Artifact and diagnostics

`artifacts/moments/v1.json` carries the priors with their SEs and sample, the variance model and its
parameters, the correlation half-life, and the bootstrap seed and block length. `artifacts/policy/v1.json`
carries `f`, `L`, `D`, `τ` per sleeve and the shipped weights per state bin. `wo check moments|policy`
validates both. A heatmap of the equity and short-variance weights over (index variance forecast × VRP)
is saved to `runs/policy/v1/policy.png` and looked at before any replay.

## Next action

Fit HAR-RV against the EWMA control on the stage-1 realised variance and log the QLIKE comparison. The
variance forecast is the one input every sleeve shares, so it is built and checked before the solver.
