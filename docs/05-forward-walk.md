# Stage 5 — The forward walk

**Deliverable:** `decide()` running every session against the IBKR account through the API, with
guards, reconciliation and a daily report. It runs first on paper (the rehearsal), then live at
reduced size.

## The instrument table

Every number is read from a named source on a named date and re-read before live orders. Fees are the
broker's all-in per side.

| exposure | instrument | notional per contract | initial margin | fee per side | source |
|---|---|---|---|---|---|
| index delta | MES | ~$30k | broker page, the day | $0.62 (MarketMaker record, 2026-09-27) | IBKR schedule |
| duration | micro 10Y yield (or ZN on tracking failure) | ~$10 per bp (ZN ~$110k) | broker page | to read | IBKR schedule |
| vega | MES options: the straddle, or straddle plus wings | ~$30k | broker page (SPAN) | to read | IBKR schedule |

SPX options (~$600k a contract), SPY on Reg T leverage, and front-month VIX futures are ruled out for the
reasons in `STATUS.md`.

## The daily cycle (all times CT)

1. **15:05:** pull today's settlements (GLBX) and the account's positions and NAV (IBKR).
2. **Reconcile.** Broker positions must equal the journal's holdings. On a mismatch, place no orders
   and send an alert.
3. **`decide()`** with the current artifact.
4. **Guards** (below). Any guard that fires blocks the orders it names and is journalled.
5. **Execute.** Post limit orders at the touch for futures. Option legs go as a combo at mid, stepped
   toward the far side on a fixed schedule. Anything unfilled by 15:50 is cancelled, and the journal
   records the shortfall. The next day's `decide()` starts from actual holdings.
6. **Journal and report.** The same journal format as the backtest, plus live fills, slippage against
   settlement, and the guard log.

Treasury futures settle at 14:00 CT and equity at 15:00 CT, but orders go in after 15:00. The
difference from settlement is a cost, measured every day and compared to the cost model's allowance.

## Guards (carried over from MarketMaker's stage 6)

- **Stale data:** today's settlement is missing or older than today → no orders.
- **Loss limit:** if the session's P&L reaches −`D` (10% of NAV), flatten futures and close option legs
  at the next opportunity, then halt until a person clears it.
- **Margin:** the post-trade margin estimate is above `M` → scale the orders down until it is not.
- **Kill file:** its presence cancels open orders and places nothing.
- **Reconciliation:** above; a mismatch halts.
- **Expiry and roll:** an option leg inside 2 DTE that the calendar did not roll → alert and roll.

Each guard is driven in the rehearsal by replaying a historical day that should trip it.

## The sequence

1. **Rehearsal (paper).** Run on the IBKR paper account from the day it is ready. Before that, the
   rehearsal period (2026-07-01 onward) is replayed through the live code path with a simulated broker.
   Parity check: on every replayed day, the live path's intended orders equal the backtest engine's.
2. **Quarter size (live).** `decide()`'s targets are computed on a quarter of NAV. At $50k this may round
   a sleeve to zero; where it does, that sleeve starts at one contract, and the size actually run is
   stated.
3. **Half, then full**, each step after one month in which the daily report is inside the bands below.

## Daily report and the go/no-go

Reported per sleeve: realised vs predicted cost, realised vs target exposure, per-sleeve return against
the backtest engine's replay of the same day, margin used, which constraint bound, and any guard that
fired.

**Go to full size** requires two months where realised cost per sleeve is inside the backtest's
predicted band (the backtest's cost ± 2 SE of its daily cost) and realised exposures track the targets.
The differences and their SEs are logged. Two months of absolute return is not a criterion.

## Next action

Copy MarketMaker's TWS connection and order code into `research/src/wo/live/`. Run it on the paper
account with a no-op `decide()` for one session, and confirm that positions, NAV and fills round-trip
into the journal.
