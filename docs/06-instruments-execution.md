# Stage 6 — Instruments, costs, replay; and the forward walk (stage 8)

**Deliverable:** the exposure map from sleeves to contracts at $50k; the instrument table with a source
for every number; the cost model; `wo replay` on the ledger with every cost and roll applied and a
journal; then, in stage 8, the walk and its daily report.

## Instrument selection is the cost model's job

An instrument is a vector of exposures per unit, a cost per side, a margin footprint and a worst-day
loss. The optimiser hands over target exposures; this stage delivers them at the lowest cost under the
same loss cap:

```
minimise over holdings h:   trading cost(h) + carry and roll(h) + margin cost(h)
subject to:                 E(s) h = w*(s),   worst one-interval loss(h) <= D
```

`E` maps holdings to exposures. For a future it is a constant; for an option it is the Greeks and it
moves. Three rules follow, and they decide the table below:

1. **One instrument per exposure, linear wherever a linear one exists.** Delta in futures, duration in
   futures, vega in the option structure stage 3 chose, carry in the micro futures.
2. **Bundling is a hidden constraint.** A short put is long delta, short vega and short gamma in a ratio
   the strike chose, and its delta grows as the market falls, against the vol-scaling rule. The delta is
   therefore hedged in futures and the option carries vega only.
3. **Theta speed is turnover.** A shorter expiry collects the same premium per unit of risk faster and
   pays a wider spread relative to a smaller premium more often. Expiry is chosen by the ledger's
   net numbers, not by theta.

## The instrument table (sources named; numbers at an index level near 6,000)

Every number here is read from a named source on a named date and re-read before the walk. Fees are
the broker's all-in per side, including exchange and regulatory fees.

| exposure | instrument | notional per contract | initial margin | fee per side | source of fee and margin |
|---|---|---|---|---|---|
| index delta | MES (ES above ~$150k NAV) | ~$30k (ES ~$300k) | broker page, the day | MES $0.62, ES $2.24 (MarketMaker record, 2026-09-27) | IBKR schedule |
| duration | ZF or ZN | $100k face | broker page | to read | IBKR schedule |
| vega | MES options (the stage-3 structure) | ~$30k | broker page (SPAN) | to read | IBKR schedule |
| FX carry | M6E, M6B, MJY, M6A; CAD and CHF micros where listed | $7k–$15k | broker page | to read | IBKR schedule; `definition` file for listings |
| commodity carry | MGC, SIL, MHG, MCL; full size where no micro | $7k–$35k | broker page | to read | as above |

Why not the alternatives, for the record: SPX options are ~$600k a contract, twelve times the account;
SPY leverage needs portfolio margin, which has a six-figure minimum; front-month VIX futures have doubled
in a day with margin near half the notional; single-name options are a different sleeve, not a way to
express a basket.

## The cost model

Per instrument, per side: half the quoted spread at the time the trade would be made + the all-in fee
+ the execution-model term. The execution-model term comes from the MarketMaker record and is the reason
that project's investment pays into this one:

| ES, per contract, net of fee, at 5 s | ticks |
|---|---|
| post at the touch and accept the adverse fill | −0.26 |
| cross the spread | −0.68 |

A daily hedge or rebalance is timing-indifferent, so the replay charges the posting cost and the walk
posts; the microprice sign says when timing is not indifferent and the order crosses. Rolls are charged
at the calendar spread's quoted width. Option legs are charged at half the settlement spread where the
data carries it and at a stated tick otherwise. **The cost share of gross return is reported for every
sleeve and every arm**; a result whose cost share moves it across zero is reported as such.

## Margin model and the loss cap

Initial margin per contract from the broker's page, summed over holdings, must sit at or below 50% of
NAV (the leverage cap `L` of `docs/05-optimiser.md`). The loss cap `D` is enforced on the *instrument*
holdings, not the sleeve weights: the option structure's worst day is computed from its Greeks at the
close and from the bootstrap's worst index move, and the defined-risk arm's is its contract maximum.
Whichever is binding is logged daily.

## Replay

`wo replay --config CFG --out DIR` runs the full stack on the ledger from the fit split's first day:
moments → policy → exposures → contracts → costs → journal. The journal (`runs/replay/<ver>/journal.jsonl`)
has one line per day: state, targets, holdings before and after, trades, costs, margin, worst-day
estimate, and which constraint bound. The walk writes the identical journal, so a replay of the walk's
own days is the parity check: intended trades must match the live journal's.

The guards MarketMaker's stage 6 specified are carried over and driven from the ledger's own calendar in
replay, not only by unit tests: stale-data guard, margin guard, daily loss limit in dollars, kill file,
reconciliation of broker positions against the journal on every session.

## Forward walk (stage 8)

- Size: the optimiser's weights at one quarter of NAV for the first month, then half, then full, each
  step gated on the daily report matching the replay.
- Daily report: realised vs predicted cost per sleeve; realised vs target exposure; per-sleeve return
  against the ledger's own record of the same day (the ledger keeps building from live settlements);
  margin used; which constraint bound; any guard that fired.
- Go/no-go for full size: two months where the realised cost per sleeve sits inside the replay's
  predicted band and the realised exposures track the targets, with the differences and their SEs
  logged. Absolute return over two months is not a criterion; it cannot be.

## Next action

Fill the instrument table from the broker's pages and the `definition` file on one stated date, with the
listings of every micro confirmed, before any exposure map is coded.
