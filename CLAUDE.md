# Wealth Optimiser — session entry point

The unified wealth problem: maximise the long-run growth rate of a $50k account, net of the cost of
trading, over three return streams ("sleeves"): equity (MES), duration (micro yield futures or ZN), and
short variance (delta-hedged MES straddles). V1 delivers a forward-walkable daily function, `decide()`,
and a five-year backtest of that same function.
Opened 2026-09-29 after the market-making line (`vivere7108-lab/MarketMaker`) closed. That project's
microstructure record is this project's cost model, and its validation discipline is this project's
discipline. Its assessment (`MarketMaker/docs/11-assessment.md`) is the reason this repo exists.

## Read before working

1. `PLAN.md` — the problem, the sleeves, the stage list, and where each stage stops.
2. `STATUS.md` — what is done, what is in flight, what is next. **Update it at the end of every
   session.** A number without its mean ± SE and n does not go in it.
3. `docs/00-problem.md` — the objective, the notation, and the ledger contract every sleeve honours.
   Do not change a contract without updating this file and every consumer.
4. The stage doc for the stage being built (`docs/0N-*.md`).

## Workflow: orchestrator + one agent per stage

Carried over from MarketMaker (decided there 2026-09-26). The main session orchestrates; **one
subagent implements each stage**. This file is the standing authorisation to use the Agent tool for
that purpose.

- A stage agent writes code and tests under `research/`, leaves `make py-test` and `make lint`
  green, and reports its numbers. It commits nothing and edits no doc.
- The orchestrator owns `PLAN.md`, `STATUS.md`, `docs/**`, every commit, and every number that
  reaches the record — **re-run, never relayed**. A subagent's "tests pass" or "Sharpe 0.9" is a
  claim until the orchestrator has reproduced it.
- The protocol is `MarketMaker/docs/delegation.md`. Copy it to `docs/delegation.md` when the first
  agent is spawned, and adapt the command names.

## Commands

**Nothing builds yet.** Stage 0 creates these, and this list is updated when they exist:

```bash
make py-test    # cd research && uv run pytest
make lint       # ruff over research/
make data       # price, then pull, every catalogued series (prices first; hashes every file)
make ledger     # rebuild every sleeve's return stream from raw prices; hashes recorded
wo check <sleeve|moments|policy> PATH    # validate an artifact before it is used
wo decide --date D                       # one day's decision, printed (what the live loop runs)
wo backtest --config CFG --out DIR       # decide() daily over the backtest window, journal out
wo live --paper|--live                   # the forward walk against IBKR
```

## Non-negotiable rules

Each comes from a measured failure in the predecessor projects. Breaking them is how three weeks
went into a signal that made money on synthetic tape and lost it on real tape, and how a queue
assumption moved results more than any signal did.

1. **Real data only.** No synthetic price path enters any performance number. Bootstrap resamples
   of real returns are allowed for *sizing* and *tail* estimation, never for a performance claim.
2. **Net growth rate is the metric.** Sharpe, and a sleeve's marginal Sharpe against the rest of
   the portfolio, are the diagnostics. Reject by default any change that raises gross return and
   lowers net growth, or raises turnover and lowers net growth.
3. **Paired by year, with SEs.** Report mean ± SE and t across non-overlapping years (or blocks),
   the win rate, and how many arms × metrics were tested. A pooled total alone is not a result.
4. **The identifiability rule.** Condition an input on the data only at the horizon where the data
   identifies it: variances daily, means of broad asset classes by structural prior over decades,
   premia quoted as prices (the variance premium, carry) at the horizon they are quoted. **Every
   conditional mean ships beside its constant-mean control**, and replaces it only by a
   pre-registered held-out gate.
5. **The window is pre-registered** in `STATUS.md` before any return is computed: fit 2010-06 to
   2021-06, backtest 2021-07 to 2026-06. Fit-phase runners refuse backtest-period returns by name.
   Every backtest run is numbered and logged; the report states how many looks there were.
6. **One construction per sleeve.** Each return stream is built once, by one code path, from raw
   prices, with its own costs, and cross-checked against a published index where one overlaps. No
   sleeve enters the ledger from a vendor's index alone: the stream we trade is the stream we fit.
7. **Instruments are chosen by the cost model, not by preference.** An instrument enters with its
   exposure map, its all-in cost per side, its margin, and its worst-day loss, each with a named
   source in the instrument table (`docs/05-forward-walk.md`). Fees are read from the
   schedule, never assumed.

## Layout (intended; stage 0 creates it)

```
research/    Python 3.13 via uv
  src/wo/{data,sleeves,moments,optimiser,contracts,decide,backtest}.py
  src/wo/live/   the IBKR adapter, guards, reconciliation (from MarketMaker's TWS code)
  tools/     one runner per stage, each taking --phase and enforcing the date guard
  tests/
artifacts/   fitted parameters and policies, versioned and immutable, each with a manifest
data/        catalogue only (manifests with sha256, counts, cost); files live under /home/bread/wo-data
runs/        journals, reports, plots — gitignored, reproducible from data + artifacts
docs/        the problem statement and one doc per stage
```

## Environment facts

- `DATABENTO_API_KEY` is on the VPS at `/etc/harvester.env`, not in this shell. The GLBX.MDP3 plan is
  flat-rate: every CME, CBOT, NYMEX and COMEX future and option on a future prices at $0.00. Price
  every request before pulling anyway; the catalogue records what each one cost.
- Python 3.13 via uv. The system Python 3.14 is not used.
- The account is $50k at IBKR. Futures margin is whatever the broker's page says on the day; portfolio
  margin is unavailable below its minimum, so ETF leverage is Reg T's 2× and the delta lives in futures.
- Leverage is bounded by the per-session loss cap `D` = 10% of NAV; margin is only a feasibility cap,
  `M` = 50% of NAV. At $50k one contract is about one sleeve's risk budget, so holdings are integers
  chosen by the objective, and every result is reported at both fractional and integer sizing
  (`docs/03-optimiser.md`, "Whole contracts").
- MarketMaker's measured execution costs, reused here as the cost model's ES row: all-in fee $2.24 a side
  on ES (0.179 ticks) and $0.62 on MES (0.496 ticks); a 1-lot posted at the ES touch nets −0.26 ticks at
  5 s, crossing nets −0.68 (`MarketMaker/STATUS.md`, Result log 2026-09-27).
