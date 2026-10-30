# Execution Parity — Advanced Backtesting & Trade Execution Validation (Reconciliation) for Freqtrade

A specialized historical execution and reconciliation framework for the Freqtrade ecosystem, providing auditable order-lifecycle simulation, execution-aware risk modelling and quantitative comparison against Freqtrade Dry-run behavior.

**Engine version 0.6.1 · Proprietary software · Documentation-only repository**

Execution Parity is owned by Markets & Investment Technology. The implementation is private. This repository holds product documentation, validation results and figures. It contains no source code.

![Architecture overview: a strategy feeds a historical path and a Freqtrade Dry-run path, both apply shared execution rules, and their evidence meets in reconciliation](assets/diagrams/architecture-overview.svg)

## Why execution assumptions matter

A backtest result depends on more than a strategy's signals. A candle-based simulation has to assume when a decision was made, what happened inside a candle, whether capacity was free at that moment, and what price an order received.

Those assumptions decide which trades exist at all, not only the numbers attached to them. An entry can be placed into a position slot that had not yet been released. A fill can be assumed at a price that was already gone. An order can be treated as filled when the available evidence cannot show that it was. Such differences stay hidden until the strategy runs somewhere else.

Execution Parity makes these assumptions explicit, testable and measurable.

## What it does

| Component | Purpose |
|---|---|
| **Historical Replay** | Processes signals, orders, fills and exits in causal event order over historical market data, with explicit settlement and order states |
| **Runtime integration** | Applies the same intent and risk rules inside a Freqtrade Dry-run bot, through Freqtrade's supported callbacks |
| **Reconciliation** | Compares replay and Dry-run records for entries, exits, prices, timing and positions, and attributes each divergence |
| **Execution evidence** | Records authorizations, order states and fills so that results can be audited after the fact |

Details and evidence status for each capability are in [Technical capabilities](docs/TECHNICAL_CAPABILITIES.md).

## Validation scorecard

Three separate experiments ran on Binance USD-M Futures on 2026-10-29, using a synthetic stress-test harness with no trading edge. **No real order was placed, and none of these figures measure real exchange fills.**

| Experiment | Metric | Result | Sample |
|---|---|---|---|
| **A. Runtime integrity** | Entries with approved risk authorization | **662 / 662** | Audited Dry-run snapshot |
| | Requested leverage equals applied leverage | **662 / 662** | Same |
| | Unauthorized entries; duplicate or orphaned orders | **0; 0** | Same |
| **B. Historical Replay vs Dry-run** | Deterministic signal regeneration | **100.00 %** | 899 / 899 signals |
| | Entry F1 | **96.54 %** | Recall 642 / 662, precision 642 / 668 |
| | Entry F1, healthy connectivity only | **98.53 %** | Recall 436 / 448, precision 436 / 437 |
| | Strict exit agreement | **59.66 %** | 383 / 642 matched entries |
| | Exit reason agreement | **100 %** | 635 trades closed in both systems |
| **C. Price diagnostic** (standalone) | Within 1 bps of the Dry-run fill: Depth10 | **96.32 %** | 890 / 924 paired orders |
| | Within 1 bps of the Dry-run fill: AGG_TRADE_PROXY | **69.70 %** | 644 / 924 paired orders |
| | Mean absolute price error: Depth10 vs proxy | **0.07 vs 1.02 bps** | 924 paired orders |

![Reconciliation results: signal agreement 100 percent, entry recall 96.98, entry precision 96.11, entry F1 96.54, strict exit agreement 59.66](assets/charts/reconciliation-kpi-overview.svg)

### How to read these numbers

* **Experiments B and C are different things.** Experiment B is an integrated Historical Replay run using the Aggregate Trades fill model. Experiment C is a standalone diagnostic that priced Dry-run orders from recorded order books. It was not a replay run, and book-based execution is not part of engine 0.6.1.
* **The 96.32 % figure is agreement with Freqtrade's Dry-run simulator.** It is not end-to-end trade reconciliation and not real-fill accuracy.
* **Experiment C depends on a 0.885-second clock-offset correction.** The offset was measured at start-up, and drift over the session was not independently verified. Without it, Depth10 falls to about 79–81 % within 1 bps and the proxy to about 58–61 %.
* **100 % signal agreement is deterministic regeneration,** not proof that every signal was independently observed: 674 of the 899 appeared in Dry-run evidence, and all of those matched.
* **Strict exit agreement is the weakest result, and it is reported as measured.** Replay exits arrive later than Dry-run exits because the Aggregate Trades model waits for printed trades, while the Dry-run fills against its order book at once.
* **The test window is short:** 3.8 hours, 20 symbols, one harness.

Definitions and denominators are in the [Methodology](docs/METHODOLOGY.md). The full results are in the [Validation report](docs/VALIDATION_REPORT.md).

## What sets the approach apart

* **Orders are events, not candle prices.** A fill is applied at its own instant, using only information available from that instant on. Capacity is reserved from submission and released only at a confirmed outcome.
* **The same rules run in two places.** Replay and the Dry-run bot share the intent and risk logic, so a difference between them points at execution behavior rather than two rule sets.
* **Incomplete evidence stays incomplete.** If the evidence cannot determine a fill, the order is labelled as unresolved. The engine does not invent an intrabar path or assume a fill.
* **Every percentage has a denominator, and weak results are published.** Each divergence is assigned to a named cause. Strict exit agreement fell when the first reconciliation was corrected, and that is shown in the [Validation report](docs/VALIDATION_REPORT.md).

A direct benchmark between native Freqtrade backtesting and Execution Parity has **not** been performed, so this project makes no measured claim of superiority over it. The architectural differences are described in the [Product overview](docs/PRODUCT_OVERVIEW.md#how-it-relates-to-freqtrade).

## Limitations

* Historical execution modelling cannot prove real fills.
* Public Aggregate Trades are not an order book, and the proxy's fills arrive later than the Dry-run's.
* Historical Depth10 data covers 14 of the 20 test symbols, and integrated book-based replay is not implemented.
* Fixed timing models do not reproduce every live loop, and connectivity outages affect ordering and position occupancy.
* Evidence comes from one short window, one symbol universe and one synthetic harness.
* Limit Take-Profit management with DCA-aware replacement and conditional Market fallback is future research, not a current feature.

The full list is in [Limitations](docs/LIMITATIONS.md).

## Status and availability

The implementation is proprietary and is available only through an agreement with the owner. No open-source licence is granted by this repository, and no licence to use the software is implied. The engine has not been used for live trading in any validation described here.

Markets & Investment Technology is open to discussions with technical partners, quantitative teams and trading infrastructure developers about integration or licensing.

Execution Parity is an independent product. It is not affiliated with or endorsed by the Freqtrade project or Binance. See the [Notice](NOTICE.md).

## Documentation

| Document | Content |
|---|---|
| [Product overview](docs/PRODUCT_OVERVIEW.md) | The problem, intended users, and how the product relates to Freqtrade's own tools |
| [Technical capabilities](docs/TECHNICAL_CAPABILITIES.md) | What is implemented, with the evidence status of each capability |
| [Validation report](docs/VALIDATION_REPORT.md) | Results of the three experiments |
| [Methodology](docs/METHODOLOGY.md) | Test design, metric definitions and denominators |
| [Limitations](docs/LIMITATIONS.md) | Known limits and future work |
| [FAQ](docs/FAQ.md) | Common questions |
| [Changelog](CHANGELOG.md) | Documentation history |
