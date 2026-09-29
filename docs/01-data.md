# Stage 1 — Data and the window

**Deliverable:** every series catalogued in `data/catalogue.json` with source, sha256, record count,
date range and cost; history depths confirmed; ES realised variance computed; **the window registered in
`STATUS.md` before any sleeve return is computed**.

## Sources

Price before pulling, every time, and record the price. On the GLBX plan the answer is $0.00; the
catalogue records that it was checked.

| series | source | schema / form | use | to confirm |
|---|---|---|---|---|
| ES, MES futures, all expiries | Databento GLBX.MDP3 | `definition`, `statistics` (settlement, OI), `ohlcv-1d` | equity sleeve; the straddle hedge | ES from mid-2010; MES from 2019-05 |
| ES 1-minute bars | Databento GLBX.MDP3 | `ohlcv-1m` | realised variance, HAR-RV | from mid-2010 |
| ES and MES options, all strikes and expiries | Databento GLBX.MDP3 | `definition`, `statistics`, `ohlcv-1d` | short-variance sleeve | ES options mid-2010; MES options 2020 |
| Option quotes near 15:00 CT, if `statistics` has no bid–ask | Databento GLBX.MDP3 | a quote schema, priced first | option half-spreads | whether it is needed at all |
| ZN, ZF futures | Databento GLBX.MDP3 | as ES | duration stream in fit; the fallback instrument | mid-2010 |
| Micro Treasury yield futures (10Y; 5YY as an alternative) | Databento GLBX.MDP3 | as ES | the traded duration instrument | listed 2021; symbols from `definition` |
| Daily market excess return, risk-free rate | Ken French library | CSV | equity prior; cash rate | 1926– |
| Treasury constant-maturity yields | FRED (DGS2, DGS5, DGS10) | CSV | duration prior | 1962– |
| VIX daily close | Cboe | CSV | the implied side of the VRP; a check on the straddle's own IV | 1990– |
| Cboe PUT index | Cboe | CSV | the stage-2 cross-check of option settlement handling | 1986– |

**Realised variance:** the sum of squared 5-minute log returns of the front ES contract over RTH, plus
the squared overnight return, per day. The 1-, 5- and 15-minute versions are reported side by side so the
sampling choice is visible.

## The window

Registered in `STATUS.md` by this stage, before any return is computed:

- **fit:** 2010-06-01 to 2021-06-30;
- **backtest:** 2021-07-01 to 2026-06-30;
- **rehearsal:** 2026-07-01 to the walk's start.

**The date guard.** Every runner takes a `--phase` (fit, backtest, rehearsal). A fit-phase runner raises
on any *return or performance* read past 2021-06-30, and there is no override flag. Two reads are
exempt, and are named in the guard's code rather than toggled: construction checks that compare prices
without computing a strategy's P&L (micro yield vs ZN tracking, MES vs ES settlement), and the annual
refit, which reads through its own 30 June.

## Catalogue rules

Carried over from MarketMaker's stage 1:

1. Every series has an id, a dataset/schema/symbol triple or URL, a date range, a record count, a sha256,
   and the request's cost.
2. `make data` prices, pulls, then hashes; a hash that changes on re-pull is an incident.
3. Writing the catalogue refuses to drop a series or change its schema without `--force` and a reason.
4. Missing settlements, holidays and short sessions are flagged, never imputed silently. A day with under
   40% of the median record count is flagged short.

## Next action

Price one day of each GLBX product (a `definition` and a `statistics` request each), confirm the depths
and the micro yield symbols, answer the option bid–ask question, and write the catalogue skeleton before
pulling anything in bulk.
