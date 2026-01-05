|   SIP-Number | |
|         ---: | :--- |
|        Title | Permissionless Package Introspection |
|  Description | Adds native functions to sui::package for querying package metadata without holding the UpgradeCap |
|       Author | Greshamscode, @92GC |
|       Editor | <Leave this blank; it will be assigned by a SIP Editor> |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-06-25 |
| Comments-URI | |
|       Status | |
|     Requires | |

## Abstract

This proposal adds native functions to `sui::package` that allow any smart contract to query package metadata—including upgrade policy, version, immutability status, and the currently executing package ID—without holding the `UpgradeCap`.

Currently, all package introspection requires the `UpgradeCap`, making it impossible for protocols to programmatically verify security properties of third-party packages. This SIP enables **permissionless verification**, allowing DAOs and other protocols to enforce security requirements on external modules and pin execution to specific audited package versions.

## Motivation

### The Permissioning Gap

Sui's `sui::package` module provides functions to query upgrade policy, version, and package ID—but they all require a reference to the `UpgradeCap`:

```move
// Existing API - requires UpgradeCap
public fun upgrade_policy(cap: &UpgradeCap): u8;
public fun version(cap: &UpgradeCap): u64;
public fun upgrade_package(cap: &UpgradeCap): ID;
```

Third-party developers keep their `UpgradeCap`. This creates two critical security gaps:

### Gap 1: Cannot Verify Third-Party Package Security

A DAO protocol allows third-party developers to register "Action Modules." To protect users, the DAO wants to enforce: **"Any module registered here must be either Immutable or Additive-Only."**

This is impossible to verify on-chain because:
1. `sui::package::upgrade_policy` requires a reference to the `UpgradeCap`
2. Third-party developers keep their `UpgradeCap`
3. Even if a developer claims immutability, there's no native Move function to verify `is_immutable(package_id)`

**Attack vector:** A developer registers a "Compatible" (mutable) package, waits for the DAO to approve a high-value proposal, then upgrades the package logic to redirect funds right before execution.

### Gap 2: Cannot Pin Execution to Audited Versions

Sui's upgrade model provides seamless data continuity—objects created in V1 remain compatible with V2 because `TypeName` stays anchored to the Original ID. However, this "identity transparency" creates a security gap for protocols relying on **Logic Continuity** or **BCS Determinism**.

**Attack vector:** A proposal is staged to call a function with BCS-serialized arguments. If the package is upgraded between "Staging" and "Execution" phases, the new code might interpret the old BCS bytes differently (e.g., swapped struct fields), leading to unauthorized state transitions or fund theft.

Currently, there is no native way for Move code to answer: *"Which specific version (physical address) of this code is currently running?"*

The existing `std::type_name` functions don't solve this:
- `defining_id<T>()` returns the physical ID **only if the type was first defined in the current version**
- `original_id<T>()` returns the V1 address even when called from V2 or V3 code

For types that existed in V1, neither function can detect which version is executing.

### Who Benefits

- **DAOs/Governance protocols**: Verify third-party module security, pin proposals to audited versions
- **Intent-based systems**: Users signing off-chain intents can pin to specific audited package versions
- **Cross-chain bridges/Oracles**: Ensure verification logic runs against specific immutable bytecode
- **Security auditors**: On-chain verification that interactions use audited code

## Specification

We propose adding the following native functions to `sui::package`:

```move
module sui::package {
    // ... existing functions ...

    // ===== Permissionless Introspection =====

    /// Returns the physical Object ID of the package currently executing.
    ///
    /// If called from a function in package 0xAAA (V1), returns 0xAAA.
    /// If that package is upgraded to 0xBBB (V2), and the same function
    /// is called in the context of V2, it returns 0xBBB.
    public native fun current_package_id(): address;

    /// Returns the upgrade policy for any package, without requiring the UpgradeCap.
    /// 0 = COMPATIBLE (full logic changes allowed)
    /// 128 = ADDITIVE (only new functions/structs allowed)
    /// 192 = DEP_ONLY (only dependency changes allowed)
    /// 255 = IMMUTABLE (no upgrades possible)
    public native fun policy_for(package_id: address): u8;

    /// Returns true if the package is immutable and cannot be upgraded.
    /// Equivalent to: policy_for(package_id) == 255
    public native fun is_immutable(package_id: address): bool;

    /// Returns the current version (upgrade count) of any package.
    /// Returns 1 for initial deployments, 2 after first upgrade, etc.
    public native fun version_for(package_id: address): u64;
}
```

### VM Implementation Details

**For `current_package_id()`:** The Move VM already tracks the `ModuleId` (containing the physical `Address`) of the currently executing function to load correct bytecode. This primitive exposes the `address` portion of the current `ModuleId` from the VM stack.

**For `policy_for()`, `is_immutable()`, `version_for()`:** The Sui network stores this information in Package object metadata within global state. These functions read from the same source as the existing `UpgradeCap`-gated functions, but without requiring the capability.

## Rationale

### Why `sui::package` (not `std::type_name`)?

The concept of "Physical Object ID" and "Package Upgrades" is specific to the Sui blockchain object model. The `std` library is intended to be platform-agnostic, whereas `sui::package` is the appropriate home for blockchain-specific deployment metadata.

Additionally, `sui::package` already contains all upgrade-related types (`UpgradeCap`, `UpgradeTicket`, `UpgradeReceipt`) and functions. These new functions are natural extensions—read-only views of the same metadata.

### Why Not Extend Existing Functions?

The existing functions (`upgrade_policy`, `version`, `upgrade_package`) require `&UpgradeCap` by design—they're part of the upgrade authorization flow. Adding permissionless variants with different names (`policy_for`, `version_for`) maintains backward compatibility and clearly distinguishes "I hold the cap" from "I'm just querying."

### Why Not Use Version Constants?

Developers could manually define `const VERSION: u64 = 1;`. However:
1. Error-prone—developers forget to update it
2. Easily spoofed—a malicious dev could upgrade code but keep the constant unchanged

A native primitive provides **on-chain truth** that cannot be manipulated by the package developer.

### Why Native Functions?

The Move VM's security model prohibits smart contracts from arbitrarily accessing ledger state. Native functions are required to bridge package metadata into the Move execution context.

## Use Cases

### Use Case 1: DAO Action Module Registration

```move
public fun register_action_module(registry: &mut Registry, pkg_id: address) {
    let policy = package::policy_for(pkg_id);
    // Only allow Additive (128) or Immutable (255)
    assert!(policy >= 128, EUnsafeUpgradePolicy);

    vector::push_back(&mut registry.allowed_modules, pkg_id);
}
```

### Use Case 2: Governance Version Pinning

```move
// At Staging Time (in the DAO)
public fun stage_proposal(action: Action, target_module: address) {
    let spec = ActionSpec {
        data: bcs::to_bytes(&action),
        pinned_version: target_module, // Physical package ID at staging time
    };
    // ... store spec
}

// At Execution Time (in the third-party module)
public fun execute_action(spec: ActionSpec) {
    // Ensure the code running NOW is the same code that was audited/staged
    assert!(
        package::current_package_id() == spec.pinned_version,
        EPackageUpgradedSinceStaging
    );

    // Safe to deserialize BCS - struct layout hasn't changed
    let action: Action = bcs::from_bytes(spec.data);
    // ... execute
}
```

### Use Case 3: Security Audit Registry

```move
/// A registry of audited package versions
public fun is_audited(audit_registry: &AuditRegistry, pkg_id: address): bool {
    // Check if this specific package version is in the audited set
    table::contains(&audit_registry.audited_packages, pkg_id)
}

public fun execute_only_if_audited(
    audit_registry: &AuditRegistry,
    /* ... */
) {
    let current = package::current_package_id();
    assert!(is_audited(audit_registry, current), ENotAudited);
    // ... proceed with execution
}
```

## Backwards Compatibility

This proposal is purely additive. It introduces new native functions but does not change the behavior of existing `UpgradeCap` functions or the upgrade flow itself. No existing smart contracts will be affected.

## Test Cases

### Test 1: `current_package_id()` Returns Correct Physical ID

```move
// In package V1 (0xAAA)
#[test]
fun test_current_package_id_v1() {
    assert!(package::current_package_id() == @0xAAA, 0);
}

// After upgrade to V2 (0xBBB), same test returns 0xBBB
```

### Test 2: `policy_for()` Matches `upgrade_policy()`

```move
#[test]
fun test_policy_for_matches_cap(cap: &UpgradeCap) {
    let pkg_id = object::id_to_address(&package::upgrade_package(cap));
    assert!(
        package::policy_for(pkg_id) == package::upgrade_policy(cap),
        0
    );
}
```

### Test 3: `is_immutable()` After `make_immutable()`

```move
#[test]
fun test_is_immutable(cap: UpgradeCap) {
    let pkg_id = object::id_to_address(&package::upgrade_package(&cap));
    assert!(!package::is_immutable(pkg_id), 0);

    package::make_immutable(cap);
    assert!(package::is_immutable(pkg_id), 1);
}
```

### Test 4: `version_for()` Increments on Upgrade

```move
#[test]
fun test_version_increments(/* upgrade scenario */) {
    // After initial publish
    assert!(package::version_for(pkg_id) == 1, 0);
    // After first upgrade
    assert!(package::version_for(new_pkg_id) == 2, 1);
}
```

## Reference Implementation

To be developed by core engineers if the SIP is approved.

The implementation requires:
1. New native function bindings in the Sui runtime
2. Read access to Package object metadata from the Move VM context
3. For `current_package_id()`: exposing the address portion of the current `ModuleId` from the VM call stack

## Security Considerations

### Read-Only Access

These functions are strictly read-only. They do not grant any power to upgrade a package, only to observe its current state. The `UpgradeCap` remains the sole authority for performing upgrades.

### State Consistency

The values returned must reflect the state of the package at the start of the current transaction. Sui's upgrade model prevents "flash-upgradability" tricks because upgrades require their own transaction and cannot occur mid-execution.

### Increased Security Surface

Exposing package metadata actually **increases** security by enabling:
- **Formal Verification**: Prove that execution traces occurred within specific immutable bytecode
- **Audit Enforcement**: Restrict interactions to specific physical addresses marked as "Audited"
- **Trustless Policy Verification**: Remove reliance on off-chain claims about package immutability

### Potential Misuse

Developers might incorrectly implement version checks that "brick" their own contracts. This is a standard risk with any logic-controlling primitive and is the developer's responsibility.

## Copyright

Greshamscode 2025
