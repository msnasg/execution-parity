# Limitations

Execution Parity measures and models execution behavior. It has real limits, and the evidence behind it is narrow.

## Evidence

* **Historical execution cannot prove real fills.** No experiment placed a real order. Every fill in this project is either simulated by Freqtrade's Dry-run or modelled by this engine. Agreement between them is agreement between two simulations.
* **The test window is limited.** The evidence comes from one 3.8-hour window on one day, 20 Binance USD-M symbols, 1-minute candles and one synthetic harness. Other market conditions, symbols, order sizes and strategy shapes are untested.
* **Many capabilities have synthetic tests only.** Limit orders, position adjustment and several portfolio limits are covered by regression tests but were not exercised in the experiments. See [Technical capabilities](TECHNICAL_CAPABILITIES.md).
* **No benchmark against native backtesting exists.** No measured superiority over Freqtrade's backtester is claimed.

## Fill modelling

* **Public Aggregate Trades are not an order book.** They show what traded, not what was available to trade. The `AGG_TRADE_PROXY` model waits for printed trades to cover an order, so its fills arrive later than a Dry-run fill. In Experiment B the proxy's entry fills came a median of 2.6 seconds after submission, against about 0.26 seconds in the Dry-run.
* **The proxy is optimistic relative to the Dry-run.** Its mean signed bias was −0.90 bps per order in the paired diagnostic, and it grows for orders that walk deeper into the book.
* **Some outcomes stay unknown.** When public prints cannot determine a fill, the order is labelled unresolved. That reports the limit of the evidence and does not mean the order would have failed. Such handling can leave replay positions different from the Dry-run's.
* **Partial fills and queue position add uncertainty.** Limit-order queue position is not modelled. Limit-order outcomes from public trades carry labelled uncertainty, and IOC, FOK and post-only outcomes are undetermined without a book at submission. Market impact is not modelled.

## Historical Depth10

* **Coverage is limited.** Depth10 history exists for 14 of the 20 test symbols, and it cannot be recovered after the fact: no public history of it exists, and a missed collection interval is lost.
* **Full end-to-end historical-book replay is not implemented.** The 96.32 % result comes from a standalone diagnostic. An integrated mode might behave differently, and it has not been built or tested.
* **Depth10 is not Freqtrade's book.** Freqtrade reads a 20-level REST snapshot at order time. The recorded data is a 10-level, 100 ms websocket stream observed slightly earlier. Those REST snapshots were not stored, so exact reconstruction is impossible.
* **Ten levels may not suffice elsewhere.** They covered every order here, at about 81 to 753 USDT notional. Larger orders or thinner books may exceed them.
* **The general Historical Replay must remain usable without Depth10.** Most symbols and periods have no such data.

## Timing

* **Fixed timing models do not reproduce every live loop.** A live iteration has variable latency, with tails of tens of seconds. The replay uses one fixed value, so individual minutes differ. The exact live loop schedule cannot be reconstructed from the records.
* **A timing calibration is specific to its setup.** The value used in Experiment B reflects that machine, symbol count and strategy. It is not a universal default.
* **Clock synchronization matters.** The Depth10 result depends on a 0.885-second offset measured at start-up. Drift during the session is not verified. Without the correction, within-1-bps agreement falls by about 15 to 17 percentage points for Depth10 and about 10 for the proxy.

## Connectivity

* **Outages affect ordering and position occupancy.** During connectivity failures in the Dry-run, 18 exits were delayed by up to 385 seconds. A historical replay does not experience those outages, so the two systems hold different positions afterwards. Outages account for 25 of the 26 replay-only entries.

## Remaining differences

* **Exit timing:** strict exit agreement is 59.66 %, and replay exits are typically later than Dry-run exits.
* **Portfolio state:** exact position-count agreement is 51.53 %.
* **Attribution is not proof.** "0 unexplained divergences" means every case fits a defined category.

## Platform scope

* Freqtrade is required for the runtime integration. Validated versions are listed in [Technical capabilities](TECHNICAL_CAPABILITIES.md).
* The engine is not exchange-independent. Evidence comes from Binance USD-M Futures.
* Cross margin, hedge mode, inverse contracts and stop orders on the exchange are not supported, and are refused.
* The isolated liquidation calculation is approximate.

## Future research

These items are **not** part of engine 0.6.1:

* An optional integrated book-based Historical Replay mode, including market entries.
* A Limit Take-Profit lifecycle with DCA-aware replacement.
* Conditional Market fallback for that lifecycle.

They are research directions, and none has been implemented or validated.
