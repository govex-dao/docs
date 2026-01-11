# Advanced Sui Design Patterns

*A series exploring Move patterns developed for the [Govex](https://github.com/govex-dao/govex) futarchy protocol.*

---

## Table of Contents

| # | Pattern | Problem Solved | Category |
|---|---------|----------------|----------|
| 1 | [Atomic Intent](#1-the-atomic-intent-pattern) | Object Stealing / Parameter Injection | Security |
| 2 | [Balance Wrappers](#2-balance-wrappers) | Type Explosion with N outcomes | Scaling |
| 3 | [Iterative Hot Potato](#3-the-iterative-hot-potato) | Generic Loop Constraints | Move Limitations |
| 4 | [Blank Coins](#4-the-blank-coins-pattern) | Runtime Coin Type Creation | Infrastructure |
| 5 | [Registry-Validated Witness](#5-registry-validated-witness) | Dynamic Package Authorization | Authorization |
| 6 | [Config Migration](#6-config-migration) | Zero-Downtime Schema Upgrades | Upgradability |
| 7 | [Resource Request Composition](#7-resource-request-composition) | In-Flight Asset Staging | Composition |
| 8 | [Atomic Object Accumulator](#8-the-atomic-object-accumulator) | Constructor Argument Limits | Scalability |

## Why These Patterns?

Building futarchy governance on Sui required solving problems at the intersection of:
- **Security**: Governance actions must be tamper-proof
- **Scalability**: Markets need N outcomes without N type parameters
- **Move Constraints**: Static generics vs. dynamic runtime needs

Each pattern addresses a fundamental limitation in Sui/Move development.

---

# 1. The Atomic Intent Pattern

*Preventing Object Stealing in Sui PTBs*

Traditional governance execution relies on "God Dispatchers" or passing arbitrary bytes to a VM, which are vulnerable to parameter injection and "Object Stealing" in a Programmable Transaction Block (PTB).

## The Problem

In a PTB, an executor can:
1. Call your contract with fake parameters
2. Intercept returned objects and redirect them
3. Reorder calls to manipulate state

Standard governance patterns that accept PTB arguments or return objects are fundamentally unsafe.

## The 3-Layer Architecture

Govex enforces parameter integrity by moving arguments from the PTB to immutable storage.

### Layer 1: Layered Determinism

Action parameters are BCS-serialized into an `ActionSpec`. Execution functions *must* peel data from this spec; they do not accept PTB arguments.

```move
// Parameters are embedded in the proposal, not passed at execution time
public struct ActionSpec has store {
    action_type: String,
    params: vector<u8>,  // BCS-encoded, immutable
}
```

### Layer 2: The Executable (Hot Potato)

A struct with no abilities is issued to track progress. It must be consumed in the same transaction, forcing atomic completion.

```move
// No abilities = must be consumed in same transaction
public struct Executable {
    proposal_id: ID,
    actions_remaining: u64,
}
```

### Layer 3: The Bag Pattern

To prevent an executor from "stealing" an object mid-PTB, actions never return objects. They move them into a `UID`-based "Bag" inside the Executable.

```move
// Secure Resource Passing
executable_resources::provide_coin(executable, b"asset", coin, ctx);

// Deterministic Retrieval
let coin = executable_resources::take_coin<Outcome, T>(executable, b"asset");
```

## Why This Matters

This pattern ensures:
- **No parameter injection**: Params come from storage, not the PTB
- **No object stealing**: Objects stay in the bag until explicitly retrieved
- **Atomic execution**: Hot potato forces same-transaction completion

**Source Code:**
- [executable.move](https://github.com/govex-dao/govex/blob/main/packages/smart_account/packages/protocol/sources/executable.move)
- [executable_resources.move](https://github.com/govex-dao/govex/blob/main/packages/smart_account/packages/protocol/sources/executable_resources.move)

---

# 2. Balance Wrappers

*Scalable Type Erasure for Multi-Outcome Markets*

Futarchy requires N outcomes. In Move, defining `Pool<T0, T1...TN>` leads to **Type Explosion**, making code unmanageable as N grows.

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

**Source Code:**
- [conditional_balance.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_primitives/sources/conditional/conditional_balance.move)
- [coin_escrow.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_primitives/sources/conditional/coin_escrow.move)

---

# 3. The Iterative Hot Potato

*PTB Loop Unrolling*

Move generics are static. You cannot write a loop that interacts with a variable number of unique types (e.g., `Coin<T0>`, `Coin<T1>`).

## The Problem

```move
// This is impossible in Move
for i in 0..n {
    let coin: Coin<OutcomeType[i]> = mint(...);  // Types can't be dynamic
}
```

Move requires all generic types to be known at compile time. But futarchy needs to mint N different conditional coin types in a single operation.

## The Solution: PTB Loop Unrolling

Govex "unrolls" these loops into the Programmable Transaction Block (PTB), using a **Progress Hot Potato** to ensure every unique type is processed sequentially.

### The Progress Guard

```move
// The Potato has no abilities - must be consumed
public struct SplitProgress {
    next_outcome: u64,
    total: u64
}
```

### Sequential Enforcement

Each PTB command handles **one** type and increments the index:

```move
// Step i: Handles unique type T_i
public fun split_step<..., T_i>(progress: &mut SplitProgress, ...): Coin<T_i> {
    assert!(index == progress.next_outcome, EWrongSequence);
    progress.next_outcome = progress.next_outcome + 1;
    // ... mint T_i ...
}
```

### Completion Check

The `finalize` function asserts the counter equals the total outcome count:

```move
public fun finalize(progress: SplitProgress) {
    let SplitProgress { next_outcome, total } = progress;
    assert!(next_outcome == total, EIncomplete);
    // Hot potato consumed - transaction can complete
}
```

### The PTB Structure

```typescript
// Client-side: "Unroll" the loop into PTB commands
const ptb = new TransactionBlock();

const progress = ptb.moveCall({ target: 'pkg::split::begin', arguments: [...] });

// Each type gets its own call
ptb.moveCall({ target: 'pkg::split::step', typeArguments: ['Outcome0'], arguments: [progress, ...] });
ptb.moveCall({ target: 'pkg::split::step', typeArguments: ['Outcome1'], arguments: [progress, ...] });
ptb.moveCall({ target: 'pkg::split::step', typeArguments: ['Outcome2'], arguments: [progress, ...] });

ptb.moveCall({ target: 'pkg::split::finalize', arguments: [progress] });
```

## Why This Matters

This pattern allows **Dynamic Runtime Logic** (handling N outcomes) to coexist with **Static Compile-Time Safety** (generic type parameters).

- **Enforces completeness**: Can't skip an outcome
- **Enforces order**: Must process sequentially
- **Atomic**: Hot potato ensures same-transaction execution
- **Type-safe**: Each step is fully typed

It is the only way to maintain a 1:N "Quantum Invariant" across an arbitrary number of conditional markets without hitting Move's type-parameter limits.

**Source Code:**
- [split_operations.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_operations/sources/split_operations.move)
- [merge_operations.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_operations/sources/merge_operations.move)

---

# 4. The Blank Coins Pattern

*Dynamic Coin Acquisition*

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

**Source Code:**
- [blank_coins.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_one_shot_utils/sources/blank_coins.move)

---

# 5. Registry-Validated Witness

*Runtime Package Authorization via Type Introspection*

Move's witness pattern provides compile-time authorization, but what if you need to authorize packages dynamically? Govex solves this with runtime type introspection against a registry.

## The Problem

Standard witness patterns hardcode authorization:

```move
// Compile-time authorization - can't add new packages without upgrade
public fun protected_action<W: drop>(_witness: W) {
    // Only packages that can instantiate W can call this
    // But W is fixed at compile time!
}
```

For governance systems, you need to add/remove authorized packages via proposals, not contract upgrades.

## The Solution: Registry-Validated Witnesses

Extract the package address from a witness type at runtime and verify it against a dynamic registry.

### The Core Mechanism

```move
fun assert_authorized_witness<W: drop>(registry: &PackageRegistry) {
    // Extract package address from witness type
    let witness_type = type_name::with_defining_ids<W>();
    let package_addr_string: AsciiString = type_name::address_string(&witness_type);
    let package_addr = address::from_ascii_bytes(package_addr_string.as_bytes());

    // Verify against dynamic registry
    assert!(
        package_registry::is_authorized_package(registry, package_addr),
        EUnauthorizedWitness
    );
}
```

### How It Works

1. Caller provides any witness type `W` from their package
2. `type_name::with_defining_ids<W>()` captures the full type info including package address
3. `type_name::address_string()` extracts just the package address
4. Registry lookup verifies the package is authorized
5. If authorized, proceed with the protected operation

This single mechanism enables two powerful use cases.

---

## Use Case A: Governance Sponsorship

Authorize specific packages to sponsor governance proposals:

```move
public struct SponsorshipAuth has drop {}

public fun create<W: drop>(registry: &PackageRegistry, _witness: W): SponsorshipAuth {
    assert_authorized_witness<W>(registry);
    SponsorshipAuth {}
}

// In authorized package (futarchy_governance):
struct Witness has drop {}

public fun sponsor_proposal(...) {
    // Create auth by providing our package's witness
    let auth = sponsorship_auth::create(registry, Witness {});

    // Use auth to call protected function
    proposal::sponsor_with_auth(auth, ...);
}
```

---

## Use Case B: Permissionless Vault Access

Sometimes you need to allow **anyone** to trigger a sensitive action (like post-dissolution redemption), but only specific code paths should execute the withdrawal.

### The Witness Definition

```move
// In dissolution_actions.move
// Only this module can instantiate
public struct DissolutionWitness has drop {}
```

### The Protected Vault Function

```move
// In vault.move
/// Withdraw without governance approval - requires package witness
public fun withdraw_permissionless<Config, CoinType, W: drop>(
    _witness: W,  // Caller must provide witness
    account: &mut Account,
    registry: &PackageRegistry,
    ...
): Coin<CoinType> {
    // Verify witness package is authorized in registry
    let witness_type = type_name::with_defining_ids<W>();
    let package_addr = extract_package_address(witness_type);
    assert!(is_permissionless_authorized(registry, package_addr), EUnauthorized);

    // Proceed with withdrawal...
}
```

### The Permissionless Entry Point

```move
/// Anyone can call this - no special permissions needed
public fun redeem<Config: store, AssetType, RedeemCoinType>(
    capability: &DissolutionCapability,
    account: &mut Account,
    registry: &PackageRegistry,
    asset_coins: Coin<AssetType>,  // User provides their tokens
    vault_name: String,
    clock: &Clock,
    ctx: &mut TxContext,
): Coin<RedeemCoinType> {
    // Safety checks...
    assert!(capability.dao_address == account.addr(), EWrongAccount);
    assert!(clock.timestamp_ms() >= capability.unlock_at_ms, ETooEarly);

    // Burn user's tokens first
    currency::public_burn<Config, AssetType>(account, registry, asset_coins);

    // Withdraw using our package witness
    vault::withdraw_permissionless<Config, RedeemCoinType, DissolutionWitness>(
        DissolutionWitness {},  // Only WE can instantiate this
        account,
        registry,
        ...
    )
}
```

### The Security Model

```
User calls redeem()
        |
        v
+------------------------------+
|  dissolution_actions module  |
|  - validates capability      |
|  - burns user's tokens       |
|  - creates DissolutionWitness|
+------------------------------+
        |
        v (passes witness)
+------------------------------+
|       vault module           |
|  - checks witness package    |
|  - executes withdrawal       |
+------------------------------+
```

- **Users** can call `redeem()` freely
- **Attackers** cannot call `withdraw_permissionless()` directly (can't create witness)
- **Only dissolution_actions** can create `DissolutionWitness`

---

## Why This Matters

- **Dynamic authorization**: Add/remove packages via governance proposals
- **No contract upgrades**: Registry is mutable, not hardcoded
- **Permissionless UX**: Users can trigger "admin" actions if they pass through correct code gates
- **Type-safe**: Still uses Move's witness pattern under the hood
- **Audit-friendly**: Package address extraction is deterministic

### Breaking Circular Dependencies

This pattern also solves a structural problem: when `package_A` needs to authorize calls from `package_B`, but both depend on each other. By moving the auth module to a leaf package and using runtime registry lookups, you break the cycle.

```
Before:
  futarchy_markets_core <-> futarchy_governance (circular!)

After:
  futarchy_core (leaf, has sponsorship_auth)
       ^                    ^
  futarchy_markets_core    futarchy_governance
       (uses auth)         (creates auth)
```

**Source Code:**
- [sponsorship_auth.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_core/sources/sponsorship_auth.move)
- [dissolution_actions.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_actions/sources/dissolution/dissolution_actions.move)
- [package_registry.move](https://github.com/govex-dao/govex/blob/main/packages/smart_account/packages/protocol/sources/package_registry.move)

---

# 6. Config Migration

*TypeName Tracking for Zero-Downtime Schema Upgrades*

How do you upgrade a DAO's config schema without breaking existing intents or requiring account recreation? Govex tracks config types at runtime using TypeName, enabling safe migrations.

## The Problem

Move structs are immutable once deployed. If your Account stores `ConfigV1` and you need to add fields, you can't just modify the struct. You need a migration path.

```move
// Version 1 - deployed
public struct ConfigV1 has store {
    threshold: u64,
}

// Version 2 - needs new field
public struct ConfigV2 has store {
    threshold: u64,
    quorum: u64,  // New field!
}
```

The naive approach fails:
```move
// This aborts - stored type is ConfigV1, not ConfigV2
let config: &ConfigV2 = df::borrow(&account.id, ConfigKey {});
```

## The Solution: Dual Dynamic Field Storage

Store both the config data AND its TypeName in separate dynamic fields.

### Storage Structure

```move
public struct ConfigKey has copy, drop, store {}
public struct ConfigTypeKey has copy, drop, store {}

// At account creation:
df::add(&mut account.id, ConfigKey {}, config);
df::add(&mut account.id, ConfigTypeKey {}, type_name::get<Config>());
```

### The Migration Function

```move
public(package) fun migrate_config<OldConfig: store, NewConfig: store>(
    account: &mut Account,
    registry: &PackageRegistry,
    new_config: NewConfig,
    version_witness: VersionWitness,
): OldConfig {
    // 1. Verify caller is authorized
    account.deps().check(version_witness, registry, account.account_deps());

    // 2. Validate OldConfig matches stored type
    let stored_type = df::borrow<ConfigTypeKey, TypeName>(&account.id, ConfigTypeKey {});
    let old_type = type_name::get<OldConfig>();
    assert!(&old_type == stored_type, EWrongConfigType);

    // 3. Atomically swap config
    let old_config: OldConfig = df::remove(&mut account.id, ConfigKey {});
    df::add(&mut account.id, ConfigKey {}, new_config);

    // 4. Update type tracking
    let _: TypeName = df::remove(&mut account.id, ConfigTypeKey {});
    df::add(&mut account.id, ConfigTypeKey {}, type_name::get<NewConfig>());

    // 5. Return old config for validation/destruction
    old_config
}
```

### Usage

```move
// Migration action in a governance proposal
public fun do_migrate_config<OldConfig, NewConfig, Outcome>(
    executable: &mut Executable<Outcome>,
    account: &mut Account,
    registry: &PackageRegistry,
    new_config: NewConfig,
    version: VersionWitness,
) {
    let old_config: OldConfig = account::migrate_config(
        account,
        registry,
        new_config,
        version,
    );

    // Validate old config before dropping
    validate_migration(&old_config);
    // old_config dropped here (must have drop ability or be consumed)
}
```

## Runtime Type Checking

The TypeName field also enables runtime type validation for reads:

```move
/// Get config type for runtime checks
public fun config_type(account: &Account): TypeName {
    *df::borrow<ConfigTypeKey, TypeName>(&account.id, ConfigTypeKey {})
}

// Usage: verify before borrowing
let expected = type_name::get<FutarchyConfigV2>();
assert!(account::config_type(account) == expected, EWrongVersion);
let config: &FutarchyConfigV2 = account::config(account);
```

## Why This Matters

- **Zero-downtime upgrades**: Migrate config without account recreation
- **Active intent safety**: Intents in-flight are unaffected by config changes
- **Type-safe migrations**: Compiler catches mismatched old/new types
- **Audit trail**: Old config returned for validation before destruction
- **Runtime introspection**: Check config version without knowing the type

## Migration Best Practices

1. **Return old config**: Let caller validate the migration
2. **Atomic swap**: Remove old, add new in same function
3. **Version witness**: Ensure authorized packages only
4. **Type tracking**: Always update ConfigTypeKey

**Source Code:**
- [account.move](https://github.com/govex-dao/govex/blob/main/packages/smart_account/packages/protocol/sources/account.move)

---

# 7. Resource Request Composition

*In-Flight Asset Staging*

Complex governance actions often need assets to flow between multiple operations within a single transaction. Govex uses Resource Requests to stage assets in-flight without intermediate vault deposits.

## The Problem

Consider a liquidity rebalance: remove LP from Pool A, add to Pool B. The naive approach:

```move
// Step 1: Remove liquidity -> coins go to vault
vault::deposit(vault, asset_coin);
vault::deposit(vault, stable_coin);

// Step 2: Withdraw from vault -> add to new pool
let asset = vault::withdraw(vault, ...);
let stable = vault::withdraw(vault, ...);
pool_b::add_liquidity(asset, stable);
```

This works but:
- Extra gas for vault round-trips
- Requires vault permissions for intermediate state
- Complex to compose in a PTB

## The Solution: Resource Requests

Create a hot potato that stages assets in-flight between actions.

### The Request Structure

```move
/// Hot potato - MUST be fulfilled in same transaction
#[allow(lint(missing_key))]
public struct ResourceRequest<phantom T> {
    id: UID,
    context: UID,  // Dynamic fields for flexible data
}

/// Completion marker
public struct ResourceReceipt<phantom T> has drop {
    request_id: ID,
}
```

### The Flow: Request -> Fulfill

**Step 1: Action returns a request**

```move
public fun do_remove_liquidity_to_resources<AssetType, StableType, LPType, Outcome>(
    executable: &mut Executable<Outcome>,
    ...
): ResourceRequest<RemoveLiquidityToResourcesAction<AssetType, StableType, LPType>> {
    // Validate action, parse params...
    let action = RemoveLiquidityToResourcesAction { ... };

    // Return hot potato with action stored inside
    resource_requests::new_resource_request(action, ctx)
}
```

**Step 2: Fulfillment provides resources to executable**

```move
public fun fulfill_remove_liquidity_to_resources<...>(
    request: ResourceRequest<RemoveLiquidityToResourcesAction<...>>,
    executable: &mut Executable<Outcome>,
    spot_pool: &mut UnifiedSpotPool<...>,
    ...
): ResourceReceipt<RemoveLiquidityToResourcesAction<...>> {
    // Extract action from request
    let action = resource_requests::extract_action(request);

    // Take LP coin from prior action's output
    let lp_coin: Coin<LPType> = executable_resources::take_coin(
        executable::uid_mut(executable),
        action.lp_resource_name,
    );

    // Execute: remove liquidity
    let (asset_coin, stable_coin) = unified_spot_pool::remove_liquidity(
        spot_pool, lp_coin, ...
    );

    // Stage outputs for NEXT action
    executable_resources::provide_coin(
        executable::uid_mut(executable),
        action.asset_output_name,  // "rebalance_asset"
        asset_coin,
        ctx,
    );
    executable_resources::provide_coin(
        executable::uid_mut(executable),
        action.stable_output_name,  // "rebalance_stable"
        stable_coin,
        ctx,
    );

    // Return receipt (drops action)
    resource_requests::create_receipt(action)
}
```

**Step 3: Next action consumes staged resources**

```move
public fun fulfill_add_liquidity<...>(
    request: ResourceRequest<AddLiquidityAction<...>>,
    executable: &mut Executable<Outcome>,
    ...
): ResourceReceipt<AddLiquidityAction<...>> {
    let action = resource_requests::extract_action(request);

    // Take coins staged by prior action
    let asset_coin: Coin<AssetType> = executable_resources::take_coin(
        executable::uid_mut(executable),
        action.asset_resource_name,  // "rebalance_asset"
    );
    let stable_coin: Coin<StableType> = executable_resources::take_coin(
        executable::uid_mut(executable),
        action.stable_resource_name,  // "rebalance_stable"
    );

    // Add to new pool...
}
```

## The PTB Structure

```typescript
const ptb = new TransactionBlock();

// Action 1: Request removal
const removeRequest = ptb.moveCall({
    target: 'pkg::liquidity_actions::do_remove_liquidity_to_resources',
    arguments: [executable, ...],
});

// Fulfill: execute removal, stage coins
const removeReceipt = ptb.moveCall({
    target: 'pkg::liquidity_actions::fulfill_remove_liquidity_to_resources',
    arguments: [removeRequest, executable, poolA, ...],
});

// Action 2: Request add
const addRequest = ptb.moveCall({
    target: 'pkg::liquidity_actions::do_add_liquidity',
    arguments: [executable, ...],
});

// Fulfill: consume staged coins, add to pool
const addReceipt = ptb.moveCall({
    target: 'pkg::liquidity_actions::fulfill_add_liquidity',
    arguments: [addRequest, executable, poolB, ...],
});
```

## Why This Matters

- **No vault round-trips**: Assets flow directly between actions
- **Composable**: Chain unlimited actions with resource names
- **Type-safe**: Phantom types ensure correct pairing
- **Flexible**: Dynamic fields in context allow any data

**Note:** This pattern applies to any resource flow between PTB steps—owned objects, coins, gaming items (Ore -> Bar -> Sword), flash loans. The core insight: use the hot potato's dynamic fields as a temporary carrier bag.

## vs. Executable Resources Alone

| Pattern | Use Case |
|---------|----------|
| Executable Resources | Simple staging within one action |
| Resource Requests | Multi-action composition with deferred execution |

Resource Requests add the hot potato layer for two-phase execution (request -> fulfill), which is useful when the action definition and resource provision are separate steps.

**Source Code:**
- [resource_requests.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_core/sources/resource_requests.move)
- [liquidity_actions.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_actions/sources/liquidity/liquidity_actions.move)

---

# 8. The Atomic Object Accumulator

*Scalable Heterogeneous Construction*

While Pattern #3 (Iterative Hot Potato) handles logic loops (doing X operation N times), this pattern handles **construction scalability**: building an object that requires more resources than a single function call can accept.

## The Problem: Constructor Argument Limits & Heterogeneous Types

In Move, transaction functions have practical limits on argument count. Furthermore, `vector` collections must be homogeneous—all elements must share the same type.

Consider a Proposal that needs 10 different conditional coin types:

```move
// This is IMPOSSIBLE in Move:
// - 40+ arguments exceed practical limits
// - vector<TreasuryCap<T>> can't hold different T types
public fun create_proposal(
    cap0: TreasuryCap<Cond0Asset>, meta0: MetadataCap<Cond0Asset>,
    cap1: TreasuryCap<Cond0Stable>, meta1: MetadataCap<Cond0Stable>,
    cap2: TreasuryCap<Cond1Asset>, meta2: MetadataCap<Cond1Asset>,
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
+------------------------------------------------------------------+
|                         Single PTB                                |
+------------------------------------------------------------------+
|  1. BEGIN         -> Create unshared object with empty Bags       |
|  2. ACCUMULATE    -> Call add_outcome N times (heterogeneous)     |
|  3. FINALIZE      -> Validate completeness, create AMMs, share    |
+------------------------------------------------------------------+
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
    asset_meta: MetadataCap<AC>,
    stable_cap: TreasuryCap<SC>,
    stable_meta: MetadataCap<SC>,
    // ...
) {
    // Bag allows mixing MetadataCap<Cond0Asset> with MetadataCap<Cond1Asset>
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
    // 40 objects: 10 * (TreasuryCap + MetadataCap) * 2 coin types
    ac0: TreasuryCap<AC0>, am0: MetadataCap<AC0>,
    sc0: TreasuryCap<SC0>, sm0: MetadataCap<SC0>,
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

**Source Code:**
- [proposal.move - begin_proposal](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_core/sources/proposal.move#L1220)
- [proposal.move - add_outcome_coins_10](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_core/sources/proposal.move#L1542)
- [proposal.move - finalize_proposal](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_core/sources/proposal.move#L1688)

---

*All patterns are implemented in the [Govex repository](https://github.com/govex-dao/govex).*
