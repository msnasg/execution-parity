# FAQ

## Product

### What is Execution Parity?

A historical execution and reconciliation framework for Freqtrade. It replays a strategy's orders chronologically, applies the same intent and risk rules inside a Freqtrade bot, and compares the two.

### Is the source code available?

No. The implementation is proprietary and available only through an agreement with the owner, Markets & Investment Technology. This repository contains documentation only.

### Is it affiliated with Freqtrade?

No. It is an independent product, and neither Freqtrade nor Binance endorses it.

### Is it in production use?

No production deployment or customer is reported here. The published evidence comes from controlled simulation experiments.

### Why are there no performance figures?

Execution Parity is not a strategy. The experiments used a random stress harness with no trading edge, so any such figure would be meaningless and potentially misleading. Reporting covers execution behavior only.

## Accuracy

### Does it predict real exchange fills?

No, and no accuracy against real fills is claimed. All comparisons are against Freqtrade's Dry-run simulator.

### Is it more accurate than Freqtrade backtesting?

That has not been measured. No direct benchmark was run. The [Product overview](PRODUCT_OVERVIEW.md#how-it-relates-to-freqtrade) describes the architectural differences only.

### Does it replace Freqtrade's backtester or forward testing?

No. It is a separate engine, and forward testing remains necessary.

### What does 100 % signal agreement mean?

The replay and the Dry-run's reconstructed signals are identical over the window, across 899 signals. This is deterministic regeneration from a fixed seed, so it is expected and is a consistency check. Of those signals, 674 appeared directly in Dry-run evidence, and all of them matched.

### What does Entry F1 of 96.54 % mean?

Of 662 Dry-run entries, 642 were matched by the replay (recall 96.98 %). Of 668 replay entries, 642 matched the Dry-run (precision 96.11 %). F1 is the harmonic mean of the two. It covers which entries occurred, not their prices or timing.

### Why is strict exit agreement only 59.66 %?

Because it is strict: same exit reason and exit fills within 10 seconds, over all 642 matched entries. Exit reasons agreed on all 635 trades closed in both systems. The gap is timing. Replay fills depend on public trades printing after the order, while the Dry-run fills immediately against an order book.

### Why did exit agreement fall in the corrected run?

The original run's early entries partly cancelled the replay's late fills, so its 66.93 % was partly two errors offsetting. Correcting the entry timing exposed the fill latency. The lower number describes the model more accurately.

### Why report results for a healthy-connectivity subset?

Connectivity failures in the Dry-run delay exits and change which positions are open, and a historical replay does not experience them. The subset shows how much of the gap comes from outages. It raises Entry F1 from 96.54 % to 98.53 %, and strict exit agreement from 59.66 % to 62.39 %.

## Depth10

### Is Depth10 execution part of the product?

No. Engine 0.6.1 does not include book-based market entry execution. The Depth10 results come from a standalone diagnostic.

### Then what does 96.32 % mean?

Across 924 paired Dry-run market orders on 14 symbols, a recorded 10-level book priced with Freqtrade's own routine came within 1 bps of the Dry-run fill on 96.32 % of orders. The Aggregate Trades proxy reached 69.70 % on the same orders. This is agreement with a simulator's price, not trade reconciliation and not real-fill accuracy.

### Does that result depend on anything fragile?

Yes. It depends on a 0.885-second clock-offset correction measured at start-up, and drift over the session was not independently verified. Without the correction, Depth10 falls to about 79–81 % and the proxy to about 58–61 %.

### Would using Depth10 in replay fix exit agreement?

Unknown. It might reduce fill latency, but an integrated mode has not been built or tested.

### Can Historical Replay run without Depth10?

Yes. It runs on public Aggregate Trades. That was the configuration used in Experiment B.

## Features

### Does it manage Limit Take-Profit orders with DCA-aware replacement or Market fallback?

No. That lifecycle is future research and is not implemented.

### Which exchanges are supported?

The evidence comes from Binance USD-M Futures. Binance Spot is supported by the engine but was not part of the experiments. No other exchange has been validated.

### What happens when evidence is incomplete?

The order is labelled unresolved, or the action is refused. The engine does not guess a fill or invent an intrabar price path.

## Reading the results

### Where are the definitions and denominators?

In the [Methodology](METHODOLOGY.md).

### Can the results be combined into one score?

No. The three experiments answer different questions, and no combined score is defined.
