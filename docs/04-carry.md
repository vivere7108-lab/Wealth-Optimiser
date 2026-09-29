# Stage 4 — The carry sleeves: FX and commodity term structure

**Deliverable:** the FX carry and commodity carry streams under the ledger contract, their marginal
Sharpe against the stage-2 portfolio on the fit split, and the breadth gate decided for commodity carry.

## What carry is, in one line

Carry is the return a position earns if prices do not change. For a future it is read off the curve:
`carry_t = (F_front − F_second) / F_second × (12 / months between expiries)`, positive in backwardation.
For an FX future the same calendar spread is the interest differential under covered interest parity,
so one definition serves both sleeves and one data source (GLBX settlements) feeds both. The
policy-rate series from FRED is a cross-check of the FX read, not its source.

Who pays: in commodities, hedging pressure (producers selling forward push curves into backwardation;
consumers hedging push them into contango). In FX, investors bearing crash risk in the high-rate
currency. Both premia pass the durability test; both have a left tail that arrives with equity
drawdowns, FX carry's more so (the carry unwind of 2008).

## FX carry

- **Universe:** the G10 crosses against USD with a CME future: EUR, GBP, JPY, AUD, CAD, CHF (and NZD
  if the micro exists). The `definition` file names what is listed; nothing is typed from a spec sheet.
- **Signal:** the annualised calendar-spread carry above, per currency, at the previous settlement.
- **Portfolio:** long the top half by carry, short the bottom half, each leg scaled to equal
  ex-ante vol, rebalanced monthly. Weekly rebalance is reported as an arm.
- **Mean:** the signal is the mean: `μ_i,t = β · carry_i,t`, with a single `β` fitted on the fit split
  across all currencies (a pooled regression of next-month excess return on carry). A `β` that is not
  positive with t ≥ 3 on the fit split closes the sleeve before validate is opened.
- **Instrument at $50k:** the micro FX futures (M6E, M6B, MJY, M6A; CAD and CHF micros where listed),
  each $7k–$15k notional, so a six-leg basket at one contract a leg is about $60k gross. Sized by the
  optimiser like every other sleeve.
- **Costs:** the FX row of the instrument table, per side, per leg, plus the monthly roll.

## Commodity carry, and why it is measured before it is traded

- **Universe:** energy (CL, NG), metals (GC, SI, HG), grains (ZC, ZW, ZS). Eight names, all on GLBX.
- **Signal, portfolio, mean:** as FX, sorted within the universe, long the most backwardated, short the
  most contango'd, equal ex-ante vol per leg, monthly.
- **The framing the user asked for** — "a risk source mimicking the S&P 500" — is read as: a diversified
  basket held for its premium, treated the way the equity index is (a constant scale on a quoted signal,
  a daily variance forecast), not as a single commodity bet. That reading needs breadth.
- **The breadth problem at $50k.** Micros exist for gold (MGC, ~$35k), silver (SIL, ~$35k), copper (MHG,
  ~$12k) and crude (MCL, ~$7k); natural gas has a micro where listed; the grains' minis are thinly
  traded and the full-size contracts are $25k–$50k each. A long/short basket of eight at one contract
  a leg is roughly $250k gross, five times the account, before the other sleeves. A basket of three is
  not the premium; it is idiosyncratic commodity risk with the premium's name on it.
- **Breadth gate (pre-registered here, decided on the fit split):** the sleeve is traded in V1 only if
  (a) at least six names can be held at one contract each with the sleeve's gross notional at or below
  1.5× NAV, and (b) the six-name basket's fit-split Sharpe is within one SE of the eight-name basket's.
  If either fails, the sleeve is **measured** (its stream stays in the ledger, its marginal Sharpe is
  reported every stage) and **not traded** until capital or listings change. This is the "drop it for
  V1" the user offered, made into a rule with numbers instead of a judgement.

## Marginal Sharpe: how a sleeve earns its place

For each carry sleeve, regress its net daily return on the stage-2 portfolio's (60/40 by risk weight,
vol-scaled). The **appraisal ratio**, alpha over residual vol, is the number that decides its weight,
not its standalone Sharpe. Report it with its SE, paired by year, on the fit split. A sleeve with an
appraisal ratio under one SE from zero is kept in the ledger and gets no capital.

## Gates

1. Each stream reproduces byte for byte, manifest complete, no look-ahead (the shift test).
2. FX: pooled `β > 0`, t ≥ 3 on fit; appraisal ratio reported with SE.
3. Commodity: the breadth gate decided and logged; appraisal ratio reported for the eight- and
   six-name baskets.
4. Both: worst month and its date, so the crash beta is visible next to the Sharpe.
5. Tests counted: 2 sleeves × 2 rebalance arms × 5 metrics.

## Not in this stage

- Momentum overlays on carry, curve-position choice (the "optimum yield" trick), seasonality. All V2.
- Any commodity outside the eight. Breadth is added when it can be held.

## Next action

Compute the calendar-spread carry for every name from the stage-1 settlements and plot its
distribution by year. If the FX carry read disagrees with the policy-rate differential by more than the
funding basis explains, the spread is being read off the wrong expiry pair.
