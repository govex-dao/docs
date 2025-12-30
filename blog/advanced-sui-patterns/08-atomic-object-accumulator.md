# The Atomic Object Accumulator: Scalable Heterogeneous Construction

*Part 8 of the [Advanced Sui Design Patterns](./README.md) series*

---

While Pattern #3 (Iterative Hot Potato) handles logic loops (doing X operation N times), this pattern handles **construction scalability**: building an object that requires more resources than a single function call can accept.

## The Problem: Constructor Argument Limits & Heterogeneous Types

In Move, transaction functions have practical limits on argument count. Furthermore, `vector` collections must be homogeneous—all elements must share the same type.

Consider a Proposal that needs 10 different conditional coin types:

```move
// This is IMPOSSIBLE in Move:
// - 40+ arguments exceed practical limits
// - vector<TreasuryCap<T>> can't hold different T types
public fun create_proposal(
    cap0: TreasuryCap<Cond0Asset>, meta0: CoinMetadata<Cond0Asset>,
    cap1: TreasuryCap<Cond0Stable>, meta1: CoinMetadata<Cond0Stable>,
    cap2: TreasuryCap<Cond1Asset>, meta2: CoinMetadata<Cond1Asset>,
    // ... 16 more parameters for remaining outcomes
    // ... plus all the proposal config parameters
) { }
```

Two hard walls:

1. **Argument Limit**: Cannot pass 40+ objects into a single function
2. **Type Heterogeneity**: Cannot use `vector<TreasuryCap<T>>` when each `T` is different

## The Solution: The Staged Builder

Split creation into three atomic phases within a single Programmable Transaction Block (PTB):

```
┌─────────────────────────────────────────────────────────────────┐
│                         Single PTB                               │
├─────────────────────────────────────────────────────────────────┤
│  1. BEGIN         → Create unshared object with empty Bags      │
│  2. ACCUMULATE    → Call add_outcome N times (heterogeneous)    │
│  3. FINALIZE      → Validate completeness, create AMMs, share   │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 1: The Empty Bucket

```move
/// Returns Proposal by value—NOT shared
/// It is "local" to the transaction block until finalized
public fun begin_proposal<AssetType, StableType>(
    dao_account: &Account,
    title: String,
    outcome_messages: vector<String>,
    // ... other config
    ctx: &mut TxContext,
): (Proposal<AssetType, StableType>, TokenEscrow<AssetType, StableType>) {
    let proposal = Proposal {
        id: object::new(ctx),
        conditional_treasury_caps: bag::new(ctx), // Empty Bag
        conditional_metadata: bag::new(ctx),      // Empty Bag
        // ...
    };
    let escrow = TokenEscrow { /* ... */ };

    (proposal, escrow)  // Return by value, unshared
}
```

Key insight: The proposal exists only within the PTB—it cannot be accessed by anyone else until shared.

### Phase 2: Heterogeneous Accumulation

```move
/// Add ONE outcome's conditional coins (4 type parameters per outcome)
public fun add_outcome_coins<
    AssetType, StableType,  // Base types (2)
    AC, SC,                  // Conditional asset/stable types (2)
>(
    proposal: &mut Proposal<AssetType, StableType>,
    escrow: &mut TokenEscrow<AssetType, StableType>,
    outcome_index: u64,
    asset_cap: TreasuryCap<AC>,
    asset_meta: CoinMetadata<AC>,
    stable_cap: TreasuryCap<SC>,
    stable_meta: CoinMetadata<SC>,
    // ...
) {
    // Bag allows mixing CoinMetadata<Cond0Asset> with CoinMetadata<Cond1Asset>
    let asset_key = ConditionalCoinKey { outcome_index, is_asset: true };
    let stable_key = ConditionalCoinKey { outcome_index, is_asset: false };

    bag::add(&mut proposal.conditional_metadata, asset_key, asset_meta);
    bag::add(&mut proposal.conditional_metadata, stable_key, stable_meta);

    // Register caps with escrow
    coin_escrow::register_outcome_caps(escrow, outcome_index, asset_cap, stable_cap);
}
```

For bulk operations, provide batched variants:

```move
/// Batch version: 10 outcomes = 22 type parameters
public fun add_outcome_coins_10<
    AssetType, StableType,  // 2 base types
    AC0, SC0,  // Outcome 0
    AC1, SC1,  // Outcome 1
    AC2, SC2,  // Outcome 2
    // ... up to
    AC9, SC9,  // Outcome 9
>(
    proposal: &mut Proposal<AssetType, StableType>,
    escrow: &mut TokenEscrow<AssetType, StableType>,
    start_outcome_index: u64,
    // 40 objects: 10 * (TreasuryCap + Metadata) * 2 coin types
    ac0: TreasuryCap<AC0>, am0: CoinMetadata<AC0>,
    sc0: TreasuryCap<SC0>, sm0: CoinMetadata<SC0>,
    // ... 9 more sets
) {
    let outcome_count = proposal.outcome_data.outcome_count;

    // Add each outcome that exists
    if (start_outcome_index + 0 < outcome_count) {
        add_outcome_coins(proposal, escrow, start_outcome_index + 0,
                          ac0, am0, sc0, sm0, ...);
    } else {
        destroy_unused_cap_and_metadata(ac0, am0);
        destroy_unused_cap_and_metadata(sc0, sm0);
    };
    // ... repeat for outcomes 1-9
}
```

### Phase 3: The Seal

```move
#[allow(lint(share_owned))]
public fun finalize_proposal<AssetType, StableType, LPType>(
    mut proposal: Proposal<AssetType, StableType>,
    mut escrow: TokenEscrow<AssetType, StableType>,
    spot_pool: &UnifiedSpotPool<AssetType, StableType, LPType>,
    // ...
) {
    // 1. INTEGRITY CHECK: Do we have all the coins we expect?
    let outcome_count = proposal.outcome_data.outcome_count;
    assert!(
        bag::length(&proposal.conditional_metadata) == outcome_count * 2,
        EMissingConditionalCoins
    );
    assert!(
        coin_escrow::caps_registered_count(&escrow) == outcome_count,
        EMissingConditionalCoins
    );

    // 2. LOGIC EXECUTION: Create conditional AMM pools
    let amm_pools = liquidity_initialize::create_outcome_markets(
        &mut escrow,
        outcome_count,
        // ...
    );

    // 3. POINT OF NO RETURN: Share objects
    transfer::public_share_object(proposal);
    transfer::public_share_object(escrow);
}
```

## The PTB in Practice

```typescript
const ptb = new TransactionBlock();

// Phase 1: Create unshared bucket
const [proposal, escrow] = ptb.moveCall({
    target: `${pkg}::proposal::begin_proposal`,
    typeArguments: [assetType, stableType],
    arguments: [daoAccount, title, outcomeMessages, ...],
});

// Phase 2: Accumulate (batch of 10)
ptb.moveCall({
    target: `${pkg}::proposal::add_outcome_coins_10`,
    typeArguments: [
        assetType, stableType,
        cond0Asset, cond0Stable,
        cond1Asset, cond1Stable,
        // ... 8 more pairs
    ],
    arguments: [
        proposal, escrow,
        0, // start_outcome_index
        cap0Asset, meta0Asset, cap0Stable, meta0Stable,
        // ... 9 more sets
    ],
});

// For 15 outcomes: add 5 more individually
for (let i = 10; i < 15; i++) {
    ptb.moveCall({
        target: `${pkg}::proposal::add_outcome_coins`,
        typeArguments: [assetType, stableType, condAsset[i], condStable[i]],
        arguments: [proposal, escrow, i, caps[i], metas[i], ...],
    });
}

// Phase 3: Validate and share
ptb.moveCall({
    target: `${pkg}::proposal::finalize_proposal`,
    typeArguments: [assetType, stableType, lpType],
    arguments: [proposal, escrow, spotPool, ...],
});
```

## Why This Matters

| Approach | Complexity Limit | Types |
|----------|------------------|-------|
| Single Constructor | O(1) - max ~20 args | Homogeneous |
| Atomic Accumulator | O(N) - gas limit only | Heterogeneous |

This pattern:

- **Bypasses argument limits**: Chain as many `add_outcome` calls as gas allows
- **Enables heterogeneous collections**: Bags store any type with phantom keys
- **Guarantees atomicity**: Half-initialized proposals never appear on-chain
- **Supports variable scale**: Same code handles 2, 10, or 50 outcomes

## Security Properties

1. **No incomplete objects**: Finalize validates completeness before sharing
2. **No unauthorized access**: Unshared objects exist only within the PTB
3. **Type safety**: Phantom type keys ensure correct coin pairing
4. **Idempotent validation**: Each outcome index can only be registered once

## Source Code

- [proposal.move - begin_proposal](https://github.com/govex-dao/govex/blob/main/packages/futarchy_markets_core/sources/proposal.move#L1220)
- [proposal.move - add_outcome_coins_10](https://github.com/govex-dao/govex/blob/main/packages/futarchy_markets_core/sources/proposal.move#L1542)
- [proposal.move - finalize_proposal](https://github.com/govex-dao/govex/blob/main/packages/futarchy_markets_core/sources/proposal.move#L1688)

---

*End of series. [Back to README](./README.md)*
