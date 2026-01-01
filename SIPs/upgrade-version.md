
This is a formal draft for a Sui Improvement Proposal (SIP). You can use this text to open a discussion on the [Sui Forums](https://forums.sui.io/) or submit it to the [Sui SIP repository](https://github.com/sui-foundation/sips).

---

# SIP: Exposing Runtime Physical Package ID

**Status**: Draft  
**Author**: [Your Name/Organization]  
**Category**: Framework/Move  
**Created**: 2024-05-20  

## Abstract

This SIP proposes adding a native function, `sui::package::current_package_id(): address`, to the Sui Framework. This function will return the **physical Object ID** of the package currently executing. Unlike existing `type_name` functions which return the "Original ID" to maintain type continuity, this primitive allows Move code to distinguish between different versions (upgrades) of the same package at runtime.

## Motivation

Sui’s upgrade model is designed for seamless data continuity. When a package is upgraded from V1 to V2, objects created in V1 remain compatible with V2 because their `TypeName` remains anchored to the Original ID.

However, this "identity transparency" creates a security gap for protocols that rely on **Logic Continuity** or **BCS Determinism**, specifically:

1.  **Governance/DAO Protocols**: A proposal might be staged to call a specific function with BCS-serialized arguments. If the package is upgraded between the "Staging" and "Execution" phases, the new code might interpret the old BCS bytes differently (e.g., swapped fields), leading to unauthorized state transitions or fund theft.
2.  **Intent-Based Systems**: Users signing off-chain intents for on-chain execution need to "pin" their intent to a specific audited version of a package.
3.  **Cross-Chain/Oracles**: Verification logic that must ensure it is running a specific immutable logic set rather than a potentially malicious upgrade.

Currently, there is no native way for a Move contract to programmatically answer the question: *"Which specific version (physical address) of my code is currently running?"*

## Specification

We propose adding the following native function to the `sui::package` module:

```move
module sui::package {
    /// Returns the physical Object ID of the package currently executing.
    /// 
    /// If called from a function in package 0xAAA (V1), returns 0xAAA.
    /// If that package is upgraded to 0xBBB (V2), and the same function 
    /// is called in the context of V2, it returns 0xBBB.
    public native fun current_package_id(): address;
}
```

### VM Implementation Details
The Move VM already tracks the `ModuleId` (which contains the physical `Address`) of the currently executing function to load the correct bytecode. This primitive simply exposes the `address` portion of the current `ModuleId` from the VM stack to the Move environment.

## Use Case Example: Governance Pinning

With this primitive, a DAO can prevent "Upgrade Attacks":

```move
// At Staging Time
public fun stage_proposal(action: Action, action_version: address) {
    // action_version is fetched via a call to the third-party module
    let spec = ActionSpec { 
        data: bcs::to_bytes(&action),
        pinned_version: action_version 
    };
    // ... store spec
}

// At Execution Time (In the third-party module)
public fun execute_action(spec: ActionSpec) {
    // Ensure the code running NOW is the same code that was audited/staged
    assert!(sui::package::current_package_id() == spec.pinned_version, EWrongVersion);
    
    // Proceed to deserialize BCS safely
    let action = bcs::new(spec.data);
    // ...
}
```

## Rationale

### Why not use `type_name::get_defining_id<T>()`?
As of current Sui Move implementation, `get_defining_id` returns the physical ID **only if the type was first defined in the current version**. For any type that existed in the original V1, `get_defining_id` returns the V1 address even when called from V2 or V3 code. This makes it useless for version detection of legacy types.

### Why not use Version Constants?
Developers can manually define a `const VERSION: u64 = 1;`. However:
1.  It is error-prone (developers forget to update it).
2.  It is easily spoofed (a malicious dev could upgrade code layout but keep the version constant the same to trick a DAO).
A native primitive provides **On-Chain Truth** that cannot be manipulated by the package developer.

### Why `sui::package` and not `std::type_name`?
The concept of a "Physical Object ID" and "Package Upgrades" is specific to the Sui blockchain object model. The `std` library is intended to be platform-agnostic, whereas `sui::package` is the appropriate home for blockchain-specific deployment metadata.

## Backward Compatibility
This is a purely additive change. It does not modify existing `TypeName` behavior or break existing contracts.

## Security Considerations

Exposing the package ID actually **increases** security by enabling:
- **Formal Verification**: Ability to prove that a specific execution trace occurred within a specific immutable bytecode set.
- **Audit Enforcement**: Users can restrict their interactions to specific physical addresses that have been marked as "Audited" by a trusted registry.

One minor consideration is that developers might use this to "brick" their own contracts if they incorrectly implement version checks, but this is a standard risk with any logic-controlling primitive.