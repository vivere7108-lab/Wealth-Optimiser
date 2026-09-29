# Stage 7 — Validation discipline

The predecessor projects produced, across three weeks and then two days, one durable result each,
and everything else was killed by one of the rules below. The market-making project's own value was
that its five negatives were clean. This file is the checklist that keeps this project's results clean.

## The rules

1. **Real data only.** No synthetic path in any performance number. Bootstrap resamples of real returns
   are for sizing and tails, and they are labelled as such wherever they appear.
2. **Paired by year, with SEs and the win rate.** Report mean ± SE and t across years, and the share
   of years the arm beat its control. A pooled total hides an arm that won two years of ten.
3. **Count the tests.** State the number of arms × metrics next to every table. Five arms × six
   metrics is thirty tests and about one and a half spurious stars at t ≥ 2.
4. **Net growth first, everything else second.** A change that raises gross return or Sharpe and
   lowers net growth is rejected by default. A change that raises turnover must show it pays for itself
   in the cost model's own numbers.
5. **The identifiability rule.** A conditional mean ships only beside its constant control and only
   through a pre-registered held-out gate with a pre-stated effect size. df = 3 on validate means a t
   alone proves nothing.
6. **Sealed period.** 2022 onward is read exactly once, by the sign-off script, at the end of V1.

## Arms every comparison runs

- **buy and hold** — the number to beat.
- **60/40** — by dollar weight and by risk weight.
- **vol-scaled equity** — the strongest single-sleeve arm.
- **full stack** — every traded sleeve, the shipped policy.
- **full stack minus one** — the full stack with each sleeve removed in turn. The difference is that
  sleeve's marginal contribution, and it is the number that says whether the sleeve earned its place.

## What V1 must be able to state at sign-off

Written as the claims, so it is obvious when one is unsupported:

1. Every sleeve rebuilds from the catalogue byte for byte, and its cross-check passed (short variance
   against PUT/BXM; FX carry against the policy-rate differential).
2. Every conditional mean that shipped passed its held-out gate; every one that failed shipped its
   constant control, and both are logged.
3. The full stack's net growth rate exceeds each benchmark's on validate, paired by year, with t and win
   rate reported, and the marginal contribution of each sleeve is reported with its SE.
4. The sealed period reproduces (3) without refitting anything.
5. The forward walk's realised costs and exposures sit inside the replay's predicted bands.
6. **Absolute return is not claimed from replay.** Four validate years and four sealed years decide the
   sign of nothing; the walk's job is to test costs and exposures, and the claim about the return is
   that it was computed honestly.

## Reporting format

One markdown table per comparison, committed under `runs/reports/<date>-<topic>.md`, with: arm,
n years, metric, mean ± SE, t, win rate, and the test count for the whole table. Paste the same table
into `STATUS.md` under the stage.
