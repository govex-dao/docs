# Advanced Sui Design Patterns

A series exploring Move patterns developed for the Govex futarchy protocol.

## The Series

| # | Pattern | Problem Solved | Category |
|---|---------|----------------|----------|
| 1 | [Atomic Intent](./01-atomic-intent.md) | Object Stealing / Parameter Injection | Security |
| 2 | [Balance Wrappers](./02-balance-wrappers.md) | Type Explosion with N outcomes | Scaling |
| 3 | [Iterative Hot Potato](./03-iterative-hot-potato.md) | Generic Loop Constraints | Move Limitations |
| 4 | [Blank Coin Registry](./04-blank-coin-registry.md) | Runtime Coin Type Creation | Infrastructure |
| 5 | [Registry-Validated Witness](./05-registry-validated-witness.md) | Dynamic Package Authorization | Authorization |
| 6 | [Config Migration](./06-config-migration.md) | Zero-Downtime Schema Upgrades | Upgradability |
| 7 | [Resource Request Composition](./07-resource-request-composition.md) | In-Flight Asset Staging | Composition |
| 8 | [Atomic Object Accumulator](./08-atomic-object-accumulator.md) | Constructor Argument Limits | Scalability |

## Why These Patterns?

Building futarchy governance on Sui required solving problems at the intersection of:
- **Security**: Governance actions must be tamper-proof
- **Scalability**: Markets need N outcomes without N type parameters
- **Move Constraints**: Static generics vs. dynamic runtime needs

Each pattern addresses a fundamental limitation in Sui/Move development.

## Source Code

All patterns are implemented in the [Govex repository](https://github.com/govex-dao/govex).
