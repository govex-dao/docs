# Balance Wrappers: Scalable Type Erasure for Multi-Outcome Markets

*Part 2 of the [Advanced Sui Design Patterns](./README.md) series*

---

Futarchy requires $N$ outcomes. In Move, defining `Pool<T0, T1...TN>` leads to **Type Explosion**, making code unmanageable as $N$ grows.

## The Problem

Move generics are powerful but static. If you need a pool with 10 outcomes, you'd need:

```move
// This doesn't scale
public struct Pool<T0, T1, T2, T3, T4, T5, T6, T7, T8, T9> { ... }
```

Every function touching the pool needs 10 type parameters. Add outcome 11? Rewrite everything.

## The Solution: Type Re-hydration

Govex erases type information for storage and math, then "re-hydrates" it at transaction time.

### Storage (Type Erasure)

Assets are stored as a `vector<u64>`. This allows looping over hundreds of outcomes in a single gas-efficient call.

```move
// Internal: Loop-friendly u64 storage
public struct ConditionalMarketBalance<phantom Asset, phantom Stable> {
    balances: vector<u64>,  // [out0_asset, out0_stable, out1_asset, out1_stable, ...]
}
```

### Interface (Re-hydration)

When external DeFi (like a DEX) requires a typed `Coin<T>`, the protocol borrows a `TreasuryCap` from a dynamic field and mints the coin on-demand.

```move
// External: PTB provides the type to "re-hydrate" the asset
public fun unwrap_to_coin<..., ConditionalCoinType>(
    wrapper: &mut ConditionalMarketBalance,
    outcome_idx: u8,
    ctx: &mut TxContext,
): Coin<ConditionalCoinType>
```

### TreasuryCap Vector Simulation

The `TreasuryCap` for each outcome type is stored via dynamic fields with index-based keys:

```move
public struct AssetCapKey has copy, drop, store {
    outcome_index: u64,
}

// Store caps with vector-like indexing
dynamic_field::add(&mut escrow.id, AssetCapKey { outcome_index: 0 }, cap_0);
dynamic_field::add(&mut escrow.id, AssetCapKey { outcome_index: 1 }, cap_1);

// Retrieve by index
let cap: &mut TreasuryCap<T> = dynamic_field::borrow_mut(&mut escrow.id, AssetCapKey { outcome_index });
```

## Why This Matters

- **Scales to 200+ outcomes** without code changes
- **Gas efficient**: Loop over `u64` values, not objects
- **Composable**: External protocols get typed coins they expect
- **Zero type parameter explosion**: Core logic uses 2 phantom types max

## Source Code

- [conditional_balance.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_primitives/sources/conditional/conditional_balance.move)
- [coin_escrow.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_primitives/sources/conditional/coin_escrow.move)

---

*Next: [The Iterative Hot Potato: PTB Loop Unrolling](./03-iterative-hot-potato.md)*
