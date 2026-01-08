# Object Vault and Spam Protection

## Problem

Sui allows anyone to `transfer::public_transfer(obj, address)` to any address. This means:

1. Anyone can spam a DAO account with millions of junk objects
2. Enumerating owned objects becomes impractical
3. Objects/coins sent directly to the account address can become **effectively lost**

### Why Discovery Matters

To receive an owned object, you need its exact ID:

```move
public fun do_withdraw_coin<CoinType>(
    receiving: Receiving<Coin<CoinType>>,  // Must provide exact object ID
    ...
)
```

Discovery relies on RPCs like `getOwnedObjects(address)` which:
- Paginate through all objects owned by the address
- Have limits (e.g., 50 objects per page, timeouts)
- With 10M spam objects, real assets are buried in noise

**If you can't find the ID, you can't receive it. The asset is effectively lost.**

Example: Someone sends 1000 SUI directly to the DAO address (not through vault). With spam pollution, you may never find that coin's ID to withdraw it.

## Current Architecture (Spam-Safe)

All critical assets use Dynamic Fields (DF) or Dynamic Object Fields (DOF), accessed by typed keys:

| Storage | Key Type | What's Stored |
|---------|----------|---------------|
| `ConfigKey {}` | DF | FutarchyConfig |
| `VaultKey(name)` | DF | Vault (Bag of Balances) |
| `UpgradeCapKey(name)` | DOF | UpgradeCap |
| `TreasuryCapKey<T>()` | DOF | TreasuryCap |
| `MetadataCapKey<T>()` | DOF | MetadataCap |
| `CapKey<Cap>()` | DOF | Any AdminCap (access_control) |

These are O(1) lookup by key. Spam at the account address doesn't affect them.

## What's At Risk

`owned.move` uses Transfer-to-Object (TTO) pattern with `Receiving<T>`:

- `do_withdraw_object` - Withdraw object by ID
- `do_withdraw_coin` - Withdraw coin by type/amount
- `merge_and_split` - Merge/split coins

**The core issue:** These require you to already know the object ID. If you can't discover IDs (due to spam), assets are effectively frozen.

Note: Currently nothing in futarchy actually uses `owned.move` - all funds flow through vaults. But the risk exists for:
- Accidental direct transfers to DAO address
- External parties sending assets incorrectly
- Any future features that rely on owned objects

## Proposed Solutions

### 1. Named Object Vault

Like `vault.move` but for non-fungible objects:

```move
public struct ObjectVaultKey(String) has copy, drop, store;

public struct ObjectVault has store {
    objects: Bag, // String name -> object
}
```

Usage:
- `deposit_object<T>(vault_name, object_name, obj)` - Governance deposits named object
- `withdraw_object<T>(vault_name, object_name)` - Governance withdraws by name

Benefits:
- Named, enumerable objects within vault
- Governance-controlled deposits/withdrawals
- Spam-proof (DF storage)

### 2. Personal Storage Slots

For permissionless deposits (e.g., someone submitting an NFT for DAO consideration):

```move
public struct PersonalStorageKey(address) has copy, drop, store;
```

- Anyone deposits to their own slot (keyed by sender address)
- Only depositor can withdraw
- Governance can "accept" by moving to ObjectVault

Benefits:
- Permissionless but not spammable (one slot per address)
- Clear ownership during pending state
- DAO explicitly accepts what it wants

### 3. Deprecate `owned.move`

Mark `owned.move` as legacy escape hatch only:
- Keep for emergency "someone sent us something" recovery
- Document that new features should use DF-based storage
- Add migration path for any owned objects → ObjectVault

## Implementation Priority

1. **ObjectVault** - Core feature for storing DAO assets (AdminCaps, NFTs, etc.)
2. **Personal Storage** - Nice-to-have for donation/submission flows
3. **owned.move deprecation** - Documentation and migration tooling

## Completed Work

**Removed ObjectTracker** (Jan 2026):
- Removed `ObjectTracker` struct and all related functions from `account.move`
- Removed deposit/whitelist config actions from `config.move`
- Simplified `keep()` to just `transfer::public_transfer(obj, account.addr())`
- Removed tracking from `receive()` - now just calls `transfer::public_receive`

The ObjectTracker was security theater - it only protected deposits going through `account.keep()`, but anyone could bypass it via `transfer::public_transfer()`. The real protection is using DFs/DOFs for critical assets, which we already do.

## Related

- `access_control.move` - Already uses `CapKey<Cap>()` for arbitrary caps
- `vault.move` - Template for ObjectVault design
- `owned.move` - Legacy pattern, kept for emergency recovery only
