# Preventing Object Stealing in Sui PTBs: The Atomic Intent Pattern

*Part 1 of the [Advanced Sui Design Patterns](./README.md) series*

---

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

## Source Code

- [executable.move](https://github.com/govex-dao/govex/blob/main/packages/move-framework/packages/protocol/sources/executable.move)
- [executable_resources.move](https://github.com/govex-dao/govex/blob/main/packages/move-framework/packages/protocol/sources/executable_resources.move)

---

*Next: [Balance Wrappers: Scalable Type Erasure](./02-balance-wrappers.md)*
