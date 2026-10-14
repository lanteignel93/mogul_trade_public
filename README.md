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

## Results (daily-marked on every trading day, 100 $ of vega per daily entry, 1× costs)

| | Design window 2015–2023 | Holdout 2024-01 → 2026-09 (read once) | Backcast 2008-07 → 2013 (read once, second vendor) |
|---|---|---|---|
| Sharpe, carried book¹ | **1.45** | **1.65** | **0.39** |
| Sharpe, frozen original candidate | 0.92 | **1.48** (passed all four registered gates) | 0.37 |
| Max drawdown, carried book | 2.0 × annual vol | 1.2 × annual vol | 1.6 × annual vol |
| Luck check, block-bootstrap p-value | 0.0002 | 0.008 | 0.19 |

¹ The carried book is the call-heavy strangle with the level gate, at constant size; the vol-of-vol sizing adds about
0.15 on the design window and nothing on the holdout. It was chosen after the holdout look, so its holdout column is a
re-read, not a blind test. The blind holdout results are the registered candidates: every one was positive, and the
frozen candidate passed.

*Correction, 2026-10-14: an earlier version of this page scored the gated book on its active days only (1.58 on the
design window, 1.74 on the holdout). A book that is flat on some days must count those days as zero; every number here
now does.*

**What the three windows say together:**
- The premium is real, and it travelled to data nobody had read.
- It is regime-dependent. A run of back-to-back vol-of-vol shocks (2010–2011) earns almost nothing.
- A long-run Sharpe near 1 is the fair expectation, not the last decade's 1.5–1.7.

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

## Augmenting the strategy with chairlift

[chairlift](https://github.com/lanteignel93/chairlift_public) is the machine-learning research pipeline behind these
studies: a cached graph from data to walk-forward folds to fitted models to books, with leakage checks inside each
fold, a sealed holdout, and every experiment counted in a trial ledger. Once mogul had a baseline that travelled,
chairlift was pointed at it with one question: **using only what is known at entry, can a model size the daily entries
better than constant size?**

**The setup.**
- **Target:** each daily entry's final P&L, delta-hedged, at 100 $ of vega. The full structure, and each leg alone.
- **Inputs:** 29 values known at entry: the VIX complex, the realized vol of VIX, the futures slope, the legs' implied
  vols and the smile, the credit, SPX returns and drawdown, scheduled macro releases inside the hold, the weekday.
- **Fit:** walk-forward on the design window only, refitted each year, with an embargo as long as an entry can live.
  The holdout stayed sealed.
- **Use:** a model may only redistribute size. The mean size stays at 1, so it can win only by timing.

**What it found** (test years 2017–2023, so the constant book differs from the 2015–2023 card above):

| | constant size | ridge-sized | gain [99% CI] | test years better |
|---|---|---|---|---|
| carried book (with the gate) | 1.57 | 2.04 | +0.46 [−0.24, +1.16] | 4 of 7 |
| the call wing alone, with the gate | 1.72 | 2.36 | +0.64 [+0.15, +1.18] | 4 of 7 |
| without the gate | 1.26 | 0.97 | −0.28 [−0.68, +0.49] | 3 of 7 |

- **Without the gate, the models learn the VIX level** and size up into the entries that get stopped. That adds risk,
  not Sharpe.
- **Inside the gate, a ridge with stable signs sorts out the entries that get run over.** The stop rate falls from
  24% in the lowest fifth of predictions to 2% in the highest. It sizes up after an SPX drawdown and when the call wing
  is rich. It sizes down after a quarter of volatile VIX, after a VIX rise, and when a CPI release falls inside the
  hold.
- **Learning the position directly** (a mean-variance policy) was no better. Sizing each leg by its own model gained,
  and all of the gain came through the call wing.
- **Searching the trade's own settings** found the family (the gate, a call-heavy structure, a stop) but not a single
  best setting: the best cell in hindsight does not carry forward (probability of backtest overfitting 0.65).

**The verdict: no ML sizing is adopted.** The gain has the right sign and an economic story, but it holds in only 4 of
7 test years, the in-sample fit far exceeds the out-of-sample fit, and about 225 books were evaluated to find it. What
chairlift added is mostly confirmation of what mogul found by hand: the gate, the call-heavy structure, the stop, and
smaller size in persistent vol-of-vol. If one ML candidate is carried forward, it is the gated call-wing ridge, judged
once on data it has not seen.

The ML layer itself (the chairlift client, the review of the model and of the pipeline that produced it, the figures)
is in [mogul_ml_trade_public](https://github.com/lanteignel93/mogul_ml_trade_public).

## Research protocol

- **One engine,** identity-tested and checked against synthetic markets, used by every notebook.
- **An append-only event log** of 513 decisions, looks, predictions, quirks and reviews, rendered into ledgers.
- **Every look at the data counted** before it was read: about 500 on the design window, deflated Sharpe at the
  running count.
- **A holdout sealed in code.** Reading it raises an error until a ledger line records who opened it and why. It was
  opened once. A second, earlier window was sealed and opened the same way.
- **Predictions registered before every run, then scored:** 70 hit, 20 partial, 43 miss.
- **Data defects disclosed with their effect:**
  - stale settlement prints in the exchange's own history
  - a vendor greek bias
  - a snapshot-clock dependence of daily stops on spike days
- **Since the holdout was opened, every design choice is made on the design window only** and re-read afterwards.
- **Errors in our own reporting are logged like data defects,** with the corrected numbers next to the old ones (the
  active-day correction above is one).

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
├── chairlift_client/        # the ML layer: study, experiments, review of the model and the pipeline
└── tests/                   # holdout guard, engine identities, data QC, freeze pin
```

*Nothing here is investment advice. `strategy_results.ipynb` reproduces every figure from the small derived series in
`data/`.*
