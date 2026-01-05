|   SIP-Number | |
|         ---: | :--- |
|        Title | On-Chain Liveness Oracle |
|  Description | Extends sui::clock to act as a trustless liveness oracle, enabling on-chain circuit breakers. |
|       Author | Greshamscode, @92GC |
|       Editor | <Leave this blank; it will be assigned by a SIP Editor> |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-06-25 |
| Comments-URI | https://github.com/sui-foundation/sips/pull/59 |
|       Status | Draft |
|     Requires | |

## Abstract

This proposal extends the existing `sui::clock` module to function as a native liveness oracle. It exposes two new functions: one to query the largest time gap between consecutive commits within a specified time range, and one to get the median commit gap in the history window. This enables smart contracts to detect network stalls relative to typical network behavior and act as trustless circuit breakers, removing the need for off-chain social consensus on network health.

## Motivation

Currently, there is no way for a smart contract to programmatically determine if the Sui network has recently experienced significant downtime or a stall. This forces critical decisions about network health to rely on off-chain social consensus. This proposal introduces an on-chain, trustless liveness oracle.

TWAP (Time-Weighted Average Price) functions should not be trusted if the chain has experienced significant downtime. [Govex.ai](https://www.govex.ai/), a futarchy-based DAO management platform on Sui, needs to run TWAPs over short-duration prediction markets. We cannot risk a chain stall occurring during a market, nor do we want to trust an admin with the power to invalidate markets.

By comparing the maximum gap during a specific time period against the median (typical) gap, contracts can detect anomalies in a way that automatically adjusts to protocol upgrades and changing network conditions.

## Specification

This SIP extends the existing `sui::clock` module with the following public functions:

```move
module sui::clock {
    // ... existing functions ...

    /// Returns the largest time gap in milliseconds between any two consecutive
    /// commits where both commits fall within the specified time range.
    ///
    /// - Returns `Some(max_gap)` if the range is valid and within accessible history
    /// - Returns `None` if the range is invalid or extends beyond accessible history
    public native fun largest_commit_gap_between(
        start_timestamp_ms: u64,
        end_timestamp_ms: u64
    ): Option<u64>;

    /// Returns the median time gap in milliseconds between consecutive commits
    /// within the protocol-defined history window.
    ///
    /// This represents the "typical" commit gap and can be used as a baseline
    /// to detect anomalies. The median is resistant to outliers - a single
    /// large stall will not skew this value.
    ///
    /// Note: This value may be cached and updated at checkpoint boundaries
    /// rather than computed live.
    public native fun median_commit_gap_in_window(): u64;
}
```

### Example Usage

```move
/// Circuit breaker check for a prediction market
public fun is_market_valid(market_start: u64, market_end: u64): bool {
    let max_gap = clock::largest_commit_gap_between(market_start, market_end);
    let typical_gap = clock::median_commit_gap_in_window();

    match (max_gap) {
        Some(gap) => gap <= typical_gap * 100, // 100x typical = stall
        None => false, // Invalid range
    }
}
```

## Rationale

### Why Commit Timestamps (not Checkpoint Timestamps)

Per feedback from Mysten Labs: checkpoint construction and execution are asynchronous, so commit timestamps provide more accurate timing information.

### Why Extend Clock (not a New Module)

Adding these functions to the existing `sui::clock` module is cleaner than introducing a new `sui::checkpoint` module. The Clock object already represents time-related functionality.

### Why Median (not EMA)

We considered exponential moving averages (EMAs) as suggested by reviewers. However:

1. **Intuitive**: "Typical gap" is easier to reason about than abstract α values (0.1, 0.01, 0.001)
2. **Outlier resistant**: Median is not skewed by a single large stall, whereas EMAs are affected by spikes
3. **Self-adjusting**: Like EMAs, median automatically adjusts to protocol upgrades that change commit intervals
4. **Simpler crash recovery**: Median can be recomputed from the history window on startup; doesn't require maintaining complex state across crashes

### Why a Time-Range Query

A global "max gap in history" doesn't help our use case. We need to check: "Was there a stall during *this specific market's duration*?" Hence `largest_commit_gap_between(start, end)`.

### Implementation Notes

The `median_commit_gap_in_window()` function does not need to be computed live. Precomputing and caching at checkpoint boundaries is sufficient - "median as of recent checkpoint" works for detecting anomalies. This simplifies crash recovery: recompute once on startup from the bounded history window.

## Backwards Compatibility

This proposal is purely additive. It extends an existing module with new functions and does not alter any existing functions or data structures. No existing smart contracts will be affected.

## Test Cases

To be developed following initial review of the proposal.

## Reference Implementation

A reference implementation will be developed by core engineers if the SIP is approved.

## Security Considerations

1. **Denial-of-Service (DoS) via Resource Exhaustion:** A malicious actor could call `largest_commit_gap_between` with a very large time range.
   * **Mitigation:** Gas cost should scale with the size of the time range queried. The protocol should enforce a maximum queryable range.

2. **Validator State Bloat:** Requiring validators to store infinite history is unsustainable.
   * **Mitigation:** The protocol defines a fixed-size history window (e.g., last 2-4 million commits). Data older than this window is pruned and only accessible via off-chain archival solutions.

3. **Median Computation Cost:** Computing median over millions of data points could be expensive.
   * **Mitigation:** Cache the median value and update it at checkpoint boundaries rather than computing on every call. On crash recovery, recompute once from the bounded history window.

## Copyright

Greshamscode 2025
