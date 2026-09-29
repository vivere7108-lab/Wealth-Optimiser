# Stage 2 — The three sleeves

**Deliverable:** equity, duration and short-variance streams (plus the defined-risk arm) under the
sleeve record, each built from settlements with its own costs. The cross-checks pass, and fit-period
statistics are tabulated with SEs.

Everything in this stage is built and checked on **fit** (2010-06 to 2021-06). The builder also writes
the backtest years, because `decide()` needs them. No strategy P&L is computed on them here.

## Equity

- **Stream:** the front ES future, rolled on the manifest's calendar (the exchange roll date, eight days
  before expiry, at settlement), return per dollar of notional over cash.
- **Traded in:** MES. The stream is the same index at 1/10 the size. On the 2019–2021 overlap, MES and ES
  settlement returns are compared as a construction check.
- **Costs:** the MarketMaker ES row (fee and posting cost), scaled to MES by its own fee ($0.62 a side).
  Roll at the calendar spread's quoted width.

## Duration

- **Stream in fit:** the front ZN future, rolled before first notice day, on the manifest's calendar,
  return per dollar of notional over cash.
- **Traded in:** micro 10-year yield futures if they pass tracking. These are cash-settled on the yield
  at about $10 a basis point, so their P&L is `−DV01 × Δyield`, not a bond price return. On the overlap
  (2021 onward, a construction check under the date guard), regress daily micro-yield P&L on ZN
  returns at matched DV01. **Tracking passes** if the residual vol is under 20% of ZN's own vol and the
  mean residual is within 2 SE of zero. On failure, duration trades in ZN, and the whole-contract rule
  decides whether a ZN position is worth holding at $50k.
- **Costs:** the fee schedule's row for each, plus the quoted half-spread.

## Short variance

Built on **ES options** for history (mid-2010) and traded in **MES options** (listed 2020, the same
underlying at 1/10 the size).

- **Position:** short one at-the-money straddle (the strike nearest the forward) on the listed expiry
  with 20–45 days to expiry, opened at the roll day's settlement.
- **Roll:** at 7 DTE, or on the first day the held expiry is below 20 DTE and a later monthly is listed,
  whichever is first.
- **Hedge (the stream):** delta-neutral at each settlement with the front future, delta from Black-76 at
  the straddle's own implied vol. Hedge trades pay the ES cost row.
- **Hedge (the account):** a 1-lot MES straddle's delta runs between −1 and +1, which whole MES
  contracts cannot hedge on their own. The traded hedge is netted into the equity sleeve's MES position
  and rounded once (`docs/03-optimiser.md`). This stage reports the stream's extra variance at a
  residual delta of ±0.5 MES per straddle, so the optimiser knows that cost in advance.
- **Sizing convention:** constant vega per dollar of notional. The constant-notional version is
  reported beside it.
- **Costs:** half the option bid–ask at open and roll (from the stage-1 source), plus fees and the hedge
  trades.
- **Settlement handling:** a missing settlement is flagged, never interpolated. An in-the-money expiry
  is exercised into the future at settlement and closed the same day.

**Defined-risk arm:** the same straddle plus long 10-delta wings at the same expiry. Its worst session
is bounded by the strikes, so `D` holds by construction. Its cost is the skew premium the wings pay. The
backtest decides between the two on net growth under `D`.

## Cross-checks and sanity checks

1. **PUT rebuild.** Rebuild Cboe PUT from the same ES option settlements under its methodology (monthly
   ATM put, held to expiry, cash-collateralised). Pass: daily levels track the published series within
   a stated tolerance over the fit period. The residual's mean ± SE is reported, and every day beyond
   tolerance is explained. **Nothing built on these settlements is trusted until this passes.**
2. **P&L attribution** per day: theta + gamma term (−½ΓS²r²) + vega × ΔIV + residual. The residual's
   share of variance is under a stated bound. Over fit, the gamma term's mean has the sign of implied
   minus realised variance.
3. The straddle's daily variance rises with the equity variance forecast. Every month with a 3σ index
   move loses, by roughly what the prior close's Greeks imply.
4. Every stream rebuilds byte for byte, and the one-day shift test passes.

## Fit-period table

For each of the four streams: growth rate, Sharpe, worst day, worst month, turnover, cost share of
gross, by year with mean ± SE over the 11 fit years. The test count is 4 × 6 = 24.

## Next action

With stage 1's option settlements on disk, rebuild PUT for one calendar year and compare it to Cboe's
levels. If that does not track, nothing downstream can be trusted, and it is the first thing to know.
