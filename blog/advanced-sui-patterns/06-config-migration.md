# Config Migration with TypeName Tracking

*Part 6 of the [Advanced Sui Design Patterns](./README.md) series*

---

How do you upgrade a DAO's config schema without breaking existing intents or requiring account recreation? Govex tracks config types at runtime using TypeName, enabling safe migrations.

## The Problem

Move structs are immutable once deployed. If your Account stores `ConfigV1` and you need to add fields, you can't just modify the struct. You need a migration path.

```move
// Version 1 - deployed
public struct ConfigV1 has store {
    threshold: u64,
}

// Version 2 - needs new field
public struct ConfigV2 has store {
    threshold: u64,
    quorum: u64,  // New field!
}
```

The naive approach fails:
```move
// This aborts - stored type is ConfigV1, not ConfigV2
let config: &ConfigV2 = df::borrow(&account.id, ConfigKey {});
```

## The Solution: Dual Dynamic Field Storage

Store both the config data AND its TypeName in separate dynamic fields.

### Storage Structure

```move
public struct ConfigKey has copy, drop, store {}
public struct ConfigTypeKey has copy, drop, store {}

// At account creation:
df::add(&mut account.id, ConfigKey {}, config);
df::add(&mut account.id, ConfigTypeKey {}, type_name::get<Config>());
```

### The Migration Function

```move
public(package) fun migrate_config<OldConfig: store, NewConfig: store>(
    account: &mut Account,
    registry: &PackageRegistry,
    new_config: NewConfig,
    version_witness: VersionWitness,
): OldConfig {
    // 1. Verify caller is authorized
    account.deps().check(version_witness, registry, account.account_deps());

    // 2. Validate OldConfig matches stored type
    let stored_type = df::borrow<ConfigTypeKey, TypeName>(&account.id, ConfigTypeKey {});
    let old_type = type_name::get<OldConfig>();
    assert!(&old_type == stored_type, EWrongConfigType);

    // 3. Atomically swap config
    let old_config: OldConfig = df::remove(&mut account.id, ConfigKey {});
    df::add(&mut account.id, ConfigKey {}, new_config);

    // 4. Update type tracking
    let _: TypeName = df::remove(&mut account.id, ConfigTypeKey {});
    df::add(&mut account.id, ConfigTypeKey {}, type_name::get<NewConfig>());

    // 5. Return old config for validation/destruction
    old_config
}
```

### Usage

```move
// Migration action in a governance proposal
public fun do_migrate_config<OldConfig, NewConfig, Outcome>(
    executable: &mut Executable<Outcome>,
    account: &mut Account,
    registry: &PackageRegistry,
    new_config: NewConfig,
    version: VersionWitness,
) {
    let old_config: OldConfig = account::migrate_config(
        account,
        registry,
        new_config,
        version,
    );

    // Validate old config before dropping
    validate_migration(&old_config);
    // old_config dropped here (must have drop ability or be consumed)
}
```

## Runtime Type Checking

The TypeName field also enables runtime type validation for reads:

```move
/// Get config type for runtime checks
public fun config_type(account: &Account): TypeName {
    *df::borrow<ConfigTypeKey, TypeName>(&account.id, ConfigTypeKey {})
}

// Usage: verify before borrowing
let expected = type_name::get<FutarchyConfigV2>();
assert!(account::config_type(account) == expected, EWrongVersion);
let config: &FutarchyConfigV2 = account::config(account);
```

## Why This Matters

- **Zero-downtime upgrades**: Migrate config without account recreation
- **Active intent safety**: Intents in-flight are unaffected by config changes
- **Type-safe migrations**: Compiler catches mismatched old/new types
- **Audit trail**: Old config returned for validation before destruction
- **Runtime introspection**: Check config version without knowing the type

## Migration Best Practices

1. **Return old config**: Let caller validate the migration
2. **Atomic swap**: Remove old, add new in same function
3. **Version witness**: Ensure authorized packages only
4. **Type tracking**: Always update ConfigTypeKey

## Source Code

- [account.move](https://github.com/govex-dao/govex/blob/main/packages/move-framework/packages/protocol/sources/account.move)

---

*Next: [Resource Request Composition](./07-resource-request-composition.md)*
