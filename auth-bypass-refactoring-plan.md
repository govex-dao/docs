# Auth Bypass Pattern Refactoring Plan

**Date:** 2026-01-28
**Status:** Accepted for v1

## Summary

This document identifies all functions that bypass the standard 3-layer action pattern (Intent → Auth → Execute) and documents their security model.

---

## Key Finding: Most `*_unshared` Functions Are Already Secure

After investigation, we found that most `*_unshared` functions:
1. Require `VersionWitness` (only creatable by registered packages)
2. Have `assert_not_initialized(account)` runtime check
3. Are called from external futarchy packages → **cannot be `public(package)`**

The existing security model is sufficient.

---

## Accepted for v1

### 1. `withdraw_permissionless`

**File:** `packages/smart_account/actions/sources/lib/vault.move:441`

**Current Pattern:**
```move
public fun withdraw_permissionless<Config: store, CoinType, W: drop>(
    _witness: W,
    account: &mut Account,
    registry: &PackageRegistry,
    ...
): Coin<CoinType>
```

**Risk:** Any registered package with a `W: drop` witness can drain vaults.

**Current Callers (only 2):**
- `futarchy_actions::dissolution_actions` (DissolutionWitness)
- `futarchy_actions::liquidity_init_actions` (LiquidityWitness)

**Why Accepted for v1:**
- Only 2 known witnesses exist in controlled packages
- Package upgrades require UpgradeCap (controlled)
- No untrusted packages are registered

**v2 Fix: WithdrawWitnessRegistry**
```move
public struct WithdrawWitnessRegistry has key {
    id: UID,
    allowed_witnesses: VecSet<TypeName>,
}

// Register during init:
registry.allow<DissolutionWitness>();
registry.allow<LiquidityWitness>();

// Check in withdraw_permissionless:
assert!(registry.contains(type_name::get<W>()), EUnauthorizedWitness);
```

~~**Alternative: RedemptionPool Pattern**~~
```move
/// (More complex, not needed for v1)
public struct RedemptionPool<phantom CoinType> has key {
    id: UID,
    dao_id: ID,
    balance: Balance<CoinType>,
    total_asset_supply: u64,  // Snapshot at dissolution
}

/// Users claim directly from pool, never touching Account
public fun claim<CoinType, AssetType>(
    pool: &mut RedemptionPool<CoinType>,
    asset_coins: Coin<AssetType>,
    ctx: &mut TxContext,
): Coin<CoinType>
```

**Benefits:**
- Account vault is never touched after dissolution
- No witness pattern needed
- Cleaner separation of concerns
- Pool can be consumed/destroyed when empty

---

## Functions Already Secure (No Changes Needed)

### `*_unshared` Functions

These functions have two layers of protection:
1. **`VersionWitness` requirement** - Only registered packages can create this
2. **`assert_not_initialized(account)`** - Runtime check blocks calls on shared accounts

| Function | Protection | Called From |
|----------|------------|-------------|
| `do_deposit_unshared` | VersionWitness + assert_not_initialized | futarchy_actions, futarchy_factory |
| `do_spend_unshared` | VersionWitness + assert_not_initialized | (unused) |
| `create_stream_unshared` | assert_not_initialized | internal only → `public(package)` ✅ |
| `do_lock_cap_unshared` (access_control) | assert_not_initialized | integration_tests |
| `do_lock_cap_unshared` (currency) | assert_not_initialized | tests |
| `do_lock_metadata_cap_unshared` | assert_not_initialized | futarchy_factory |
| `do_remove_treasury_cap_unshared` | assert_not_initialized | internal |
| `do_remove_metadata_cap_unshared` | assert_not_initialized | (unused) |

---

## Completed Fixes

| Fix | Status |
|-----|--------|
| `add_dep_no_auth_check` → `public(package)` | ✅ Done |
| `create_stream_unshared` → `public(package)` | ✅ Done |
| Magic number → `u64::max_value!()` | ✅ Done |
| Document `Builder` copy ability | ✅ Done |
| Document `PackageAdminCap` store ability | ✅ Done |

---

## Security Model Summary

| Function | Auth Method | Risk | Status |
|----------|-------------|------|--------|
| `withdraw_permissionless` | Any witness from registered package | **High** | Needs RedemptionPool refactor |
| `*_unshared` with VersionWitness | VersionWitness + runtime check | Low | Already secure |
| `*_unshared` without VersionWitness | Runtime check only | Medium | Acceptable for init |
| `add_dep_no_auth_check` | Caller responsibility | Low | Fixed to `public(package)` |

---

## Notes

The `withdraw_permissionless` pattern is problematic because:
1. Accepts any witness `W: drop` from any registered package
2. Works on shared accounts (no init check)
3. Can be called at any time, not just during init
4. Security depends on ALL registered packages being trustworthy

This is the only remaining architectural issue that needs v2 refactoring.
