# Dynamic Coin Acquisition: The Blank Coins Pattern

*Part 4 of the [Advanced Sui Design Patterns](./README.md) series*

---

Sui coin types must be defined at compile time. You cannot create `Coin<MyNewToken>` dynamically at runtime. This is a fundamental limitation for protocols that need N conditional token types per proposal.

## The Two Hard Constraints

### Constraint 1: No Runtime Type Creation

Move's type system is entirely static. Types must be known at compile time:

```move
// This is impossible - types can't be created at runtime
public fun create_market(outcome_count: u64) {
    for i in 0..outcome_count {
        let coin_type = create_new_coin_type();  // Can't do this
    }
}
```

### Constraint 2: Can't Use Types in the Same PTB They're Published

Even if you publish a coin module in a PTB, you cannot use that type as a type argument in the same transaction:

```typescript
const ptb = new Transaction();

// Step 1: Publish coin module - returns packageId as runtime Result
const [upgradeCap] = ptb.publish({ modules: [...], dependencies: [...] });

// Step 2: Try to use it - FAILS
ptb.moveCall({
  target: `${pkg}::escrow::register`,
  typeArguments: [`${???}::my_coin::MY_COIN`], // Can't interpolate Result!
});
```

Type arguments must be **string literals known at PTB build time**. The publish result is a runtime value—you cannot splice it into a type string.

This means "publish and use in one click" is impossible without this pattern.

Every conditional market needs unique coin types for each outcome. Deploying new coin modules per-proposal doesn't scale.

## The Solution: Pre-Created Blank Coins

Govex uses a **permissionless registry** of pre-created "blank" coin types that can be acquired on-demand, organized by decimal precision.

### The Registry Structure

```move
/// Composite key for storing coin sets: (decimals, cap_id)
/// Enables efficient routing by decimals while maintaining unique cap_id lookup
public struct CoinSetKey has copy, drop, store {
    decimals: u8,
    cap_id: ID,
}

/// A single coin set ready for use as conditional tokens
public struct CoinSet<phantom T> has store {
    treasury_cap: TreasuryCap<T>,
    metadata_cap: MetadataCap<T>,  // Cap to modify Currency<T> metadata
    owner: address,    // Who deposited (gets paid on acquisition)
    fee: u64,          // Fee in SUI to acquire
    decimals: u8,      // Decimals of the coin (validated at deposit)
}

/// Global registry storing available blank coin sets
/// BUCKETED BY DECIMALS for efficient matching
public struct BlankCoinsRegistry has key {
    id: UID,
    total_sets: u64,
    sets_by_decimals: vector<u64>,  // Index = decimals (0-18), value = count
}
```

### Blank Coin Validation

Coins must be truly "blank" and follow naming conventions:

```move
fun validate_coin_set_for_registry<T>(
    sui_registry: &SuiCoinRegistry,
    treasury_cap: &TreasuryCap<T>,
    metadata_cap: &MetadataCap<T>,
) {
    // Supply must be zero
    assert!(treasury_cap.total_supply() == 0, ESupplyNotZero);

    // Must be registered in Sui's CoinRegistry
    assert!(coin_registry::exists<T>(sui_registry), ECoinNotRegistered);

    // Module name must be "conditional_N" (content moderation)
    let type_info = type_name::get<T>();
    let module_name = type_name::get_module(&type_info);
    // Validate: starts with "conditional_", rest are digits only
    // e.g., conditional_0, conditional_42, conditional_999
}
```

### Permissionless Deposit with Decimal Bucketing

Anyone can contribute coin sets, organized by their decimal precision:

```move
public fun deposit_coin_set<T>(
    registry: &mut BlankCoinsRegistry,
    sui_registry: &SuiCoinRegistry,
    currency: &coin_registry::Currency<T>,
    treasury_cap: TreasuryCap<T>,
    metadata_cap: MetadataCap<T>,
    expected_decimals: u8,  // Depositor declares, VALIDATED against Currency<T>
    fee: u64,
    clock: &Clock,
    ctx: &TxContext,
) {
    assert!(registry.total_sets < MAX_COIN_SETS, ERegistryFull);
    assert!(fee <= MAX_FEE, EFeeExceedsMaximum);  // DoS protection

    validate_coin_set_for_registry(sui_registry, &treasury_cap, &metadata_cap);

    // CRITICAL: Validate declared decimals match actual coin decimals
    let actual_decimals = coin_registry::decimals(currency);
    assert!(expected_decimals == actual_decimals, EDecimalsMismatch);

    let coin_set = CoinSet {
        treasury_cap, metadata_cap,
        owner: ctx.sender(),
        fee,
        decimals: actual_decimals,
    };

    // Store with composite key (decimals, cap_id) for bucketed lookup
    let key = CoinSetKey { decimals: actual_decimals, cap_id: object::id(&treasury_cap) };
    dynamic_field::add(&mut registry.id, key, coin_set);
}
```

### PTB-Friendly Acquisition

Coin sets can be acquired by decimal precision and used in the same transaction:

```move
public fun take_coin_set_for_ptb<T>(
    registry: &mut BlankCoinsRegistry,
    desired_decimals: u8,  // Routes to correct bucket (e.g., 9 for assets, 6 for stables)
    cap_id: ID,
    mut fee_payment: Coin<SUI>,
    clock: &Clock,
    ctx: &mut TxContext,
): (TreasuryCap<T>, MetadataCap<T>, Coin<SUI>) {
    let key = CoinSetKey { decimals: desired_decimals, cap_id };
    let coin_set: CoinSet<T> = dynamic_field::remove(&mut registry.id, key);

    // Pay the depositor
    let payment = fee_payment.split(coin_set.fee, ctx);
    transfer::public_transfer(payment, coin_set.owner);

    let CoinSet { treasury_cap, metadata_cap, .. } = coin_set;
    (treasury_cap, metadata_cap, fee_payment)  // Return for use in same PTB
}
```

## Decimal Bucketing

The registry organizes coins by their decimal precision, enabling automatic matching:

```move
/// Get number of coin sets available for a specific decimal value
public fun sets_available_for_decimals(registry: &BlankCoinsRegistry, decimals: u8): u64 {
    *registry.sets_by_decimals.borrow(decimals as u64)
}
```

This allows:
- **Asset-conditional coins**: Request coins with 9 decimals (matching SUI/common tokens)
- **Stable-conditional coins**: Request coins with 6 decimals (matching USDC/stablecoins)

## Economic Design

### DoS Protection

Without fee limits, attackers could fill the registry with unusable coin sets:

```move
const MAX_FEE: u64 = 10_000_000_000;  // 10 SUI max

assert!(fee <= MAX_FEE, EFeeExceedsMaximum);
```

### Content Moderation

Module names are restricted to prevent offensive content in type strings:

```move
const ALLOWED_MODULE_PREFIX: vector<u8> = b"conditional_";

// Only allows: conditional_0, conditional_1, ..., conditional_999, etc.
// Rejects: conditional_offensive_name, random_module, etc.
```

### Incentive Alignment

- **Depositors** earn fees when their coins are used
- **Consumers** pay a small fee for instant coin acquisition
- **Registry** stays populated through market incentives

## Why This Matters

- **Solves Sui's type limitation**: Runtime coin acquisition without new deployments
- **Decimal matching**: Automatic precision alignment for asset/stable pairs
- **Permissionless**: Anyone can contribute, anyone can acquire
- **Single-transaction**: Acquire coins and use them in the same PTB
- **Content-safe**: Module naming restrictions prevent abuse
- **Economically sustainable**: Fee model incentivizes registry population

This pattern enables prediction markets, conditional tokens, and any protocol needing dynamic coin types without the overhead of per-use module deployment.

## What Would Actually Fix This

This pattern exists because Sui/Move lacks runtime type capabilities. Potential protocol-level solutions:

| Solution | Description | Feasibility |
|----------|-------------|-------------|
| **Runtime type creation** | `coin::create_dynamic(decimals, ctx)` returns opaque type handle | Major change to Move's type system |
| **PTB type interpolation** | Allow `Result` values as type arguments | Requires protocol-level tx execution changes |
| **Deferred type resolution** | Callback-based publish where type is bound within closure | Requires higher-order types in Move |

Until one of these exists, the Blank Coins Registry remains the only way to achieve single-transaction dynamic coin acquisition on Sui.

## Source Code

- [blank_coins.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy_one_shot_utils/sources/blank_coins.move)

---

*Next: [Runtime Package Authorization via Type Introspection](./05-registry-validated-witness.md)*
