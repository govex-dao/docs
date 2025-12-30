# Dynamic Coin Acquisition: The Blank Coin Registry Pattern

*Part 4 of the [Advanced Sui Design Patterns](./README.md) series*

---

Sui coin types must be defined at compile time. You cannot create `Coin<MyNewToken>` dynamically at runtime. This is a fundamental limitation for protocols that need N conditional token types per proposal.

**Why not publish in the same PTB?** Type arguments must be string literals at build time—you can't interpolate a `Result` from `tx.publish()` into a `moveCall`'s type parameters. This pattern is the only known way to achieve single-click market creation on Sui.

## The Problem

```move
// This is impossible - types can't be created at runtime
public fun create_market(outcome_count: u64) {
    for i in 0..outcome_count {
        let coin_type = create_new_coin_type();  // Can't do this
    }
}
```

Every conditional market needs unique coin types for each outcome. Deploying new coin modules per-proposal doesn't scale.

## The Solution: Pre-Created Blank Coins

Govex uses a **permissionless registry** of pre-created "blank" coin types that can be acquired on-demand.

### The Registry Structure

```move
public struct CoinSet<phantom T> has store {
    treasury_cap: TreasuryCap<T>,
    metadata: CoinMetadata<T>,
    owner: address,    // Who deposited (gets paid on acquisition)
    fee: u64,          // Fee in SUI to acquire
}

public struct CoinRegistry has key {
    id: UID,
    total_sets: u64,
    // CoinSets stored as dynamic fields with cap_id as key
}
```

### Blank Coin Validation

Coins must be truly "blank" - empty metadata that consumers can customize:

```move
fun validate_coin_set_for_registry<T>(
    treasury_cap: &TreasuryCap<T>,
    metadata: &CoinMetadata<T>
) {
    assert!(treasury_cap.total_supply() == 0, ESupplyNotZero);
    assert!(metadata.get_name().is_empty(), ENameNotEmpty);
    assert!(metadata.get_symbol().is_empty(), ESymbolNotEmpty);
    assert!(metadata.get_description().is_empty(), EDescriptionNotEmpty);
    assert!(metadata.get_icon_url().is_none(), EIconUrlNotEmpty);
}
```

### Permissionless Deposit

Anyone can contribute coin sets to the registry:

```move
public fun deposit_coin_set<T>(
    registry: &mut CoinRegistry,
    treasury_cap: TreasuryCap<T>,
    metadata: CoinMetadata<T>,
    fee: u64,
    clock: &Clock,
    ctx: &TxContext,
) {
    assert!(registry.total_sets < MAX_COIN_SETS, ERegistryFull);
    assert!(fee <= MAX_FEE, EFeeExceedsMaximum);  // DoS protection

    validate_coin_set_for_registry(&treasury_cap, &metadata);

    let coin_set = CoinSet { treasury_cap, metadata, owner: ctx.sender(), fee };
    dynamic_field::add(&mut registry.id, object::id(&treasury_cap), coin_set);
}
```

### PTB-Friendly Acquisition

Coin sets can be acquired and used in the same transaction:

```move
public fun take_coin_set_for_ptb<T>(
    registry: &mut CoinRegistry,
    cap_id: ID,
    mut fee_payment: Coin<SUI>,
    clock: &Clock,
    ctx: &mut TxContext,
): (TreasuryCap<T>, CoinMetadata<T>, Coin<SUI>) {
    let coin_set: CoinSet<T> = dynamic_field::remove(&mut registry.id, cap_id);

    // Pay the depositor
    let payment = fee_payment.split(coin_set.fee, ctx);
    transfer::public_transfer(payment, coin_set.owner);

    let CoinSet { treasury_cap, metadata, .. } = coin_set;
    (treasury_cap, metadata, fee_payment)  // Return for use in same PTB
}
```

## Economic Design

### DoS Protection

Without fee limits, attackers could fill the registry with unusable coin sets:

```move
const MAX_FEE: u64 = 10_000_000_000;  // 10 SUI max

assert!(fee <= MAX_FEE, EFeeExceedsMaximum);
```

### Incentive Alignment

- **Depositors** earn fees when their coins are used
- **Consumers** pay a small fee for instant coin acquisition
- **Registry** stays populated through market incentives

## Why This Matters

- **Solves Sui's type limitation**: Runtime coin acquisition without new deployments
- **Permissionless**: Anyone can contribute, anyone can acquire
- **Single-transaction**: Acquire coins and use them in the same PTB
- **Economically sustainable**: Fee model incentivizes registry population

This pattern enables prediction markets, conditional tokens, and any protocol needing dynamic coin types without the overhead of per-use module deployment.

## Source Code

- [coin_registry.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy_one_shot_utils/sources/coin_registry.move)

---

*Next: [Runtime Package Authorization via Type Introspection](./05-registry-validated-witness.md)*
