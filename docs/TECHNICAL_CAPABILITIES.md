# Technical capabilities

This is an architectural specification of engine version 0.6.1. It describes what the engine does, not how it is implemented.

![Capabilities and evidence status for Historical Replay, runtime integration, reconciliation and data-quality evidence, with future work shown separately](../assets/diagrams/capability-evidence.svg)

## Status labels

| Label | Meaning |
|---|---|
| **Implemented and tested** | In the engine and covered by regression tests |
| **Implemented with limitations** | In the engine, with a material restriction, or covered by synthetic tests only |
| **Research diagnostic only** | Demonstrated in an experiment using separate research tooling; not an engine feature |
| **Proposed** | Not implemented |

## Validated environment

| Item | Scope |
|---|---|
| Freqtrade | 2026.9 in the audited Dry-run; 2025.12 and 2026.9 in regression tests |
| Python | CPython 3.13 |
| Market | Binance USD-M linear perpetuals, isolated margin, one-way mode, 1-minute strategy timeframe |
| Engine tests | 188 regression tests pass on both Freqtrade versions |

Binance Spot is supported by the engine but was not part of the empirical experiments. No other exchange or trading platform has been validated.

## Historical execution

| Capability | Description | Status |
|---|---|---|
| Historical replay | Runs a strategy's decisions through a chronological execution model over historical data | Implemented and tested |
| Causal event processing | Events are ordered by their timestamps with deterministic tie-breaking. A fill can use only market events at or after its order, and a position slot is available only after its release is known | Implemented and tested |
| Decision timing and candle visibility | Models the Freqtrade loop schedule. A decision sees only candles closed at the start of its loop | Implemented with limitations: a fixed model, see [Limitations](LIMITATIONS.md) |
| Deferred settlement | A market entry is submitted at the decision instant, and its fill is applied at its own later instant | Implemented and tested; used in the experiments |
| Immediate settlement | The earlier model, kept as the default for existing integrations | Implemented and tested |

## Orders

| Capability | Description | Status |
|---|---|---|
| Order lifecycle | Orders move through explicit states, with idempotent events and no state reversal after a terminal state | Implemented and tested |
| Terminal states | `FILLED`, `CANCELLED`, `REJECTED`, `EXPIRED` and `UNRESOLVED`. `UNRESOLVED` means the evidence could not determine a fill. It is not an exchange state | Implemented and tested |
| Exit order continuity | One active exit order per position. An exit that cannot yet be determined stays the same order and is not resubmitted | Implemented and tested |
| Pending-order reservation | A position slot and the required capital are held from submission, so risk checks see pending exposure | Implemented and tested |
| Reservation settlement | At the fill, the reservation converts to a position. If the outcome is unresolved, it is released | Implemented and tested |
| Market orders | Filled from public Aggregate Trades as `AGG_TRADE_PROXY`, a labelled model | Implemented with limitations |
| Limit orders | Good-till-cancel limit orders, with fill certainty labelled from public trades | Implemented with limitations: synthetic tests only, no queue-position model |
| IOC, FOK and post-only | Reported as undetermined unless a book snapshot is supplied at submission | Implemented with limitations: synthetic tests only |
| Freqtrade Dry-run venue rule | Reproduces Freqtrade's Dry-run order rules on a supplied book, tested against the installed Freqtrade code on recorded books | Implemented with limitations: applies to exits and limit orders, not to market entries under deferred settlement |
| Stop orders on the exchange | Not supported; refused | Not supported |

## Risk and portfolio

| Capability | Description | Status |
|---|---|---|
| Risk authorization | One pre-trade authorization function used by both replay and the runtime integration | Implemented and tested; exercised in the Dry-run |
| Position and gross-exposure limits | Maximum open positions and maximum gross exposure relative to equity | Implemented and tested; exercised in the Dry-run |
| Other portfolio constraints | Symbol and cluster exposure, positions per cluster, net directional limit, re-entry cooldown | Implemented with limitations: synthetic tests only |
| Position and slot accounting | A slot is held until a confirmed full close | Implemented and tested |
| Position adjustment | Adding to a position (DCA) and partial exits | Implemented with limitations: synthetic tests only |

## Futures accounting

| Capability | Description | Status |
|---|---|---|
| Leverage | Isolated-margin leverage, with a clamp to the configured maximum | Implemented and tested |
| Fees | Maker and taker fees, with account invariants checked throughout | Implemented and tested |
| Funding | Funding is charged at settlement instants, in event order | Implemented with limitations: small amounts in the experiments, not independently verified |
| Liquidation | An approximate isolated-margin calculation | Implemented with limitations: excludes the exchange's tiered maintenance schedule unless supplied |
| Cross margin, hedge mode, inverse contracts | Not supported; refused rather than approximated | Not supported |

## Runtime integration

| Capability | Description | Status |
|---|---|---|
| Freqtrade integration | Added to a strategy as a layer. It uses Freqtrade's supported callbacks and does not patch Freqtrade | Implemented and tested; exercised in the Dry-run |
| Entry authorization | Sizing and final confirmation of each entry pass through the shared rules | Implemented and tested |
| Fail-closed entry hooks | An error inside an entry hook rejects the entry, rather than falling back to Freqtrade's permissive default | Implemented and tested |
| Entry-attempt tracking | Each attempt is resolved from Freqtrade's trade database, so an authorization is never treated as proof of submission | Implemented and tested |
| Exit independence | The integration does not block or delay exits | Implemented and tested |

Freqtrade keeps ownership of its loop, slot counting, order types, partial fills, timeouts, stops and protections at runtime.

## Evidence, restart and reconciliation

| Capability | Description | Status |
|---|---|---|
| Execution evidence | Append-only records of authorizations, order states, fills and engine decisions | Implemented and tested |
| Checkpoint and restart | A replay can be checkpointed and restored mid-run, and reproduces the uninterrupted run | Implemented and tested |
| Runtime restart | After a restart, Freqtrade's trade database is treated as authoritative | Implemented and tested |
| Ledger reconciliation | Compares normalized process-loop records with recorded ledgers | Implemented and tested |
| Observed-fill comparison | Compares recorded fills with post-order public trades | Implemented with limitations: not part of the published experiments |
| Same-window Replay vs Dry-run reconciliation | The multi-metric comparison in Experiment B | Research diagnostic only: experiment tooling, not a packaged workflow |
| Divergence attribution | Assigns each divergence in Experiment B to a named cause | Research diagnostic only |

## Incomplete evidence

The default uncertainty mode fails closed. Where evidence is insufficient, the engine labels or refuses the outcome rather than guessing:

* Missing or insufficient trade data yields a labelled unresolved result, not a fill.
* A candle-only run never invents a high and low order inside a candle.
* A delayed entry with no signal re-check available is refused.
* Unsupported callbacks and order types are refused instead of approximated.

Status: implemented and tested.

## Data

| Capability | Description | Status |
|---|---|---|
| Public archive acquisition | Downloads official Binance public archives, verifies checksums and records provenance | Implemented and tested |
| Coverage records | Checks ordering, duplicates and gaps, and distinguishes unavailable history from a missing local file | Implemented and tested |
| Point-in-time universe | Ranks symbols using only data available before each selection date | Implemented and tested |
| Coverage assumptions | A run needs complete data for its window. A data gap yields a labelled outcome, not a fill. In Experiment B, the experiment tooling also refused any request outside the verified capture | Implemented and tested |
| Depth10 row-level validation | Sequence continuity, ordering and book validity checks on recorded depth data | Research diagnostic only |
| Depth10 fill pricing | Pricing an order by walking a recorded 10-level book | Research diagnostic only |

## Proposed, not implemented

| Capability | Status |
|---|---|
| Integrated book-based Historical Replay, including market entries, as an optional mode | Proposed |
| Limit Take-Profit lifecycle with DCA-aware replacement | Proposed |
| Conditional Market fallback for that lifecycle | Proposed |

Historical Replay must stay usable without Depth10 data, because such data does not exist for most symbols and periods.
