# Methodology

## Test harness

The experiments used a synthetic stress harness, not a trading strategy. Entries are random, generated from a fixed seed, so every signal can be regenerated exactly. Each trade has a tagged leverage of 1× to 5× and a scheduled hold of 1 to 3 minutes. Limits were 10 open positions and 2× gross exposure.

The harness exists to generate a high volume of orders. It has no trading edge, and no financial result is reported from it.

| Item | Value |
|---|---|
| Date | 2026-10-29 |
| Market | Binance USD-M Futures, 20 symbols, 1-minute candles |
| Dry-run | Freqtrade 2026.9, isolated instance, simulated orders only |
| Audit window | 15:57:57.83 to 19:47:21.42 UTC (3.8 hours) |
| Market data for replay | 957,206 public aggregate trades from the official Binance API, 15:50 to 19:55 UTC, with contiguous IDs and no duplicates |

The Dry-run database was captured as an immutable, hashed snapshot at the cutoff. All comparisons read that snapshot.

## Experiment A: runtime integrity

The audit is read-only. It joins the Dry-run trade database, the engine's evidence records and the Freqtrade log. A check passes only if every record has its counterpart. For example, each entry must trace to a trade, to an approved confirmation for the same attempt, pair and side, and to an earlier approved sizing decision.

"0 observed" means none were found in these records. It is a statement about this run.

## Experiment B: reconciliation

### Identity

Trades are matched by intent: `pair | side | signal candle | entry tag`. This key is unique in both systems. Numeric trade IDs are not used.

### Metrics

| Metric | Definition | Denominator |
|---|---|---|
| **Signal agreement** | Generated signals present in both, over the candle range. Dry-run signals are regenerated from the seed, not read from logs | All 899 generated signals |
| **Entry recall** | Matched entries / Dry-run entries | 662 |
| **Entry precision** | Matched entries / replay entries | 668 |
| **Entry F1** | Harmonic mean of recall and precision | None (derived) |
| **Strict exit agreement** | Matched entries closed in both systems before the cutoff, with the same exit reason and exit fills no more than 10 seconds apart | All 642 matched entries |
| **Exit reason agreement** | Same exit reason | 635 trades closed in both |
| **Entry decision offset** | Replay decision time minus Dry-run decision time | 642 matched entries |
| **Fill-time difference** | Replay fill time minus Dry-run fill time, as a signed or absolute median | 642 entries, 635 exits |
| **Price difference** | Replay price against Dry-run price, in bps. Positive means the replay is worse for the trader | 642 entries, 635 exits |
| **Position-count agreement** | Share of samples where both systems hold the same number of positions | 918 samples, every 15 seconds |

There is no combined reconciliation score. A single number would hide the difference between strong entry agreement and weak exit agreement.

### Why two reconciliations exist

The first reconciliation found the replay deciding entries a median of 10.68 seconds before the Dry-run. The corrected run changed two things in the replay setup. The engine itself did not change.

1. **Decision timing.** In the Dry-run, the first entry authorization of each minute came a median of 11.35 seconds after the candle closed. One fixed latency was derived from Dry-run evidence alone, without reference to trade outcomes, and the rule was stated before computing it.
2. **Automatic pair lock.** Freqtrade locks a pair until the next candle after an exit. The replay now models that rule, using Freqtrade's own timing behavior.

The same reconciliation code, data, identity and thresholds were used for both runs.

### Healthy-connectivity subset

An intent is excluded if its decision window, Dry-run lifetime or replay lifetime touches an audited connectivity episode or the five minutes after it. In the corrected run, 239 intents were excluded. The exclusion rule follows from the audit log and was not fitted to the results.

### Attribution

Each divergent or unmatched intent is assigned to one category, using recorded evidence: market fill model, connectivity outage, unknown or unresolved order handling, residual decision timing, or risk or slot state. "Unexplained" is reported separately. Attribution describes which recorded mechanism fits a case and is not a proof of root cause.

## Experiment C: paired price diagnostic

### Fairness

Both methods priced the same orders: same order, order timestamp, side and quantity. Orders that one method could not determine are excluded from the paired error figures and reported separately, so that both error distributions describe the same population.

### Sources compared

| Source | Method |
|---|---|
| **Reference** | The fill price Freqtrade recorded for the Dry-run order |
| **AGG_TRADE_PROXY** | The engine's existing Aggregate Trades fill model, with its logic and parameters unchanged |
| **Depth10** | Freqtrade's own Dry-run market pricing routine, applied to a recorded 10-level book |

Freqtrade's Dry-run reads a 20-level REST book when it places an order. The recorded data holds 10 levels. If 10 levels could not cover an order, the result would have been marked insufficient: levels 11 to 20 were never fabricated. No order needed this.

### Causal rules

Two rules select the book for an order at instant `t`:

* **Rule A, exchange as-of:** the latest snapshot emitted by the exchange at or before `t`.
* **Rule B, collector-known as-of:** the latest snapshot that had also reached the collector by `t`.

Rule B is the more conservative rule and is used for every published figure. The two rules gave a different price on 86 of 938 orders, and their results are close. No snapshot from after `t` was used.

### Clock correction

Dry-run order timestamps use the local clock. Freqtrade logged the local clock's offset from Binance as 877 ms and then 885 ms at start-up. The diagnostic subtracts 885 ms from every order timestamp.

The offset was not measured again during the session, so drift is **not verified**. The correction materially affects the results: without it, within-1-bps agreement falls for both methods, as shown in the [Validation report](VALIDATION_REPORT.md#experiment-c-aggregate-trades-vs-historical-depth10). Depth10 remains ahead of the proxy in that case.

### Metrics

| Metric | Definition |
|---|---|
| **Signed error** | Model price minus Dry-run price for a buy, and the reverse for a sell, in bps. Positive means the model is worse for the trader |
| **Absolute error** | The size of the signed error |
| **Within k bps** | Share of orders with absolute error of k bps or less |
| **Mean signed bias** | Mean of the signed errors. Negative means the model is optimistic relative to the Dry-run |

The pooled figures cover 924 orders. Separate entry and exit figures are in the [Validation report](VALIDATION_REPORT.md).

### Data validation

Every historical book row was parsed and checked for symbol, timestamps, update-ID continuity, duplicates, level ordering, positive values, and crossed or empty books. Gaps over 1 second were classified with sequence evidence: when each message's previous-update ID equals the prior message's update ID, no message was lost, so the gap is a quiet market. No data was interpolated or repaired.

## Evidence handling

Each experiment was fixed before analysis: one Dry-run snapshot, one data capture, one reconciliation script for both runs. Inputs and outputs are hashed internally. Because the raw evidence includes private records, it is not published here. The public figures are generated directly from the experiment metrics files.

Results are reported as measured, including results that got worse.
