# Runtime Package Authorization via Type Introspection

*Part 5 of the [Advanced Sui Design Patterns](./README.md) series*

---

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
        │
        ▼
┌──────────────────────────────┐
│  dissolution_actions module  │
│  - validates capability      │
│  - burns user's tokens       │
│  - creates DissolutionWitness│
└──────────────────────────────┘
        │
        ▼ (passes witness)
┌──────────────────────────────┐
│       vault module           │
│  - checks witness package    │
│  - executes withdrawal       │
└──────────────────────────────┘
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
  futarchy_markets_core ←→ futarchy_governance (circular!)

After:
  futarchy_core (leaf, has sponsorship_auth)
       ↑                    ↑
  futarchy_markets_core    futarchy_governance
       (uses auth)         (creates auth)
```

## Source Code

- [sponsorship_auth.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_core/sources/sponsorship_auth.move)
- [dissolution_actions.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_actions/sources/dissolution/dissolution_actions.move)
- [package_registry.move](https://github.com/govex-dao/govex/blob/main/packages/smart_account/packages/protocol/sources/package_registry.move)

---

*Next: [Config Migration with TypeName Tracking](./06-config-migration.md)*
