|   SIP-Number | |
|         ---: | :--- |
|        Title | Package Policy Introspection |
|  Description | Native functions to programmatically check upgrade policy, version, and immutability status of any package on-chain |
|       Author | Greshamscode, @92GC |
|       Editor | <Leave this blank; it will be assigned by a SIP Editor> |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-06-25 |
| Comments-URI | |
|       Status | |
|     Requires | |

## Abstract

This proposal introduces native functions to the `sui::package` module that allow any smart contract to programmatically check the upgrade policy, version, and immutability status of any package on-chain.

Currently, this information is only accessible to the holder of the `UpgradeCap`. This SIP enables "Trustless Verification of Third-Party Logic," allowing protocols (like DAOs) to enforce security requirements on the external modules they interact with.

## Motivation

Sui's upgrade system is powerful, but it is "opaque" to other Move contracts.

### The Problem
A DAO protocol allows third-party developers to register "Action Modules." To ensure the safety of its users, the DAO wants to enforce a rule: **"Any module registered here must be either Immutable or Additive-Only."**

Currently, this is impossible to verify on-chain because:
1.  `sui::package::upgrade_policy` requires a reference to the `UpgradeCap`.
2.  Third-party developers usually keep their `UpgradeCap`.
3.  Even if the developer claims the package is immutable, the DAO has no native Move function to verify `is_immutable(package_id)`.

### The Security Gap
Without package introspection, a developer can:
1.  Register a "Compatible" (mutable) package with a DAO.
2.  Wait for the DAO to approve a high-value proposal.
3.  Upgrade the package logic to redirect funds right before execution.
4.  The DAO cannot programmatically detect that the package was "Mutable" at the time of registration.

## Specification

We propose adding the following native functions to `sui::package`:

```move
module sui::package {
    /// Returns the current upgrade policy of the package.
    /// 0 = COMPATIBLE (Full logic changes allowed)
    /// 1 = ADDITIVE (Only new functions/structs allowed)
    /// 2 = DEP_ONLY (Only dependency changes allowed)
    /// 255 = IMMUTABLE (No upgrades possible)
    public native fun upgrade_policy(package_id: address): u8;

    /// Returns the current version (upgrade count) of the package.
    /// Returns 1 for initial deployments, 2 for the first upgrade, etc.
    public native fun upgrade_version(package_id: address): u64;

    /// Returns true if the package is immutable and cannot be upgraded.
    public native fun is_immutable(package_id: address): bool;
}
```

## Rationale

### Why Native?
The Sui network already stores this information in the `Package` object metadata within the global state. However, the Move VM does not currently expose this metadata to the execution context. A native function is required to "bridge" this metadata into Move.

### Why not just use the `UpgradeCap`?
The `UpgradeCap` is a permissioned object. Requiring it for introspection prevents **Permissionless Verification**.
A user or a DAO should be able to verify the security properties of a package without needing the owner's permission or the owner's capability.

### Policy Mapping
By returning the `u8` policy, contracts can make logic-based decisions:
```move
public fun register_module(pkg_id: address) {
    let policy = package::upgrade_policy(pkg_id);
    // Only allow Additive (1) or Immutable (255)
    assert!(policy == 1 || policy == 255, EUnsafeUpgradePolicy);
}
```

## Backwards Compatibility

This change is purely additive. It introduces new native functions but does not change the behavior of existing `UpgradeCap` functions or the upgrade flow itself.

## Test Cases

To be developed.

## Reference Implementation

To be developed.

## Security Considerations

1.  **Read-Only Access**: These functions are strictly read-only. They do not grant any power to upgrade a package, only to observe its current state.
2.  **State Consistency**: The values returned must reflect the state of the package at the start of the current transaction to prevent "flash-upgradability" tricks (though Sui's upgrade model generally prevents this as upgrades happen at the end of a checkpoint/transaction).

## Copyright

Greshamscode 2025
