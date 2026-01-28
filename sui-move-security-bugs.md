# Sui Move Security: Common Bug Classes and Vulnerability Patterns

A comprehensive guide to security vulnerabilities in Sui Move smart contracts, compiled from audit reports by OtterSec, MoveBit, Zellic, SlowMist, and other leading security firms.

---

## Table of Contents

1. [Object Relationship Validation](#1-object-relationship-validation)
2. [Ability System Misconfigurations](#2-ability-system-misconfigurations)
3. [Access Control Vulnerabilities](#3-access-control-vulnerabilities)
4. [Generic Type Parameter Issues](#4-generic-type-parameter-issues)
5. [Arithmetic and Precision Errors](#5-arithmetic-and-precision-errors)
6. [Hot Potato Pattern Violations](#6-hot-potato-pattern-violations)
7. [Shared Object Security](#7-shared-object-security)
8. [Denial of Service (DoS)](#8-denial-of-service-dos)
9. [Return Value Handling](#9-return-value-handling)
10. [Clock and Timestamp Issues](#10-clock-and-timestamp-issues)
11. [Token and Coin Management](#11-token-and-coin-management)
12. [Package Upgrade Hazards](#12-package-upgrade-hazards)
13. [External Dependency Risks](#13-external-dependency-risks)
14. [Oracle Manipulation](#14-oracle-manipulation)
15. [Witness Pattern Violations](#15-witness-pattern-violations)
16. [UID and Object Identity Issues](#16-uid-and-object-identity-issues)

---

## 1. Object Relationship Validation

**Severity: Critical**

When multiple shared objects have dependencies or relationships, failing to validate these relationships allows cross-protocol attacks.

### Vulnerable Pattern

```move
// VULNERABLE: No validation that whitelist belongs to this launchpad
public entry fun invest(
    launchpad: &mut Launchpad,
    whitelist: &Whitelist,  // Could be from a different launchpad!
    payment: Coin<SUI>,
    ctx: &mut TxContext
) {
    // Attacker uses whitelist from LaunchpadA to invest in LaunchpadB
}
```

### Attack Vector

Users from one launchpad's whitelist can participate in a completely different launchpad, bypassing intended access restrictions.

### Fix

```move
public entry fun invest(
    launchpad: &mut Launchpad,
    whitelist: &Whitelist,
    payment: Coin<SUI>,
    ctx: &mut TxContext
) {
    // ALWAYS validate relationships between objects
    assert!(whitelist.launchpad_id == object::id(launchpad), EWhitelistMismatch);
    // ... rest of function
}
```

### Audit Checklist
- [ ] Every function accepting multiple related objects validates their relationships
- [ ] ID fields are compared using `object::id()` or stored ID references
- [ ] Parent-child relationships are enforced at the contract level

**Source:** [MoveBit - Sui Objects Security](https://www.movebit.xyz/blog/post/Sui-Objects-Security-Principles-and-Best-Practices.html)

---

## 2. Ability System Misconfigurations

**Severity: Critical to High**

Move's ability system (`copy`, `drop`, `store`, `key`) is a core security mechanism. Misconfigurations can completely break security models.

### 2a. Accidental Drop on Hot Potato

**The most severe ability mistake.**

```move
// CRITICAL VULNERABILITY
struct FlashLoanReceipt has drop {  // This destroys your entire security model
    pool_id: ID,
    amount: u64,
}
```

**Impact:** Attackers can take flash loans and simply let the receipt drop, draining the entire pool.

**Fix:** Hot potato structs must have **zero abilities**:

```move
struct FlashLoanReceipt {  // No abilities = must be consumed
    pool_id: ID,
    amount: u64,
}
```

### 2b. Token with Copy and Drop

```move
// CRITICAL: Enables infinite minting and untracked destruction
struct TokenCoin has copy, drop, store {
    amount: u64,
}
```

**Attack:** Attacker copies tokens infinitely, then drops unwanted ones.

**Golden Rule:** Assets should only have `key` and `store`. Never `copy` or `drop`.

### 2c. Missing Phantom Type Parameter

```move
// VULNERABLE: No phantom type - any token type accepted
struct PaymentReceipt {
    amount: u64,
}

// Attacker creates worthless token, gets receipt, claims real assets
```

**Fix:**

```move
struct PaymentReceipt<phantom CoinType> {
    amount: u64,
}
```

### 2d. UID Field Confusion

Objects with `key` ability cannot have `drop` because UIDs deliberately lack it.

```move
// COMPILE ERROR: UID doesn't have drop
struct MyObject has key, drop {
    id: UID,
    data: u64,
}
```

### Ability Combinations Reference

| Struct Type | Abilities | Notes |
|------------|-----------|-------|
| Asset/Token | `key, store` | Never copy/drop |
| Hot Potato | (none) | Must be consumed |
| Capability | `key` only | Non-transferable (soulbound) |
| Transferable Cap | `key, store` | Can be transferred |
| Witness | `drop` only | One-time use pattern |

**Source:** [Mirage Audits - Ability Security Mistakes](https://www.mirageaudits.com/blog/sui-move-ability-security-mistakes)

---

## 3. Access Control Vulnerabilities

**Severity: Critical to High**

### 3a. AdminCap Shared Object Disaster

```move
fun init(ctx: &mut TxContext) {
    let admin_cap = AdminCap { id: object::new(ctx) };
    // CATASTROPHIC: Everyone is now admin
    transfer::share_object(admin_cap);
}
```

**Fix:**

```move
fun init(ctx: &mut TxContext) {
    let admin_cap = AdminCap { id: object::new(ctx) };
    // Transfer to deployer only
    transfer::transfer(admin_cap, tx_context::sender(ctx));
}
```

### 3b. Entry Function Visibility Confusion

**One of the most dangerous misconceptions in Sui Move.**

```move
// VULNERABLE: Developers think this is internal-only
public(package) entry fun privileged_action(pool: &mut Pool) {
    // WRONG! Any external user can call this directly
}
```

**Reality:** The `entry` modifier makes it directly callable as a transaction entry point by **anyone**, regardless of `public(package)` visibility.

**Fix:** Remove `entry` from internal functions, or add explicit capability checks:

```move
public(package) entry fun privileged_action(
    _admin: &AdminCap,  // Require capability
    pool: &mut Pool
) {
    // Now properly protected
}
```

### 3c. Missing Signer Validation

```move
// VULNERABLE: Accepts signer but doesn't verify ownership
public entry fun cancel_order(
    user: &signer,
    order: &mut Order,
) {
    // Missing: assert!(order.owner == signer::address_of(user), ENotOwner);
    transfer_assets_back(order);
}
```

### 3d. Over-Privileged Roles

Audit focus: If roles have permissions that affect user assets, there's a risk of over-permission.

**Source:** [SlowMist - Sui Move Auditing Primer](https://github.com/slowmist/Sui-MOVE-Smart-Contract-Auditing-Primer)

---

## 4. Generic Type Parameter Issues

**Severity: High**

### 4a. Unvalidated Generic Coin Types

```move
// VULNERABLE: No validation of COIN type
public entry fun buy_pack<COIN>(
    game: &mut Game,
    paid: Coin<COIN>,  // Attacker uses worthless custom token
    ctx: &mut TxContext
) {
    // Should validate COIN matches expected type
}
```

**Fix:**

```move
public entry fun buy_pack<COIN>(
    game: &mut Game,
    paid: Coin<COIN>,
    ctx: &mut TxContext
) {
    let coin_type = type_name::get<COIN>();
    assert!(coin_type == type_name::get<USDC>(), EInvalidCoinType);
    // ...
}
```

### 4b. Generic Role Bypass

```move
// VULNERABLE: Accepts ANY RoleCap, not just admin roles
public fun admin_action<R>(cap: &RoleCap<R>, pool: &mut Pool) {
    // Attacker creates RoleCap<FakeRole>
}
```

**Fix:** Use phantom type constraints or explicit role validation.

**Source:** [Zellic - Top 10 Aptos Move Bugs](https://www.zellic.io/blog/top-10-aptos-move-bugs/)

---

## 5. Arithmetic and Precision Errors

**Severity: Critical to Medium**

### 5a. Bitwise Operation Overflow (The Cetus Bug - $223M)

**Move aborts on overflow for `+`, `*`, `-` but NOT for bitwise operations like `<<`.**

```move
// VULNERABLE: checked_shlw validated wrong bit limit (256 vs 192)
// Allowed overflow during liquidity calculations
let shifted = value << bits;  // Does NOT abort on overflow!
```

**The Attack:**
1. Attacker selects liquidity value that passes overflow check
2. But creates overflow during actual calculation
3. Deposits 1 token, gets billions in liquidity
4. Drains pool

**Lesson:** Audit all bit shift operations manually. Move's safety doesn't catch these.

### 5b. Division Precision Loss

```move
// VULNERABLE: Fee rounds to zero for small orders
let fee = size * PROTOCOL_FEE_BPS / 10000;  // Rounds down!
```

**Fix:** Require minimum order sizes or ensure calculated fees are always nonzero.

### 5c. Multiplication Overflow

```move
// VULNERABLE: Large order sizes cause overflow
let result = large_value * another_large_value;  // Aborts on overflow
```

**Fix:** Cast to `u128` before multiplication, or use safe math libraries.

### 5d. Time Unit Confusion

```move
// VULNERABLE: Milliseconds stored in "seconds" field
let now_seconds = clock::timestamp_ms(clock);  // Returns milliseconds!
// Comparing seconds against milliseconds
if (now_seconds >= position.seconds + STAKE_LOCK_TIME_SECONDS) { ... }
```

**Source:** [Dedaub - Cetus Hack Analysis](https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/)

---

## 6. Hot Potato Pattern Violations

**Severity: Critical**

The hot potato pattern enforces that certain operations must be completed atomically within a transaction.

### Correct Implementation

```move
struct Receipt {  // NO abilities = must be consumed
    pool_id: ID,
    amount: u64,
}

public fun borrow(pool: &mut Pool, amount: u64): (Coin<SUI>, Receipt) {
    // ... issue loan
}

public fun repay(pool: &mut Pool, coin: Coin<SUI>, receipt: Receipt) {
    let Receipt { pool_id, amount } = receipt;  // Consume receipt
    assert!(pool_id == object::id(pool), EPoolMismatch);
    assert!(coin::value(&coin) >= amount, EInsufficientRepayment);
    // ... complete repayment
}
```

### Common Violations

1. **Adding any ability** - Breaks the enforcement
2. **Wrapping in `Option<T>`** - Requires explicit destruction
3. **Not validating pool ID in receipt** - Allows cross-pool attacks

**Source:** [Trail of Bits - Flash Loan Security](https://blog.trailofbits.com/2025/09/10/how-sui-move-rethinks-flash-loan-security/)

---

## 7. Shared Object Security

**Severity: High**

### 7a. Private to Shared Object Conversion

```move
// Object starts as owned
let obj = MyObject { id: object::new(ctx), sensitive_data: 100 };
transfer::transfer(obj, sender);

// Later, holder can share it - now globally accessible!
transfer::public_share_object(obj);
```

**Risk:** Objects with `store` ability can be shared by anyone who owns them.

### 7b. Frozen Object Surprise

```move
// Object with key+store can be frozen by holder
transfer::public_freeze_object(my_object);  // Now immutable and globally readable
```

**Risk:** Contracts assuming object ownership may be surprised when frozen objects become globally accessible.

### 7c. Shared Object Contention

Unlike traditional race conditions (which Sui prevents), contention issues remain:
- Hot object bottlenecks when many users access same shared object
- Version mismatch failures when concurrent transactions target same object

**Best Practice:** Split data into user-specific objects rather than central shared pools.

**Source:** [Zellic - Move Security Part 2](https://www.zellic.io/blog/move-fast-break-things-move-security-part-2/)

---

## 8. Denial of Service (DoS)

**Severity: High to Medium**

### 8a. Unbounded Iteration

```move
// VULNERABLE: Attacker adds many orders, blocks all users
public fun get_order_by_id(orders: &vector<Order>, id: u64): &Order {
    let i = 0;
    while (i < vector::length(orders)) {  // Unbounded loop
        if (vector::borrow(orders, i).id == id) {
            return vector::borrow(orders, i)
        };
        i = i + 1;
    };
    abort EOrderNotFound
}
```

**Attack:** Register thousands of orders, make critical functions hit gas limits.

**Fix:** Limit loop iterations, use indexed structures, or incentivize cleanup.

### 8b. Global Resource Storage

```move
// VULNERABLE: All orders stored globally
struct OrderBook has key {
    id: UID,
    orders: vector<Order>,  // Grows unbounded, affects all users
}
```

**Fix:** Store resources within user accounts (per-user storage).

### 8c. External Dependency Failures

If external modules fail or are upgraded incompatibly, dependent contracts may become unusable.

**Source:** [Zellic - Top 10 Aptos Move Bugs](https://www.zellic.io/blog/top-10-aptos-move-bugs/)

---

## 9. Return Value Handling

**Severity: Medium**

### Unchecked Return Values

```move
// VULNERABLE: Ignoring return value
public fun process(data: &mut Data) {
    validate_something(data);  // Returns bool, but not checked!
    // Continues even if validation failed
}
```

**Fix:** Always handle or assert on return values:

```move
assert!(validate_something(data), EValidationFailed);
```

### Option Extraction Errors

```move
// VULNERABLE: Using wrong Option function
let value = option::extract(&mut opt);  // Removes value
let same_value = option::borrow(&opt);   // ABORTS - opt is now empty!
```

**Source:** [SlowMist - Sui Move Auditing Primer](https://github.com/slowmist/Sui-MOVE-Smart-Contract-Auditing-Primer)

---

## 10. Clock and Timestamp Issues

**Severity: Medium**

### Sui Clock Security Model

Unlike Ethereum, Sui's Clock is consensus-controlled:
- Singleton at address `0x6`
- Read-only access only (immutable reference)
- Validators update timestamp at consensus
- Cannot be manipulated by users

### Remaining Risks

#### Time Unit Confusion
```move
// VULNERABLE: Clock returns milliseconds, not seconds
let now = clock::timestamp_ms(clock);  // Milliseconds!
if (now > deadline_seconds) { ... }    // Comparing wrong units
```

#### Granularity Issues
```move
// RISKY: Relying on exact timing
if (current_time == exact_unlock_time) { ... }  // May never match
```

**Best Practice:** Use grace periods and leeway in time comparisons.

**Source:** [Sui Documentation - Access On-Chain Time](https://docs.sui.io/guides/developer/sui-101/access-time)

---

## 11. Token and Coin Management

**Severity: High**

### 11a. Unfrozen Coin Metadata

```move
fun init(witness: MYCOIN, ctx: &mut TxContext) {
    let (treasury, metadata) = coin::create_currency(...);
    // VULNERABLE: Admin can change token name/symbol
    transfer::public_share_object(metadata);
}
```

**Fix:**

```move
transfer::public_freeze_object(metadata);  // Immutable forever
```

### 11b. Nested Token Confusion

Sui's nested token model creates risks:
- Inaccurate consumption calculations
- Improper transfers between nested structures
- Incorrect token splits/merges

### 11c. Account Registration (Aptos-specific)

```move
// VULNERABLE: No check if recipient can receive coin
coin::deposit(recipient, coins);  // May abort if not registered
```

**Fix:** Check `coin::is_account_registered` before deposit.

**Source:** [MoveBit - Sui Objects Security](https://www.movebit.xyz/blog/post/Sui-Objects-Security-Principles-and-Best-Practices.html)

---

## 12. Package Upgrade Hazards

**Severity: High**

### 12a. Init Function Doesn't Re-run

```move
fun init(ctx: &mut TxContext) {
    // This ONLY runs at first deployment, NOT on upgrades
    setup_critical_state(ctx);
}
```

**Risk:** Upgrades may need manual migration scripts.

### 12b. Old Package Versions Remain Callable

All package versions remain on-chain forever. Old versions may:
- Violate invariants maintained by new versions
- Allow attacks through deprecated functions

### 12c. Single-Signature Upgrade Risk (Nemo Exploit - $2.6M)

**Attack:** Unaudited code deployed via single-signature upgrade.

**Mitigation:**
- Multi-sig upgrade policies
- Mandatory audit before deployment
- Timelock on upgrades

**Source:** [The Block - Nemo Protocol Exploit](https://www.theblock.co/post/370273/nemo-protocol-unaudited-code-exploit)

---

## 13. External Dependency Risks

**Severity: Critical**

### The Cetus Lesson

Despite 3 professional audits, Cetus lost $223M because:
- The bug was in `integer-mate`, a widely-used library
- Auditors assumed "everyone uses this, it must be safe"
- The vulnerability was an incorrect bit-shift validation

### Audit Recommendations

1. **Audit dependencies with same rigor as own code**
2. **Review all math libraries for edge cases**
3. **Don't assume popular = secure**
4. **Check for updates/patches in dependencies**

**Source:** [Cyfrin - Cetus Exploit Analysis](https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis)

---

## 14. Oracle Manipulation

**Severity: High**

### Internal Price Oracle Attacks

```move
// VULNERABLE: Using pool ratio as price
let price = pool.token_a_balance / pool.token_b_balance;
```

**Attack:** Flash loan to manipulate ratio, exploit price, repay loan.

### Mitigations

1. **TWAP (Time-Weighted Average Prices)** - Resist single-transaction manipulation
2. **Multiple independent oracles** - Cross-reference prices
3. **Circuit breakers** - Pause on extreme price movements
4. **Price bounds** - Reject transactions outside reasonable ranges

**Source:** [Zellic - Top 10 Aptos Move Bugs](https://www.zellic.io/blog/top-10-aptos-move-bugs/)

---

## 15. Witness Pattern Violations

**Severity: High**

The witness pattern proves module ownership by constructing a type.

### One-Time Witness (OTW) Requirements

1. Named after module in UPPERCASE (`module foo` → `struct FOO`)
2. Has only `drop` ability
3. Created only in `init` function
4. Must be consumed immediately (not stored)

```move
module example::my_coin {
    struct MY_COIN has drop {}  // OTW

    fun init(witness: MY_COIN, ctx: &mut TxContext) {
        // Use witness to create currency
        let (treasury, metadata) = coin::create_currency(
            witness,  // Consumed here
            ...
        );
    }
}
```

### Violations

- Creating multiple witnesses
- Not consuming witness immediately
- Storing witness (shouldn't be possible with only `drop`)

**Source:** [Sui Documentation - Witness Pattern](https://docs.sui.io/concepts/sui-move-concepts/patterns/witness)

---

## 16. UID and Object Identity Issues

**Severity: Medium**

### Historical: UID Swapping (Patched)

Previously, objects could have their UIDs swapped:

```move
// HISTORICAL BUG (patched Dec 2022)
public entry fun transmute(cat: Cat, dog: Dog, ctx: &mut TxContext) {
    let Cat { id: cat_id } = cat;
    let Dog { id: dog_id } = dog;
    transfer::transfer(Cat { id: dog_id }, tx_context::sender(ctx));
    transfer::transfer(Dog { id: cat_id }, tx_context::sender(ctx));
}
```

### Current Protections

- Sui bytecode verifier ensures new objects get fresh UIDs
- ID leak verifier prevents UID reuse
- IDs are never reused on-chain

### Off-Chain Risks

- Object hiding/resurrection can confuse off-chain indexers
- Type changes may not be properly tracked by external systems

**Source:** [Zellic - Move Security Part 2](https://www.zellic.io/blog/move-fast-break-things-move-security-part-2/)

---

## What Move Prevents (Compared to EVM)

Move's design eliminates several vulnerability classes common in Solidity:

| Vulnerability | EVM Status | Move Status |
|--------------|------------|-------------|
| Reentrancy | Common | **Eliminated** (no callbacks) |
| Integer overflow (+, *, -) | Historical | **Auto-abort** |
| Double-spending | Possible | **Eliminated** (linear types) |
| Wallet drainer approvals | Common | **Eliminated** (asset ownership) |
| Uninitialized storage | Possible | **Eliminated** (no uninitialized state) |
| Delegatecall attacks | Common | **Eliminated** (no delegatecall) |

### What Move Does NOT Prevent

- Logic errors in business logic
- Math library bugs (especially bit operations)
- Access control misconfigurations
- Oracle manipulation
- Front-running/MEV
- Off-by-one errors

---

## Audit Checklist Summary

### Pre-Audit Setup
- [ ] Obtain design documentation
- [ ] Map all function permissions
- [ ] Identify all object types and their abilities
- [ ] List all external dependencies

### Critical Checks
- [ ] **Abilities:** Every struct has minimal required abilities
- [ ] **Hot Potatoes:** Enforcement structs have NO abilities
- [ ] **Access Control:** All entry functions require proper authorization
- [ ] **Object Validation:** Relationships between shared objects are verified
- [ ] **Generic Types:** Phantom types bind receipts to correct assets
- [ ] **AdminCaps:** Use `transfer`, not `share_object`
- [ ] **Metadata:** Coin/NFT metadata is frozen

### Math and Logic
- [ ] All bit shifts have proper overflow checks
- [ ] Division operations handle precision loss
- [ ] Time units are consistent (ms vs s)
- [ ] Loop iterations are bounded

### Dependencies
- [ ] External packages audited
- [ ] Math libraries reviewed for edge cases
- [ ] Upgrade policies verified

---

## References

### Audit Firms
- [OtterSec](https://osec.io/)
- [MoveBit](https://movebit.xyz/)
- [Zellic](https://www.zellic.io/)
- [SlowMist](https://www.slowmist.com/)
- [Blaize Security](https://security.blaize.tech/)

### Security Research
- [Zellic - Top 10 Aptos Move Bugs](https://www.zellic.io/blog/top-10-aptos-move-bugs/)
- [Zellic - Move Security Part 2](https://www.zellic.io/blog/move-fast-break-things-move-security-part-2/)
- [Zellic - Billion Dollar Bug](https://www.zellic.io/blog/the-billion-dollar-move-bug/)
- [Mirage Audits - Ability Mistakes](https://www.mirageaudits.com/blog/sui-move-ability-security-mistakes)
- [MoveBit - Sui Objects Security](https://www.movebit.xyz/blog/post/Sui-Objects-Security-Principles-and-Best-Practices.html)

### Incident Analysis
- [Dedaub - Cetus Hack Analysis](https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/)
- [Cyfrin - Cetus Exploit](https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis)

### Official Resources
- [Sui Security Audits Repository](https://github.com/sui-foundation/security-audits)
- [Sui Security Page](https://www.sui.io/security)
- [SlowMist Audit Primer](https://github.com/slowmist/Sui-MOVE-Smart-Contract-Auditing-Primer)
- [Move Audit Resources](https://github.com/0xriazaka/Move-Audit-Resources)
