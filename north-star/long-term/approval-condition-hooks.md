# Approval Condition Hooks

Actions check approval conditions at execution time based on protocol state.

## Core Idea

```move
public fun do_spend<...>(...) {
    // 1. Deserialize action
    let (vault_name, amount, ...) = ...;

    // 2. CHECK APPROVAL CONDITIONS
    let policy = get_vault_policy(vault_name);
    let spend_pct = (amount * 100) / vault_balance;

    if (spend_pct > policy.twap_threshold_pct) {
        assert!(outcome_twap > policy.required_twap, ENeedsHigherConfidence);
    }
    if (spend_pct > policy.multisig_threshold_pct) {
        assert!(has_multisig_approval(executable), ENeedsMultisig);
    }

    // 3. Execute
}
```

## Use Cases

- **Runway vault**: Block if remaining < 2-3 months burn
- **Large spends**: Require higher TWAP threshold or multisig co-sign
- **Treasury moves**: >X% needs core team approval

## Policy Location Options

1. **Per-vault** - `VaultPolicy` attached to each vault
2. **Per-action-type** - Different rules for Spend vs Swap vs Stream
3. **Composable** - Mix-and-match condition objects

## Relation to Lifecycle Hooks

This is pre-execution validation from the hooks vision - but hooks can read live protocol state (TWAP, balances, other vaults) to make conditional decisions.
