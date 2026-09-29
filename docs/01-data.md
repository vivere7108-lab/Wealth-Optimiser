# Stage 1 — Data and splits

**Deliverable:** every series catalogued in `data/catalogue.json` with source, sha256, record count,
date range and cost; realised variance of ES computed from intraday bars; **the split registered in
`STATUS.md` before the first sleeve is built**.

## Sources

Price before pulling, every time, and record the price. On the GLBX plan the answer is $0.00, and the
catalogue records that it was checked rather than assumed.

| series | source | schema / form | use | history to confirm |
|---|---|---|---|---|
| ES, MES futures, all expiries | Databento GLBX.MDP3 | `definition`, `statistics` (settlement, OI), `ohlcv-1d` | equity sleeve; hedge leg of short variance | GLBX from mid-2010 (MBO only from 2017, not needed) |
| ES 1-minute bars | Databento GLBX.MDP3 | `ohlcv-1m` | realised variance, the HAR-RV forecast | as above |
| ES and MES options on futures, all strikes and expiries | Databento GLBX.MDP3 | `definition`, `statistics` (settlement), `ohlcv-1d` | short-variance sleeve | ES options from mid-2010; MES options from 2020 |
| ZF, ZN futures | Databento GLBX.MDP3 | as ES | duration sleeve | mid-2010 |
| FX futures: 6E, 6B, 6J, 6A, 6C, 6S and their micros | Databento GLBX.MDP3 | as ES, front and second expiry | FX carry; the calendar spread is the carry | mid-2010 (micros later) |
| Commodities: CL, NG, GC, SI, HG, ZC, ZW, ZS, and micros where listed | Databento GLBX.MDP3 | as ES, front and second expiry | commodity carry | mid-2010 |
| Daily market excess return and risk-free rate, 1926– | Ken French data library | CSV | the equity constant-mean prior; the long benchmark history | complete |
| Treasury constant-maturity yields, 1962– | FRED (DGS2, DGS5, DGS10) | CSV | the duration prior; the pre-2010 bond return proxy | complete |
| VIX, daily close | Cboe | CSV | the implied side of the variance premium; a cross-check of the straddle's own implied vol | 1990– |
| Cboe PUT and BXM index levels | Cboe | CSV | the stage-3 cross-check | 1986– (backfilled) |
| VIX futures settlements | Cboe historical files, or the vendor if it carries CFE | CSV | the VIX-future arm of stage 3 only | 2004– |
| Central-bank policy rates, G10 | FRED / BIS | CSV | a cross-check of the FX carry read off the calendar spread | complete |

Realised variance: sum of squared 5-minute log returns of the front ES contract over the RTH session plus
the squared overnight return, per day. The 1-minute bars are the source; 5-minute sampling is the
choice that trades microstructure noise against sample size, and stage 1 reports the 1-, 5- and
15-minute versions side by side so the choice is visible.

## The split

Proposed in `STATUS.md` and registered there by this stage, before any sleeve exists:

- **fit:** 2010-06-01 to 2017-12-31;
- **validate:** 2018-01-01 to 2021-12-31;
- **sealed:** 2022-01-01 to the start of the forward walk.

Why these edges: validate holds the two fastest vol events of the era (2018-02, 2020-03), which are
what the short-variance sleeve must survive; sealed holds 2022, the joint equity–bond drawdown 60/40
must survive. The fit split's own stress days are 2011-08, 2015-08 and 2016-02, which are milder. That
asymmetry is deliberate and stated: **the tail the bootstrap sees in fitting is smaller than the tail
the gates test.**

The long histories (French, FRED) are used only to set constant-mean priors, and the priors are fixed
and written into `artifacts/moments/v1` before validate is opened.

## Catalogue rules

Carried over from MarketMaker's stage 1, where they were paid for:

1. Every series has an id, a source URL or dataset/schema/symbol triple, a date range, a record count, a
   sha256 of the file on disk, and the cost of the request.
2. `make data` prices, then pulls, then hashes; a file whose hash changes on re-pull is an incident.
3. Writing the catalogue refuses to drop a series or change a series' schema without `--force` and a
   reason (MarketMaker's manifest clobber, 2026-09-26).
4. Every runner refuses sealed dates by name. There is no flag to override it.
5. Missing settlements, holidays and short sessions are flagged in the ledger record, never imputed
   silently. The 40%-of-median record-count rule flags a short day.

## What this stage does not do

- It does not build a sleeve. It does not compute a return.
- It does not choose the HAR-RV lags or the EWMA half-life; those are stage 5's, fitted on fit only.

## Next action

Price one day of each series (a `definition` and a `statistics` request per product), confirm the
history depths in the table, and write the catalogue skeleton with those answers before pulling.
