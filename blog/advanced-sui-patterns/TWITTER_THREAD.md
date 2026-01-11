# Advanced Sui Design Patterns - Twitter Thread

---

**Tweet 1 (Hook)**

8 design patterns we invented building futarchy on Sui.

40k lines of Move. 71 governance actions. Zero hacks.

Here's what we learned solving problems at the edge of what Move can do:

---

**Tweet 2 (Pattern 1)**

1/ THE ATOMIC INTENT PATTERN

Problem: PTB executors can steal objects mid-transaction or inject fake parameters.

Solution: BCS-serialize params into immutable storage. Use a hot potato to force atomic completion. Never return objects - stage them in a Bag.

No parameter injection. No object stealing.

---

**Tweet 3 (Pattern 2)**

2/ BALANCE WRAPPERS

Problem: Pool<T0, T1, T2...T10> = type explosion. Every function needs N type params.

Solution: Store balances as vector<u64>. "Re-hydrate" to typed Coin<T> only at tx boundaries using TreasuryCaps in dynamic fields.

Scales to 200+ outcomes. 2 phantom types max.

---

**Tweet 4 (Pattern 3)**

3/ ITERATIVE HOT POTATO

Problem: Move can't loop over dynamic types. No `for i in 0..n { mint<T[i]>() }`.

Solution: Unroll the loop into the PTB. Use a progress counter hot potato to enforce sequential processing of each type.

The only way to maintain N-outcome invariants in Move.

---

**Tweet 5 (Pattern 4)**

4/ BLANK COINS REGISTRY

Problem: Can't create coin types at runtime. Can't use types in the same PTB they're published.

Solution: Pre-create "blank" coins with zero supply. Store in a permissionless registry bucketed by decimals. Acquire and use in same tx.

Runtime type acquisition, solved.

---

**Tweet 6 (Pattern 5)**

5/ REGISTRY-VALIDATED WITNESS

Problem: Witness pattern is compile-time. Can't add authorized packages without upgrades.

Solution: Extract package address from witness via type_name introspection. Verify against a mutable registry.

Dynamic authorization without contract upgrades.

---

**Tweet 7 (Pattern 6)**

6/ CONFIG MIGRATION

Problem: Stored ConfigV1, need ConfigV2 with new fields. Can't just modify the struct.

Solution: Store config AND its TypeName in separate dynamic fields. Atomic swap with runtime type validation.

Zero-downtime schema upgrades.

---

**Tweet 8 (Pattern 7)**

7/ RESOURCE REQUEST COMPOSITION

Problem: Multi-step actions need assets to flow between operations without vault round-trips.

Solution: Hot potato with dynamic fields as a "carrier bag". Stage outputs for the next action. Request -> Fulfill pattern.

Chain unlimited actions. Zero intermediate storage.

---

**Tweet 9 (Pattern 8)**

8/ ATOMIC OBJECT ACCUMULATOR

Problem: Constructor needs 40+ heterogeneous objects. Move has argument limits. Vectors must be homogeneous.

Solution: BEGIN (empty Bags) -> ACCUMULATE (N calls) -> FINALIZE (validate & share). All in one PTB.

Half-initialized objects never hit chain.

---

**Tweet 10 (CTA)**

Full writeup with code examples:
https://greshamscode.substack.com/p/advanced-sui-design-patterns

All patterns are production code in the Govex repo:
https://github.com/govex-dao/govex

Building futarchy-as-a-service. Organizations make decisions through prediction markets instead of voting.

---

## Alt: Compressed Version (fewer tweets)

**Tweet 1**

8 Move patterns we invented building 40k LOC of futarchy governance on Sui:

1. Atomic Intent - prevent PTB object stealing
2. Balance Wrappers - type erasure for N outcomes
3. Iterative Hot Potato - PTB loop unrolling
4. Blank Coins - runtime type acquisition

(thread)

**Tweet 2**

5. Registry-Validated Witness - dynamic package auth
6. Config Migration - zero-downtime schema upgrades
7. Resource Request Composition - in-flight asset staging
8. Atomic Object Accumulator - heterogeneous construction

Each solves a fundamental Move limitation.

**Tweet 3**

Full writeup with code:
https://greshamscode.substack.com/p/advanced-sui-design-patterns

Source code:
https://github.com/govex-dao/govex

Building futarchy-as-a-service on Sui. Ship date: Jan 31.
