# Product overview

## What Execution Parity is

Execution Parity is a historical execution and reconciliation framework for Freqtrade strategies. It answers a narrow question: given a strategy's signals, what would the order lifecycle have looked like, and how closely does that match what a Freqtrade Dry-run bot actually recorded?

It has three parts:

* **Historical Replay,** a separate engine that processes a strategy's orders chronologically over historical market data.
* **Runtime integration,** which applies the same intent and risk rules inside a Freqtrade bot through its supported callbacks.
* **Reconciliation,** which compares the two sets of records and attributes the differences.

It does not generate signals, and it adds no trading logic to a strategy.

## Who it is for

* Quantitative teams that need to know how far a historical result depends on execution assumptions.
* Trading infrastructure developers building on Freqtrade.
* Technical reviewers who need auditable evidence of what a simulation did and why.

## The problem it addresses

| Failure mode | Why it matters |
|---|---|
| **Decision timing** | A simulation can act on a candle earlier than a running bot would. In this project's first reconciliation, the replay decided entries a median of 10.68 seconds before the Dry-run did |
| **Capacity ordering** | A position slot freed inside a candle can be reused in an order that a running bot could not have followed. The trade list changes, not only the prices |
| **Fill assumptions** | A filled order needs evidence of a price and of available quantity. Public trade data alone often cannot show both |
| **Unknown outcomes** | Some orders cannot be determined from the available evidence. Treating them as filled or as failed both create a false record |
| **Connectivity** | A running bot loses connections and delays exits, while a historical run does not. The two then hold different positions |

## How a trade moves through the system

![Execution lifecycle: signal, risk authorization, order, settlement, position, exit, accounting, with explicit non-fill outcomes and an evidence record](../assets/diagrams/execution-lifecycle.svg)

The diagram is conceptual. Each step has explicit states, and outcomes that are not fills are recorded as such.

## How it relates to Freqtrade

Execution Parity works alongside Freqtrade and replaces none of its tools. The descriptions of Freqtrade below are based on its stable backtesting documentation and on the installed 2026.9 source, reviewed for this release.

| Tool | Owner | How it works | Relevant limit |
|---|---|---|---|
| **Native backtesting** | Freqtrade | Simulates on candles. Entries happen at the candle open unless custom price logic is set, and orders fill at the requested price if it lies within the candle's range. Exits free their slot for a new trade on a different pair | The documentation calls the inability to know how prices moved inside a candle a major limitation, and states that backtesting will never replace Dry-run |
| **`--timeframe-detail`** | Freqtrade | Evaluates candles with activity on a lower timeframe, which simulates intra-candle movement more closely and can free slots earlier | The documentation notes that entries generally still happen at the main candle's open |
| **Dry-run** | Freqtrade | Runs the live loop on live market data with simulated orders. A market order is priced by walking a 20-level order book, with a 5 % slippage cap, and closes immediately | Fills are simulated. They are not exchange fills |
| **Historical Replay** | Execution Parity | A separate engine, not a Freqtrade mode. It replays decisions on a modelled loop schedule and settles each order at its own instant, using public trade data | A model of fills, with the limits in [Limitations](LIMITATIONS.md) |
| **Reconciliation** | Execution Parity | Compares replay and Dry-run records for the same window | Measures agreement between two simulations |

### What has not been measured

A direct benchmark of native Freqtrade backtesting against Execution Parity has **not** been completed. The table describes architectural differences only. Nothing here shows that Historical Replay is more accurate than native backtesting, with or without `--timeframe-detail`.

The measured comparison in this project is against **Freqtrade Dry-run**, and is described in the [Validation report](VALIDATION_REPORT.md).

## What it is not

* It is not a trading strategy, a signal source or a performance tool.
* It is not a promise that a historical result will match live trading.
* It is not a substitute for forward testing.
* It is not exchange-independent. All evidence so far comes from Binance USD-M Futures.
* It is not a general-purpose execution platform. It is built around Freqtrade.

## Commercial status

Execution Parity is proprietary software owned by Markets & Investment Technology and available only through an agreement with the owner. It has no customers, certifications or production deployments to report. The evidence published here comes from controlled simulation experiments.
