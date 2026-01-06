# The Iterative Hot Potato: PTB Loop Unrolling

*Part 3 of the [Advanced Sui Design Patterns](./README.md) series*

---

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

This pattern allows **Dynamic Runtime Logic** (handling $N$ outcomes) to coexist with **Static Compile-Time Safety** (generic type parameters).

- **Enforces completeness**: Can't skip an outcome
- **Enforces order**: Must process sequentially
- **Atomic**: Hot potato ensures same-transaction execution
- **Type-safe**: Each step is fully typed

It is the only way to maintain a 1:N "Quantum Invariant" across an arbitrary number of conditional markets without hitting Move's type-parameter limits.

## Source Code

- [split_operations.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_operations/sources/split_operations.move)
- [merge_operations.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy/futarchy_markets_operations/sources/merge_operations.move)

---

*Next: [Dynamic Coin Acquisition: The Blank Coin Registry](./04-blank-coin-registry.md)*
