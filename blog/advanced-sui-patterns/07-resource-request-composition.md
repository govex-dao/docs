# Resource Request Composition: In-Flight Asset Staging

*Part 7 of the [Advanced Sui Design Patterns](./README.md) series*

---

Complex governance actions often need assets to flow between multiple operations within a single transaction. Govex uses Resource Requests to stage assets in-flight without intermediate vault deposits.

## The Problem

Consider a liquidity rebalance: remove LP from Pool A, add to Pool B. The naive approach:

```move
// Step 1: Remove liquidity → coins go to vault
vault::deposit(vault, asset_coin);
vault::deposit(vault, stable_coin);

// Step 2: Withdraw from vault → add to new pool
let asset = vault::withdraw(vault, ...);
let stable = vault::withdraw(vault, ...);
pool_b::add_liquidity(asset, stable);
```

This works but:
- Extra gas for vault round-trips
- Requires vault permissions for intermediate state
- Complex to compose in a PTB

## The Solution: Resource Requests

Create a hot potato that stages assets in-flight between actions.

### The Request Structure

```move
/// Hot potato - MUST be fulfilled in same transaction
#[allow(lint(missing_key))]
public struct ResourceRequest<phantom T> {
    id: UID,
    context: UID,  // Dynamic fields for flexible data
}

/// Completion marker
public struct ResourceReceipt<phantom T> has drop {
    request_id: ID,
}
```

### The Flow: Request → Fulfill

**Step 1: Action returns a request**

```move
public fun do_remove_liquidity_to_resources<AssetType, StableType, LPType, Outcome>(
    executable: &mut Executable<Outcome>,
    ...
): ResourceRequest<RemoveLiquidityToResourcesAction<AssetType, StableType, LPType>> {
    // Validate action, parse params...
    let action = RemoveLiquidityToResourcesAction { ... };

    // Return hot potato with action stored inside
    resource_requests::new_resource_request(action, ctx)
}
```

**Step 2: Fulfillment provides resources to executable**

```move
public fun fulfill_remove_liquidity_to_resources<...>(
    request: ResourceRequest<RemoveLiquidityToResourcesAction<...>>,
    executable: &mut Executable<Outcome>,
    spot_pool: &mut UnifiedSpotPool<...>,
    ...
): ResourceReceipt<RemoveLiquidityToResourcesAction<...>> {
    // Extract action from request
    let action = resource_requests::extract_action(request);

    // Take LP coin from prior action's output
    let lp_coin: Coin<LPType> = executable_resources::take_coin(
        executable::uid_mut(executable),
        action.lp_resource_name,
    );

    // Execute: remove liquidity
    let (asset_coin, stable_coin) = unified_spot_pool::remove_liquidity(
        spot_pool, lp_coin, ...
    );

    // Stage outputs for NEXT action
    executable_resources::provide_coin(
        executable::uid_mut(executable),
        action.asset_output_name,  // "rebalance_asset"
        asset_coin,
        ctx,
    );
    executable_resources::provide_coin(
        executable::uid_mut(executable),
        action.stable_output_name,  // "rebalance_stable"
        stable_coin,
        ctx,
    );

    // Return receipt (drops action)
    resource_requests::create_receipt(action)
}
```

**Step 3: Next action consumes staged resources**

```move
public fun fulfill_add_liquidity<...>(
    request: ResourceRequest<AddLiquidityAction<...>>,
    executable: &mut Executable<Outcome>,
    ...
): ResourceReceipt<AddLiquidityAction<...>> {
    let action = resource_requests::extract_action(request);

    // Take coins staged by prior action
    let asset_coin: Coin<AssetType> = executable_resources::take_coin(
        executable::uid_mut(executable),
        action.asset_resource_name,  // "rebalance_asset"
    );
    let stable_coin: Coin<StableType> = executable_resources::take_coin(
        executable::uid_mut(executable),
        action.stable_resource_name,  // "rebalance_stable"
    );

    // Add to new pool...
}
```

## The PTB Structure

```typescript
const ptb = new TransactionBlock();

// Action 1: Request removal
const removeRequest = ptb.moveCall({
    target: 'pkg::liquidity_actions::do_remove_liquidity_to_resources',
    arguments: [executable, ...],
});

// Fulfill: execute removal, stage coins
const removeReceipt = ptb.moveCall({
    target: 'pkg::liquidity_actions::fulfill_remove_liquidity_to_resources',
    arguments: [removeRequest, executable, poolA, ...],
});

// Action 2: Request add
const addRequest = ptb.moveCall({
    target: 'pkg::liquidity_actions::do_add_liquidity',
    arguments: [executable, ...],
});

// Fulfill: consume staged coins, add to pool
const addReceipt = ptb.moveCall({
    target: 'pkg::liquidity_actions::fulfill_add_liquidity',
    arguments: [addRequest, executable, poolB, ...],
});
```

## Why This Matters

- **No vault round-trips**: Assets flow directly between actions
- **Composable**: Chain unlimited actions with resource names
- **Type-safe**: Phantom types ensure correct pairing
- **Flexible**: Dynamic fields in context allow any data

**Note:** This pattern applies to any resource flow between PTB steps—owned objects, coins, gaming items (Ore → Bar → Sword), flash loans. The core insight: use the hot potato's dynamic fields as a temporary carrier bag.

## vs. Executable Resources Alone

| Pattern | Use Case |
|---------|----------|
| Executable Resources | Simple staging within one action |
| Resource Requests | Multi-action composition with deferred execution |

Resource Requests add the hot potato layer for two-phase execution (request → fulfill), which is useful when the action definition and resource provision are separate steps.

## Source Code

- [resource_requests.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy_core/sources/resource_requests.move)
- [liquidity_actions.move](https://github.com/govex-dao/govex/blob/main/packages/futarchy_actions/sources/liquidity/liquidity_actions.move)

---

*Next: [Atomic Object Accumulator](./08-atomic-object-accumulator.md)*
