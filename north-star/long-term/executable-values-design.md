# Executable Values: Runtime Computation for Governance Actions

## The Problem

Current governance actions use **fixed amounts** specified at proposal creation time:

```
Day 0 (Proposal Created):        Day 7 (Execution):
─────────────────────────        ──────────────────────
VaultSpend(500 ASSET)      →     Pool ratio changed from 2:1 to 3:1
VaultSpend(500 STABLE)     →     Now need 1500 ASSET for 500 STABLE
AddLiquidity(500, 500)     →     Transaction fails or wastes funds
```

We need **dynamic amounts** computed at execution time based on current on-chain state.

---

## Two Approaches

### Approach A: Primitive Operations (The Verbose Way)

Build a register-based computation system where each operation is a separate action:

```
1.  SetLiteralU64("min_reserve", 1000)
2.  VaultSpend<LP>(all, "lp_to_remove")
3.  RemoveLiquidity("lp_to_remove")
4.  ReadVaultBalance<Asset>("vault_asset")
5.  ReadVaultBalance<Stable>("vault_stable")
6.  ComputeSub("vault_asset", "min_reserve", "available_asset")
7.  ComputeSub("vault_stable", "min_reserve", "available_stable")
8.  ReadPoolRatio(PoolB, "pool_b_ratio")
9.  ComputeApplyRatio("available_stable", "pool_b_ratio", "target_asset")
10. ComputeMin("available_asset", "target_asset", "actual_asset")
11. ComputeInvertRatio("pool_b_ratio", "inv_ratio")
12. ComputeApplyRatio("actual_asset", "inv_ratio", "actual_stable")
13. AssertNonZero("actual_asset")
14. AssertNonZero("actual_stable")
15. VaultSpendDynamic<Asset>("actual_asset", "asset_for_lp")
16. VaultSpendDynamic<Stable>("actual_stable", "stable_for_lp")
17. AddLiquidity("asset_for_lp", "stable_for_lp", 0)
```

**17 actions** to migrate liquidity at the current ratio.

### Approach B: Domain-Specific Composite Actions (The Practical Way)

Build high-level actions that encapsulate common patterns:

```
1. RemoveAllLiquidity(PoolA)
2. AddLiquidityAtCurrentRatio(PoolB, min_remaining_asset=1000, min_remaining_stable=1000)
```

**2 actions** for the same result.

---

## Why Is Approach A So Verbose?

### 1. Move's Constraints

Move has no closures, no dynamic dispatch, limited generics. You can't write:

```move
// This doesn't exist in Move
let amount = compute(|vault| {
    let balance = vault.balance<Asset>();
    let ratio = pool.get_ratio();
    min(balance - 1000, available_stable * ratio)
});
```

### 2. Security Model

Each action must be independently validated with a type marker:

```move
action_validation::assert_action_type<ComputeMin>(spec);
```

This prevents type confusion attacks but means every operation needs its own action type.

### 3. BCS Serialization

Actions are serialized as bytes. No way to encode "expressions" - just flat data:

```
ComputeMin: [input_a_name, input_b_name, output_name]
```

### 4. Existing Architecture

The system was designed around "one action = one state change". Computation wasn't a first-class concern.

---

## Honest Assessment

### Approach A (Primitives) Is:

**Genius because:**
- Maximum flexibility - can express any computation
- Fully auditable - every step is visible on-chain
- Composable - primitives combine in arbitrary ways
- Future-proof - new operations don't require new action types

**Overcooked because:**
- 17 actions for one logical operation is absurd
- Proposal creation UX becomes a nightmare
- Gas costs scale with action count
- Error messages are cryptic ("ComputeMin failed at action 10")
- Cognitive overhead is massive

### Approach B (Composites) Is:

**Practical because:**
- 2 actions is reasonable
- Clear intent ("add liquidity at ratio")
- Better error messages
- Lower gas

**Limited because:**
- Each pattern needs a new action type
- Can't handle edge cases without new code
- Combinatorial explosion of patterns

---

## The Real Question

**Do you need general-purpose computation, or do you need specific patterns?**

### If Specific Patterns (90% of cases):

Build composite actions for your actual use cases:

| Pattern | Composite Action |
|---------|-----------------|
| Migrate liquidity | `MigrateLiquidityAtRatio(from_pool, to_pool, min_remaining)` |
| Rebalance treasury | `RebalanceToRatio(asset_type, stable_type, target_ratio, tolerance)` |
| DCA purchase | `SwapUpToAmount(max_spend, min_receive, pool)` |
| Proportional distribution | `DistributeProportional(recipients[], total_amount)` |

### If General-Purpose (10% of cases):

Build the primitive system, but **hide it behind a DSL compiler**:

```typescript
// User writes this (in SDK/frontend):
const actions = compile(`
  remove_liquidity(pool_a, all)
  add_liquidity_at_ratio(pool_b,
    leaving: { asset: 1000, stable: 1000 }
  )
`);

// Compiler generates the 17 primitive actions
```

---

## Recommended Path Forward

### Phase 1: Composite Actions (Now)

Build 3-5 composite actions for your immediate needs:

```move
/// Remove liquidity and add to another pool at current ratio
public struct MigrateLiquidity<A, S, LP_From, LP_To> has drop {}
struct MigrateLiquidityAction {
    from_pool: ID,
    to_pool: ID,
    lp_amount: u64,           // or 0 for all
    min_asset_remaining: u64, // leave in vault
    min_stable_remaining: u64,
    slippage_bps: u64,        // max acceptable slippage
}

/// Spend from vault at pool's current ratio
public struct SpendAtPoolRatio<A, S, LP> has drop {}
struct SpendAtPoolRatioAction {
    pool_id: ID,
    max_asset: u64,
    max_stable: u64,
    min_asset_remaining: u64,
    min_stable_remaining: u64,
    asset_output: String,
    stable_output: String,
}
```

### Phase 2: Evaluate (After 6 months)

- How many composite actions did you need?
- What patterns couldn't you express?
- Is there demand for general computation?

### Phase 3: Primitives (If Needed)

Only build the full `executable_values` system if:
1. You're constantly adding new composite actions
2. Users are requesting custom computation
3. You're willing to build the compiler/UX layer

---

## Alternative: Expression-in-ActionSpec

Instead of separate actions for each operation, encode expressions in the action data:

```move
struct SpendDynamicAction<CoinType> {
    vault_name: String,
    // Instead of a fixed amount or value reference,
    // embed a mini-expression
    amount_expr: vector<u8>, // Serialized expression tree
}

// Expression opcodes
const OP_LITERAL: u8 = 0;      // [value: u64]
const OP_VAULT_BALANCE: u8 = 1; // [vault_name, coin_type]
const OP_POOL_RATIO: u8 = 2;    // [pool_id, which_side]
const OP_ADD: u8 = 10;
const OP_SUB: u8 = 11;
const OP_MUL: u8 = 12;
const OP_DIV: u8 = 13;
const OP_MIN: u8 = 14;
const OP_MAX: u8 = 15;
```

Then ONE action can compute complex amounts:

```
SpendDynamic<Asset> {
    vault: "treasury",
    amount_expr: MIN(
        SUB(VAULT_BALANCE("treasury", Asset), LITERAL(1000)),
        MUL(VAULT_BALANCE("treasury", Stable), POOL_RATIO(pool_b))
    )
}
```

**Pros:** Single action, still flexible
**Cons:** Complex expression evaluator needed, harder to audit

---

## Conclusion

The primitive approach isn't wrong - it's the **theoretically correct** solution for maximum flexibility. But theory and practice diverge:

| Dimension | Primitives | Composites |
|-----------|-----------|------------|
| Flexibility | Unlimited | Limited to patterns |
| Verbosity | 17+ actions | 2-3 actions |
| UX | Nightmare | Manageable |
| Auditability | Every step visible | Intent clear |
| Gas | High | Low |
| Implementation | Complex | Straightforward |

**Recommendation:** Start with composites. The primitive system is your escape hatch if composites prove insufficient, not your starting point.

---

## Appendix: Primitive System Design (For Reference)

If you do need primitives, the design should be:

### Storage
```move
module account_protocol::executable_values;

// Named u64 values
public fun set_u64(uid: &mut UID, name: String, value: u64);
public fun get_u64(uid: &UID, name: String): u64;

// Named ratios (num, denom)
public fun set_ratio(uid: &mut UID, name: String, num: u64, denom: u64);
public fun get_ratio(uid: &UID, name: String): (u64, u64);

// Safe math
public fun safe_add(a: u64, b: u64): u64;
public fun safe_sub(a: u64, b: u64): u64; // saturating
public fun apply_ratio(value: u64, num: u64, denom: u64): u64;
public fun min(a: u64, b: u64): u64;
public fun max(a: u64, b: u64): u64;
```

### Action Types
```
Literal:     SetLiteralU64, SetLiteralRatio
Read:        ReadVaultBalance<C>, ReadPoolRatio<A,S,LP>
Math:        ComputeAdd, ComputeSub, ComputeMin, ComputeMax, ComputeApplyRatio
Assert:      AssertGte, AssertNonZero
Dynamic:     VaultSpendDynamic<C>
```

### Execution Model
1. Actions execute in order (action_idx)
2. Values are stored in `executable_values` bag on Executable UID
3. Dynamic actions read from values, write to resources
4. Values don't need to be consumed (unlike resources)
