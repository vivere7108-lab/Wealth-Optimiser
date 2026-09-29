# Stage 2 — The three benchmarks, with standard errors

**Deliverable:** buy-and-hold, 60/40 and vol-scaled equity as sleeves on the ledger, with their growth
rate, Sharpe, max drawdown and turnover on the fit split, paired by year, and the earlier findings
either reproduced or the discrepancy explained.

## Why this stage exists

The earlier findings are the reason for the project and none of them carries a standard error yet.
A Sharpe of 0.8 measured over 20 years has an SE near 0.26; two strategies at 0.8 and 0.6 on the same
sample are not distinguishable without the paired difference. Every later stage is judged against these
three arms, so they are built first, on the same sample, with the same cost model.

## The arms

All three are expressed in the instruments V1 will trade, at unit exposure, with the cost model of
`docs/06-instruments-execution.md` applied to every rebalance and roll.

1. **Buy and hold:** `w_equity = 1`, held. Costs: the quarterly roll only.
2. **60/40:** `w_equity = 0.6`, `w_duration` set so that the bond leg's dollar exposure is 0.4 of NAV,
   rebalanced monthly. Report the 60/40 by *dollar* weight (the classic) and by *risk* weight (equal
   vol contribution) side by side; the second is what the optimiser will find, and the two differ.
3. **Vol-scaled equity:** `w_t = min(L, ERP_prior / (γ σ̂_t²))`, rebalanced daily, where `σ̂_t` is the
   stage-5 variance forecast and `ERP_prior` the constant from the long history. `γ = 2` (half Kelly)
   is the shipped value; `γ = 1` and `4` are reported.

Reported alongside, not gated: the Kalman allocation at daily and weekly steps, reproduced as
described (bond and equity returns as observables), to show its convergence to 60/40 on this sample
with SEs.

## The constant priors

- `ERP_prior`: the mean daily market excess return from the French library, 1926 to the fit split's
  end, annualised, with its SE. Report it also from 1963 and from 1990 so the sensitivity is visible.
  V1 ships the full-history value.
- `duration prior`: the mean excess return of a constant-maturity 10-year position from FRED yields,
  1962 to the fit split's end, with its SE.

These are written into `artifacts/moments/v1` by stage 5 and are not revisited after validate opens.

## Gates (fit split; this stage opens no held-out data)

1. Each arm reproduces from the catalogue byte for byte, and its manifest lists every series and hash.
2. The paired table exists: n years, mean ± SE and t of the growth-rate difference against buy and
   hold, win rate, and the test count (3 arms × 4 metrics = 12).
3. The earlier findings hold on this sample with SEs, or the discrepancy is written up:
   - 60/40 has a higher Sharpe than buy and hold;
   - vol-scaled beats buy and hold on growth rate at the same leverage cap.
   A finding that does not hold is not a failure of this stage; it is a result, and it goes in the log.

## Sanity checks

- The vol-scaled arm's exposure falls when the forecast rises, every day, by construction.
- Turnover of the vol-scaled arm is reported in contracts per month at $50k, so the cost model's
  weight in the result is visible before stage 6.
- The 60/40 by risk weight puts more than 40% of dollars in duration. If it does not, the bond return
  series is wrong.

## Next action

Build the equity and duration sleeves under the ledger contract first; the three arms are then policies
over two sleeves and a variance forecast.
