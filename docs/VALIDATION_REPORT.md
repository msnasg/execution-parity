# Validation report

Engine version 0.6.1. All experiments ran on 2026-10-29 on Binance USD-M Futures.

This report covers **three separate experiments**. They answer different questions and must not be combined into a single score.

| | Experiment | Question | Compared against |
|---|---|---|---|
| **A** | Runtime integrity | Did the runtime integration enforce its rules inside a Freqtrade Dry-run? | Dry-run database and evidence records |
| **B** | Historical Replay vs Dry-run | How closely does an integrated replay of the same window match the Dry-run? | Freqtrade Dry-run records |
| **C** | Aggregate Trades vs Depth10 | On the same orders, which price source better reproduces Dry-run fill prices? | Freqtrade Dry-run fill prices |

**Scope of all three.** The test used a synthetic stress harness with random entries and no trading edge. It is not an investable strategy, and this report contains no financial results. Dry-run fills are simulated, so **no experiment involves a real exchange fill**. Definitions and denominators are in the [Methodology](METHODOLOGY.md).

## Experiment A: runtime integrity

An isolated Freqtrade 2026.9 Dry-run ran with the runtime integration on 20 symbols. These results come from the frozen audit snapshot taken 3.82 hours after launch.

| Check | Result |
|---|---|
| Audited entries | 662 |
| Trades completed at the audit cutoff | 655 (7 still open) |
| Entries with approved risk authorization | **662 / 662** |
| Requested leverage equals applied leverage | **662 / 662** |
| Entries reproduced from seed, pair and candle | 662 / 662 |
| Unauthorized entries | **0 observed** |
| Duplicate or orphaned orders | **0 observed** (662 attempts, 662 trades; 1,317 ledger orders, 1,317 database orders) |
| Maximum open positions (limit 10) | No observed violation: peak of 10 |
| Gross exposure (limit 2× equity) | No observed violation: peak of 1.9995× over 1,324 approvals |
| Rejections at the exposure limit | 28, all correctly over the limit |
| Stale-signal rejections | 18, all before any order |
| Slot release | Never before a confirmed close, at all 662 confirmations |
| Exit reason equals the scheduled exit | 655 / 655 |

**Connectivity-related delays.** 18 of the 655 exits (2.7 %) were delayed by more than 10 seconds, by 12 to 385 seconds. All 18 coincide with recorded Binance connectivity failures. Each delayed trade still exited once with the correct reason, and its slot was released only at the fill. Without those 18, the delay from the scheduled exit to the exit order had a median of 2.71 seconds and a maximum of 8.21 seconds.

The audit attributed no delay to the strategy or to the engine.

**Operational completion.** After the audit snapshot, the session continued and was shut down in a controlled way: new entries were disabled, open positions reached their normal exits, and the session ended with 1,210 closed trades, 2,420 closed orders and zero open positions. The results above come from the audit snapshot, not from this final total.

## Experiment B: Historical Replay vs Freqtrade Dry-run

One integrated replay of the audited window was compared with the Dry-run: same strategy, configuration, seed, symbols and limits. The replay used the Aggregate Trades fill model and 957,206 public aggregate trades. The window was fixed at 3.8 hours (15:57:57.83 to 19:47:21.42 UTC).

### Results

The first reconciliation found two causes of divergence: a decision-timing gap, and Freqtrade's automatic pair lock, which the replay did not model. The corrected run addresses those two causes and is the main result. Both are shown.

| Metric | Original | Corrected | Denominator (corrected) |
|---|---|---|---|
| Deterministic signal regeneration | 100.00 % | **100.00 %** | 899 / 899 |
| Entry recall | 95.47 % | **96.98 %** | 642 / 662 Dry-run entries |
| Entry precision | 87.41 % | **96.11 %** | 642 / 668 replay entries |
| Entry F1 | 91.26 % | **96.54 %** | Harmonic mean of the two above |
| Strict exit agreement | 66.93 % | **59.66 %** | 383 / 642 matched entries |
| Exit reason agreement | 100 % | **100 %** | 635 trades closed in both |
| Median entry decision offset (signed) | −10.68 s | **−0.56 s** | 642 matched entries |
| Median absolute entry fill-time difference | 9.19 s | **3.89 s** | 642 matched entries |
| Median absolute exit fill-time difference | 7.19 s | **7.11 s** | 635 closed in both |
| Median signed exit fill-time difference | −3.67 s | **+6.32 s** | 635 closed in both |
| Median absolute entry price difference | 2.24 bps | **1.12 bps** | 642 matched entries |
| Median absolute exit price difference | 1.83 bps | **1.49 bps** | 635 closed in both |
| Exact position-count agreement | 39.87 % | **51.53 %** | 918 samples, 15 seconds apart |
| Unexplained divergences | 1 | **0** | Under the report's attribution scheme |

![Original and corrected reconciliation compared for entry recall, precision, F1, position-count agreement, strict exit agreement and entry decision offset](../assets/charts/reconciliation-original-vs-corrected.svg)

**What improved.** Entry precision rose by 8.70 points, replay-only entries fell from 91 to 26, and the median entry decision offset fell from about 10.7 seconds to about half a second. Position-count agreement improved but remains low.

**What got worse.** Strict exit agreement fell from 66.93 % to 59.66 %. The original figure was partly two errors cancelling each other: replay entries came about 10.7 seconds early, which offset the replay's later fills. Removing the first error exposed the second. Exits are now typically later in the replay, with a median of +6.32 seconds.

![Median timing differences between replay and Dry-run for the original and corrected runs](../assets/charts/execution-timing-comparison.svg)

### Reading the headline metrics

* **Signal agreement (100 %)** is deterministic regeneration. The Dry-run's generated signals were reconstructed from the seed and candles and compared with the replay's. It is not proof that every generated signal was independently observed in the Dry-run logs: 674 of the 899 appear in Dry-run evidence, and all 674 are in the replay.
* **Strict exit agreement (59.66 %)** requires a matched entry to close in both systems before the cutoff, with the same exit reason and exit fills within 10 seconds. The denominator is all 642 matched entries, so the 7 positions still open at the cutoff count as not agreed. Over the 635 trades closed in both, the same test gives 383 / 635 = 60.31 %.
* **Exit reason agreement (100 %)** shows that both systems exited for the same reason. It says nothing about when.

### Connectivity

The healthy-connectivity subset uses the same definitions, and excludes 239 intents whose decision window or lifetime touches an audited connectivity episode or the five minutes after it.

| Metric | All trades | Healthy connectivity |
|---|---|---|
| Dry-run / replay / matched entries | 662 / 668 / 642 | 448 / 437 / 436 |
| Entry recall | 96.98 % | 97.32 % |
| Entry precision | 96.11 % | 99.77 % |
| Entry F1 | 96.54 % | **98.53 %** |
| Strict exit agreement | 59.66 % (383 / 642) | 62.39 % (272 / 436) |
| Median absolute entry / exit price difference | 1.12 / 1.49 bps | 1.13 / 1.29 bps |
| Median absolute entry / exit fill-time difference | 3.89 / 7.11 s | 3.87 / 6.93 s |

![Entry F1 and strict exit agreement for all trades and the healthy-connectivity subset, with sample sizes](../assets/charts/connectivity-subset-comparison.svg)

Connectivity accounts for most of the precision gap: 25 of the 26 replay-only entries are attributed to outages. It explains little of the exit gap, which is mainly a fill-model effect.

### Attribution of remaining differences

The corrected comparison has 300 divergent or unmatched intents: 254 matched but divergent, 20 Dry-run only and 26 replay only.

| Attributed cause | Count |
|---|---|
| Market fill model | 134 |
| Connectivity outage | 66 |
| Unknown or unresolved order handling | 63 |
| Residual decision timing | 32 |
| Risk or slot state | 5 |
| Unexplained | 0 |

"Unexplained: 0" means that every case fits one of the report's categories. It does not mean the differences are harmless, and categories such as "unknown or unresolved order handling" are not root-cause proof.

### Remaining material differences

* **Exit timing:** only 59.66 % strict agreement, with replay exits systematically later.
* **Prices:** on matched trades, replay prices average 1.04 bps (entries) and 1.42 bps (exits) better for the trader than the Dry-run. The Aggregate Trades model is optimistic relative to the Dry-run simulator.
* **Portfolio state:** position counts match exactly in 51.53 % of samples, or 56.71 % outside affected intervals, with a mean absolute difference of 0.79 positions.
* **Evidence limits:** the exact live loop schedule cannot be reconstructed. The replay ran 2,523 loops against 2,537 in the Dry-run.

**Replay integrity checks.** 0 causality violations, 0 timestamp corrections, 0 duplicate active exits, a maximum of 10 slots, and 0 data requests outside verified coverage.

## Experiment C: Aggregate Trades vs historical Depth10

> **This was a standalone diagnostic, not a Historical Replay run.** It applied Freqtrade's own order-book pricing routine to recorded 10-level books at the Dry-run's order instants. Book-based execution is not implemented in engine 0.6.1.
>
> **The 96.32 % figure is agreement with Freqtrade's Dry-run simulator.** It is not end-to-end trade reconciliation, and it is not real-fill accuracy.
>
> **Clock offset.** All figures use a 0.885-second correction between the Dry-run's clock and exchange time, measured at start-up. Drift over the session was **not** independently verified. Without the correction, over all determinate orders, Depth10 falls to 79.45 % (entries) and 80.90 % (exits) within 1 bps, and the proxy to 58.17 % and 60.78 %.

### Design

Both methods priced the **same Dry-run orders**, with the same timestamp, side and quantity. The existing Aggregate Trades proxy was used unchanged.

| | Count |
|---|---|
| Dry-run market orders in the audited snapshot | 1,317 |
| Eligible, on the 14 symbols with Depth10 coverage (14 of 20) | 938 (71.22 %) |
| Determinate fill from Depth10 | 938 |
| Determinate fill from the proxy | 924 (14 unresolved) |
| Determinate under both methods (the paired sample) | **924** (460 entries, 464 exits) |
| Orders lacking sufficient 10-level depth | **0 of 938** |

The 379 orders on the other 6 symbols have no Depth10 history and are excluded. These results do not extend to those symbols.

### Results

| Metric (924 paired orders) | AGG_TRADE_PROXY | Depth10 |
|---|---|---|
| Within 1 bps of the Dry-run fill | **69.70 %** (644) | **96.32 %** (890) |
| Mean absolute price error | **1.02 bps** | **0.07 bps** |
| p95 absolute error | 5.05 bps | 0.73 bps |
| Mean signed bias | −0.90 bps | +0.01 bps |

A negative bias means the model is better for the trader than the Dry-run fill. Depth10 was closer on **506** orders, the proxy on **32**, and **386** were ties.

![Paired-order diagnostic comparing AGG_TRADE_PROXY and Depth10 on within-1-bps agreement, mean absolute error and mean signed bias](../assets/charts/price-accuracy-paired-diagnostic.svg)

| Within k bps | Proxy, entries (n = 460) | Depth10, entries | Proxy, exits (n = 464) | Depth10, exits |
|---|---|---|---|---|
| 0.5 bps | 59.35 % | 94.35 % | 63.15 % | 94.83 % |
| 1 bps | 68.70 % | 96.52 % | 70.69 % | 96.12 % |
| 2 bps | 81.09 % | 100 % | 84.05 % | 99.35 % |
| 5 bps | 94.78 % | 100 % | 94.83 % | 100 % |

![Absolute price error at the median, p90, p95, p99 and maximum for entries and exits](../assets/charts/price-error-quantiles.svg)

Both medians are 0.00 bps, because each method reproduces the Dry-run price exactly on many orders: 482 for the proxy and 869 for Depth10. The difference lies in the tail.

**The improvement is systematic.** Depth10's mean absolute error is lower in every hour and in 27 of 28 symbol-and-role groups. A sign test gives p < 1e-50 for entries and for exits. Removing orders above either method's 99th percentile leaves mean absolute errors of 0.93 vs 0.05 bps (entries) and 0.84 vs 0.05 bps (exits).

**Where the proxy's bias sits.** It concentrates in AAVE, APT, BCH, LINK, SUI and UNI, at −0.75 to −2.20 bps by symbol and role, and it grows when an order walks deeper into the book.

**Depth10's residual.** 55 of the 924 orders have a non-zero error, 34 exceed 1 bps, and the maximum is 2.94 bps. This fits a different data source (a recorded 100 ms websocket stream rather than Freqtrade's REST snapshot) and an instant a fraction of a second apart.

### Ten-level sufficiency

For these orders, ten levels were enough. The largest order used 20.45 % of the visible depth on its side. 858 of 938 orders filled at the first level, and the deepest walk was 6 levels. Order sizes were about 81 to 753 USDT notional, so this does not extend to larger orders or thinner books.

### Unknown-outcome cases

The corrected reconciliation in Experiment B has 63 unknown or unresolved divergences, and 48 of those are on covered symbols.

* **All 48 covered intents are explainable by sufficient historical resting liquidity.** In all 49 orders involved, a fresh and causally valid book covered the quantity within 10 levels.
* The Dry-run filled all 49. The proxy either filled them only after re-evaluation (36) or could not determine a fill (13).
* The 15 cases on uncovered symbols remain unverified.

Public book snapshots show that liquidity was visible. They do not prove that a real order would have filled at that price.

### Data quality

| Check | Result |
|---|---|
| Historical book rows checked | 2,086,749 (about 2.09 million) |
| Sequence-continuity breaks | 0 |
| Duplicates, invalid payloads, crossed or empty books | 0 |
| Level-ordering violations, non-positive values | 0 |
| Levels per side | 10 on every row |
| Gaps over 1 second | 361, all confirmed quiet by sequence continuity; largest 3.93 s |
| Median age of the snapshot used | About 0.19 s |
| Causality checks | 0 snapshots after the order instant, 0 proxy fills before it |

Collector receive delays reached 6.79 seconds inside the window. Sequence continuity was preserved in all of them, and the rule used for this report counted a snapshot only once the collector had received it.

### What Experiment C supports

Historical order-book data reproduces Freqtrade Dry-run prices more closely than public trade prints on these orders. This supports studying an *optional* book-based mode. It does not show that such a mode would improve exit agreement in an integrated replay, because that has not been built or tested, and it says nothing about real fills.

## Summary of what the evidence does not show

* It does not show accuracy against real exchange fills.
* It does not show better accuracy than native Freqtrade backtesting, because no such benchmark was run.
* It does not show results beyond one 3.8-hour window, these symbols and this harness.
* Experiment C does not extend to the 6 uncovered symbols.
* It does not show any strategy's viability. No financial result is reported, and the harness is not an investable strategy.
