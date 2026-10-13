# mogul: Selling VIX Vol-of-Vol, Delta-Hedged, Through the Spikes

> [!NOTE]
> **This is the public showcase repository.** To request access to the private, full-source repository, please email [laurent.lanteigne@gmail.com](mailto:laurent.lanteigne@gmail.com).

*Mogul:* a bump on a ski run. The skier goes straight through the bumps and absorbs each one with the legs. This book
sells VIX option premium every trading day, straight through the volatility spikes, and absorbs each one with a daily
futures hedge.

It is a research project, not a traded book. Its second purpose was to build a research process strict enough to
reuse: every look at the data counted before it was read, every decision and prediction logged, and the out-of-sample
data sealed in code.

## Hypothesis

VIX calls are the cheapest convex hedge against an equity crash, so tail hedgers buy them all the time. The dealers
who sell them are short the volatility of volatility, and they charge for it. A seller who hedges the VIX-future
direction every day keeps that charge.

The premium is measurable before any trade P&L exists. Over 2015–2023, the model-free implied variance of the front
VIX options exceeded the variance the VIX future then realized in 83% of monthly cohorts, in every one of nine years.
It sits in the call wing: the put side mostly diversifies.

## Results (daily-marked, 100 $ of vega per daily entry, 1× costs)

| | Design window 2015–2023 | Holdout 2024-01 → 2026-09 (read once) | Backcast 2008-07 → 2013 (read once, second vendor) |
|---|---|---|---|
| Sharpe, carried book¹ | **1.58** | **1.74** | **0.39** |
| Sharpe, frozen original candidate | 0.92 | **1.48** (passed all four registered gates) | 0.37 |
| Max drawdown, carried book | 1.9 × annual vol | 1.1 × annual vol | 1.6 × annual vol |
| Luck check, block-bootstrap p-value | 0.0001 | 0.005 | 0.18 |

¹ The carried book is the call-heavy strangle with the level gate, at constant size; the vol-of-vol sizing adds about
0.17 on the design window and nothing on the holdout. It was chosen after the holdout look, so its holdout column is a
re-read, not a blind test. The
blind holdout results are the registered candidates: every one was positive, and the frozen candidate passed.

**What the three windows say together:**
- The premium is real, and it travelled to data nobody had read.
- It is regime-dependent. A run of back-to-back vol-of-vol shocks (2010–2011) earns almost nothing.
- A long-run Sharpe near 1 is the fair expectation, not the last decade's 1.6–1.7.

## Strategy design (what it does; parameters live in the private repository)

- **The structure:** a call-heavy short strangle on the front monthly VIX options, more short calls than puts. The
  call wing is where the premium is.
- **Entry:** a new position every trading day at a fixed time, at a fixed vega size, held to expiry. About twenty
  positions are live at once, so no single entry day matters.
- **Pricing:** every option is valued against the put-call parity forward of its own expiry. Neither the vendor's
  underlying nor the futures settle is used: both are struck at a different time.
- **Hedge:** once a day, in the VIX future of the same expiry.
- **A level gate:** no new entries when VIX is low. There the premium is too thin to pay for a low-volatility blow-up.
- **A stop** at a multiple of the credit, kept as a backstop.
- **Sizing:** smaller after a quarter of volatile VIX, the regime in which the premium stops paying.

## What did not work (kept on the record)

- **Filters chosen from regime cuts on the design window.** A volatility cap, a weekly tenor, VVIX and term-structure
  conditioning all lost their design-window advantage out of sample. Structure chosen on economics travelled;
  conditioning chosen from the data did not.
- **Long VIX optionality** (ratio spreads, long-ATM / short-wing shapes) bleeds.
- **Profit-taking and time exits** lose against holding to expiry.
- **Volatility targeting of the book** sizes up into spikes.
- **A better hedge is not always a better book.** The smile-adjusted minimum-variance delta hedges every option better,
  but removes a small implicit short-futures position that was earning the VIX futures roll.

## Research protocol

- **One engine,** identity-tested and checked against synthetic markets, used by every notebook.
- **An append-only event log** of 475 decisions, looks, predictions, quirks and reviews, rendered into ledgers.
- **Every look at the data counted** before it was read: about 500 on the design window, deflated Sharpe at the
  running count.
- **A holdout sealed in code.** Reading it raises an error until a ledger line records who opened it and why. It was
  opened once. A second, earlier window was sealed and opened the same way.
- **Predictions registered before every run, then scored:** 65 hit, 19 partial, 36 miss.
- **Data defects disclosed with their effect:**
  - stale settlement prints in the exchange's own history
  - a vendor greek bias
  - a snapshot-clock dependence of daily stops on spike days
- **Since the holdout was opened, every design choice is made on the design window only** and re-read afterwards.

## Private repository structure

```
mogul_trade_private/
├── README.md                # card, what was learned, step table, honesty ledger
├── PROTOCOL.md              # the research protocol (reusable)
├── RESEARCH_PLAN.md         # steps and amendments
├── HYPOTHESIS.md            # the frozen claim, candidates, metric, power, gates
├── FINDINGS.md              # results per step, newest first, errata visible
├── DECISIONS.md LOOKS.md PREDICTIONS.md QUIRKS.md STATUS.md  # ledgers rendered from the log
├── HOLDOUT_LEDGER.md        # the seal, the openings, disclosed peeks
├── log/events.jsonl         # the append-only event log
├── engine.py                # data, chain, Black-76, trades, books, statistics, hedges
├── scripts/                 # data exports, step runners, notebook builders
├── notebooks/ (00–13)       # data QC · carry · engine · anatomy · decomposition ·
│                            #   structure · candidates · exits · regimes · kill battery ·
│                            #   comparators · holdout · backcast · sizing/stop · hedge
└── tests/                   # holdout guard, engine identities, data QC, freeze pin
```

*Nothing here is investment advice. `strategy_results.ipynb` reproduces every figure from the small derived series in
`data/`.*
