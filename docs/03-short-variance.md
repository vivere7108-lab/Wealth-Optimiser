# Stage 3 — The short-variance sleeve

**Deliverable:** the sleeve's daily stream built from option settlements with its own costs, its
cross-check against Cboe PUT and BXM, the decision between a VRP-conditioned and a constant mean by a
pre-registered held-out gate, and the tail arms (defined-risk, VIX future) as measured alternatives.

This stage answers the open question "how are the mean and variance of the short-variance leg
decided?" The short answer: **the stream is built, not borrowed; the mean is conditioned on the
variance premium only if a held-out gate says so; the variance is forecast from the same realised
variance as the equity sleeve, and the tail comes from the joint bootstrap, never from a Gaussian.**

## Why not a published index

No S&P 500 delta-hedged straddle index is known to this plan as a published benchmark. Cboe publishes
PUT (monthly at-the-money put-write, cash-collateralised) and BXM (monthly buy-write), both unhedged,
and VPD/VPN (short VIX futures, capped). Academic delta-hedged straddle series exist (Bakshi and
Kapadia 2003; Coval and Shumway 2001) but are built on vendor option data at conventions we do not
trade. Under `CLAUDE.md` rule 6 the stream we fit must be the stream we trade, at our strikes, expiries,
hedge cadence and costs. A published index is therefore a **cross-check**: it validates the settlement
handling, the roll mechanics and the option pricing on the overlap, and nothing else.

## The construction (the traded arm)

Built on **ES options** for the index (history from mid-2010) at unit sizing, and traded in **MES options**
(1/10 the notional; listed from 2020). Stage 1 confirms both depths.

- **Position:** short one at-the-money straddle (the strike nearest the forward) on the ES option
  expiry with 20–45 days to expiry, opened at the settlement of the roll day.
- **Roll:** at 7 DTE, or on the first day the position's expiry falls below 20 DTE if a later monthly
  expiry is listed, whichever comes first. The roll calendar is in the manifest.
- **Hedge:** delta-neutral at each day's settlement with the front ES future, delta from the
  Black-76 implied vol of the straddle itself at settlement. Hedge trades pay the ES row of the cost
  model (posting, since a daily hedge is timing-indifferent).
- **Sizing convention:** constant vega per dollar of notional, so the stream's units are "return per
  dollar of index notional at one unit of vega"; the optimiser applies `w`. Report the constant-notional
  convention alongside.
- **Costs:** half the settlement bid–ask of each option leg at open and roll (from `statistics` where
  available, else a per-contract tick assumption stated in the manifest), plus fees from the schedule,
  plus the hedge trades.
- **Settlement handling:** a missing settlement is flagged, never interpolated; an expiry that settles
  in the money is exercised into the future at the settlement price and closed the same day.

Two more constructions run as **arms**, built the same way:

- **Strangle 25-delta:** short the 25-delta put and call. More premium per unit of tail from the skew;
  the arm the practitioners use.
- **Defined-risk:** the straddle plus long 10-delta wings. Its worst-day loss is a contract term, which
  makes the ruin constraint `D` enforceable by construction. Its cost is the skew premium the wings
  pay; that cost is measured here and priced as insurance in stage 5.
- **Back-month VIX future:** short the second-month VIX future, rolled monthly. A different premium
  (the futures basis and vol-of-vol) with a different tail; kept in the ledger as an alternative
  expression, not a substitute. Needs a settlement source (`docs/01-data.md`).

## Cross-check (the analogue of MarketMaker's mbp-10 check)

Rebuild Cboe's PUT and BXM from the same settlement data under their published methodology (monthly
at-the-money, held to expiry, cash-collateralised at the T-bill rate). Pass: the rebuilt daily levels
track the published ones within a stated tolerance over the overlap, with the residual's mean and SE
reported, and every day with a residual beyond the tolerance explained (a holiday, a settlement
convention, a dividend). This check must pass before the straddle stream is trusted. It does not test
the hedge, which no published index carries; the hedge is tested by the P&L attribution below.

## Sanity checks

- P&L attribution per day: theta − ½Γ S² r² (the gamma term) + vega × ΔIV + residual. The residual's
  share of variance is reported; a large residual is a hedge or a settlement bug.
- The gamma term's mean over the fit split is the realised variance premium in return units; its sign
  matches the sign of implied minus realised variance on the same days.
- The stream's daily variance rises with the equity variance forecast.
- A month with a 3σ index move loses; the loss's size matches what the Greeks at the prior close imply.

## How the mean is decided

Two arms, both fitted on the fit split only, compared on validate once:

- **A, constant:** `μ_SV` = the fit-split mean of `ret_net`, with its SE. The control.
- **B, VRP-conditioned:** `μ_SV,t = a + b · VRP_t`, where `VRP_t = IV_t² − E_t[RV²]`. `IV_t` is the
  straddle's own implied vol at the previous settlement (VIX² is reported as the alternative observable
  and the two are compared). `E_t[RV²]` is the HAR-RV forecast of stage 5's variance model over the
  option's remaining life, fitted on fit only. `a` and `b` are fitted on non-overlapping monthly
  returns of the stream on the fit split (about 90 months).

**Gate (validate, read once, 48 non-overlapping months):** B replaces A only if `b > 0` with t ≥ 3 on
the validate months, the out-of-sample R² of B's monthly forecast is positive, and B's growth rate at
half Kelly exceeds A's on validate, paired by year (df = 3; effect size pre-stated as at least the
control's own SE). Otherwise A ships. The variance premium is one of the few means quoted as a price,
so this is the identifiability rule's showcase; it still has to pass.

## How the variance and the tail are decided

- **Own variance:** an EWMA of the stream's squared daily returns (half-life fitted on fit), with the
  equity variance forecast as a covariate; report both the EWMA alone and the covariate model.
- **Covariances with the other sleeves:** the stage-5 EWMA on vol-normalised streams. On calm days the
  correlation with equity is small; in a drawdown it is not. **That conditional structure is why the
  quadratic form is not the solver.**
- **Tail:** the joint block bootstrap of stage 5 (monthly blocks of the joint daily returns on the fit
  split). The worst blocks the fit split holds are 2011-08, 2015-08 and 2016-02, which are milder than
  2018-02 and 2020-03 in validate. That asymmetry is stated in `docs/01-data.md`; it is the test.
- **Sizing:** stage 5 maximises the log expectation on the bootstrap, at half Kelly, under the loss cap
  `D`. The defined-risk arm is the one whose worst day is bounded by construction; the ledger decides
  whether its premium after the wings beats the naked straddle's premium after the loss cap.

## Gates (this stage)

1. The cross-check passes, tolerance and residual reported.
2. The P&L attribution residual is under a stated share of variance on the fit split.
3. The mean gate above is run once on validate, and the result is logged whichever way it goes.
4. The arms' fit-split statistics are tabulated with SEs: growth rate, Sharpe, worst day, worst month,
   turnover, cost share of gross. Tests counted: 4 arms × 6 metrics.

## Not in this stage

- The equity mean's own VRP conditioning (stage 5).
- Single-name or dispersion variance (not in V1).
- Any intraday hedge. The hedge is daily at settlement; a faster hedge is turnover, and MarketMaker's
  record says what turnover costs.

## Next action

With stage 1's option settlements on disk, rebuild PUT for one calendar year and compare to Cboe's
published levels. If the rebuild does not track, nothing built on these settlements can be trusted, and
that is the first thing to know.
