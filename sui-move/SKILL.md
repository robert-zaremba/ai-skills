---
name: sui-move
description: Sui Move smart contract development - patterns, best practices, standard library, and idiomatic code for the Sui blockchain
license: MIT
compatibility: opencode, kilo
metadata:
  author: 
    - Robert Zaremba
    - Sui Move Book Authors
  version: "1.0.0"
  tags: blockchain, smart-contracts, move, sui, web3
---

# Sui Move Smart Contract Development Skill

Expert guidance for developing secure, efficient, and idiomatic smart contracts using Sui Move - a resource-oriented programming language for the Sui blockchain.

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Project Structure](#project-structure)
3. [Module Initializer](#module-initializer)
4. [Type System & Abilities](#type-system--abilities)
5. [Object Model](#object-model)
   - Ownership Types, Storage Functions, Transfer to Object (TTO)
   - Restricted vs Public Transfer, Internal Constraint
6. [Design Patterns](#design-patterns)
   - Capability, Witness, OTW, Hot Potato, Publisher, Versioned State
   - (Optional: Builder for complex test objects)
7. [Anti-Patterns](#anti-patterns)
8. [Standard Library Reference](#standard-library-reference)
9. [Sui Framework Reference](#sui-framework-reference)
10. [Transaction Context](#transaction-context)
11. [Epoch and Time](#epoch-and-time)
12. [Object Display](#object-display)
13. [BCS Serialization](#bcs-serialization)
14. [Data Storage Patterns](#data-storage-patterns)
15. [Error Handling](#error-handling)
16. [Events](#events)
17. [Testing](#testing)
    - Good Tests, Annotations, Random Tests, Utilities
    - Test Scenario, System Objects, Module Extensions
    - Coverage, Gas Profiling, Anti-Patterns
18. [Security Considerations](#security-considerations)
19. [Performance & Gas](#performance--gas)
    - Network Limits, Optimization Strategies
20. [Upgradeability](#upgradeability)
21. [Code Quality Checklist](#code-quality-checklist)
22. [CLI Reference](#cli-reference)
23. [Move 2024 Migration](#move-2024-migration)
24. [Further Resources](#further-resources)

---

## Core Concepts

### Language Philosophy

- **Resource-Oriented**: Move treats digital assets as first-class resources with ownership, non-copyability, and non-discardability by default
- **Type Safety**: Strong type system with abilities system (`copy`, `drop`, `key`, `store`) controls type behavior
- **Module-Centric**: Code organization around modules with private-by-default visibility
- **No Reentrancy**: Language design prevents reentrancy attacks by default

### Key Abstractions

| Concept | Description |
|---------|-------------|
| **Objects** | Structs with `key` ability representing on-chain assets with unique IDs |
| **Packages** | Collections of modules published at unique addresses |
| **Transactions** | State changes executed through function calls with explicit object references |
| **Abilities** | Type-level properties that control value behavior (`copy`, `drop`, `key`, `store`) |

### Sui vs Other Blockchains

| Feature | Sui Move | Solidity/EVM |
|---------|----------|--------------|
| Asset representation | Native (structs with abilities) | Smart contract state |
| Ownership | Built-in object model | Manual tracking |
| Parallel execution | Yes (owned objects) | Sequential |
| Reentrancy | Prevented by design | Manual guards needed |
| Upgrades | Native support | Proxy patterns |


## Project Structure

### Standard Package Layout

```
my_package/
├── Move.toml           # Package manifest
├── sources/            # Production code
│   ├── module_one.move
│   └── module_two.move
├── tests/              # Test files (not published)
│   ├── module_one_tests.move
│   └── extensions/     # Module extensions for testing
│       └── external_ext.move
├── examples/           # Examples (not published)
│   └── usage_example.move
└── build/              # Build artifacts (gitignored)
```

### Move.toml Configuration

```toml
[package]
name = "my_package"
version = "0.0.0"
edition = "2024"  # Always use 2024 edition

[dependencies]
# Sui system packages auto-included since CLI v1.45:
# MoveStdlib, Sui, System, Bridge, Deepbook
# Only add explicit dependencies for other packages

[addresses]
my_package = "0x0"  # Placeholder until published

[dev-addresses]
# Override addresses for testing
```

### File Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Module | `snake_case.move` | `donut_shop.move` |
| Tests | `module_tests.move` | `donut_shop_tests.move` |
| Extension | `module_ext.move` | `pyth_price_info_ext.move` |


## Module Initializer

The `init` function is a special function called once when a package is published. It's used for one-time setup.

### Init Function Rules

1. Must be named `init`
2. Must be private (no visibility modifier)
3. Returns no value
4. Takes one or two arguments (optional OTW, required TxContext)
5. TxContext must be the last argument
6. Called automatically on publish (not on upgrades)

### Basic Init

```move
module my_app::store;

public struct Store has key { id: UID }

// Simple init with just context
fun init(ctx: &mut TxContext) {
    transfer::transfer(
        Store { id: object::new(ctx) },
        ctx.sender()
    );
}
```

### Init with OTW

```move
module my_app::token;

// OTW for single-instance guarantee
public struct TOKEN has drop {}

fun init(otw: TOKEN, ctx: &mut TxContext) {
    // OTW proves this is the only time this runs
    let publisher = package::claim(otw, ctx);
    
    // Create initial capabilities
    let admin_cap = AdminCap { id: object::new(ctx) };
    
    transfer::public_transfer(publisher, ctx.sender());
    transfer::transfer(admin_cap, ctx.sender());
}
```

### Multiple Modules

Each module can have its own `init` function - all are called during publish:

```move
// sources/module_a.move
module my_app::module_a;
fun init(ctx: &mut TxContext) { /* setup A */ }

// sources/module_b.move  
module my_app::module_b;
fun init(ctx: &mut TxContext) { /* setup B */ }
```

### Security Considerations

```move
// ⚠️ WARNING: init doesn't prevent creating more capabilities later
public struct AdminCap has key { id: UID }

fun init(ctx: &mut TxContext) {
    transfer::transfer(AdminCap { id: object::new(ctx) }, ctx.sender());
}

// This function can be added in an upgrade!
public fun create_another_admin(ctx: &mut TxContext): AdminCap {
    AdminCap { id: object::new(ctx) }
}

// ✓ BETTER: Use OTW + enforce single instance via shared state
public struct ADMIN has drop {}
public struct AdminRegistry has key { id: UID, admin: Option<address> }

fun init(otw: ADMIN, ctx: &mut TxContext) {
    let registry = AdminRegistry {
        id: object::new(ctx),
        admin: option::some(ctx.sender()),
    };
    transfer::share_object(registry);
}
```

## Type System & Abilities

### Four Core Abilities

| Ability | Purpose | What It Enables |
|---------|---------|-----------------|
| `copy` | Value duplication | `let b = a;` copies `a` |
| `drop` | Value discard | Value can go out of scope unused |
| `key` | Global storage | Struct can be stored as an object |
| `store` | Nested storage | Struct can be a field in another struct |

### Ability Combinations & Patterns

```move
// Hot Potato - no abilities, must be consumed in same transaction
public struct Request { id: u64 }

// Witness - drop only, proves type ownership  
public struct WITNESS has drop {}

// Event - copy + drop for off-chain notifications
public struct TransferEvent has copy, drop { from: address, to: address }

// Config/Data - copy + drop + store for flexible data
public struct Metadata has copy, drop, store { value: u64 }

// Object (soulbound) - key only, restricted transfer
public struct Badge has key { id: UID, level: u8 }

// Object (transferable) - key + store, public transfer enabled
public struct Coin has key, store { id: UID, value: u64 }

// Full abilities - maximum flexibility, rarely needed for objects
public struct FlexibleData has key, store, copy, drop { id: UID }
```

### Ability Selection Decision Tree

```
Is this type an on-chain object?
├── Yes → Needs `key`
│   ├── Should it be publicly transferable?
│   │   ├── Yes → Add `store`
│   │   └── No → `key` only (restricted transfer)
│   └── Can fields be copied/dropped?
│       └── Fields must have `store` ability
│
└── No → Not an object
    ├── Is it an event?
    │   └── `copy + drop`
    ├── Is it temporary data?
    │   └── `copy + drop` (or just `drop`)
    ├── Must it be consumed (hot potato)?
    │   └── No abilities
    └── Is it a witness?
        └── `drop` only
```

### Phantom Type Parameters

Use phantom types when the type parameter doesn't affect the struct's fields:

```move
// T is phantom - not used in fields, doesn't require abilities
public struct Coin<phantom T> has key, store {
    id: UID,
    value: u64,
}

// Can instantiate with any T, regardless of abilities
public struct USD {}
public struct BTC {}

let usd_coin: Coin<USD> = coin::mint(100, ctx);
let btc_coin: Coin<BTC> = coin::mint(50, ctx);
```

### Generic Constraints

```move
// Require specific abilities on type parameters
public fun process<T: copy + drop>(value: T) { /* ... */ }
public fun store_item<T: store>(item: T, container: &mut Container) { /* ... */ }

// Multiple constraints
public fun transfer_internal<T: key + store>(obj: T, to: address) {
    transfer::public_transfer(obj, to);
}
```

## Object Model

### Object Definition Requirements

```move
public struct MyObject has key {
    id: UID,  // REQUIRED: Must be first field, named exactly "id"
    // All other fields must have `store` ability
    value: u64,           // u64 has store ✓
    owner: address,       // address has store ✓
    metadata: String,     // String has store ✓
    config: Config,       // Config must have store ✓
}

public struct Config has store {  // Required for use in MyObject
    threshold: u64,
}
```

### Ownership Types

| Type | Use Case | Transfer Function | Access Pattern |
|------|----------|-------------------|----------------|
| **Single Owner** | Personal assets, NFTs | `transfer::transfer(obj, addr)` | Only owner |
| **Shared** | Marketplaces, DEXs, pools | `transfer::share_object(obj)` | Anyone (mutable) |
| **Immutable** | Config, metadata, constants | `transfer::freeze_object(obj)` | Anyone (read-only) |
| **Object-Owned** | Parent-child relationships | Transfer to object's address | Via parent |

### Storage Functions Reference

```move
// === TRANSFER (Single Owner) ===

// Restricted - only defining module can call
transfer::transfer<T: key>(obj: T, recipient: address)

// Public - anyone can call if T has store
transfer::public_transfer<T: key + store>(obj: T, recipient: address)

// === SHARE (Shared State) ===

transfer::share_object<T: key>(obj: T)
transfer::public_share_object<T: key + store>(obj: T)

// === FREEZE (Immutable) ===

transfer::freeze_object<T: key>(obj: T)
transfer::public_freeze_object<T: key + store>(obj: T)

// === RECEIVE (From Object) ===

transfer::receive<T: key>(parent: &mut UID, receiving: Receiving<T>): T
transfer::public_receive<T: key + store>(parent: &mut UID, receiving: Receiving<T>): T
```

### Restricted vs Public Transfer

Storage functions have two variants:

| Function | Restricted Version | Public Version | Requirement |
|----------|-------------------|----------------|-------------|
| Transfer | `transfer<T: key>` | `public_transfer<T: key + store>` | `store` for public |
| Share | `share_object<T: key>` | `public_share_object<T: key + store>` | `store` for public |
| Freeze | `freeze_object<T: key>` | `public_freeze_object<T: key + store>` | `store` for public |
| Receive | `receive<T: key>` | `public_receive<T: key + store>` | `store` for public |

**Restricted Functions:**
- Can only be called by the module that defines type `T`
- Enforced by Sui Bytecode Verifier
- Use when you want full control over object lifecycle

**Public Functions:**
- Can be called from any module
- Require type `T` to have `store` ability
- Enable composability with other protocols

**Implications of `store` Ability:**

```move
// Without store - restricted transfer only
public struct SoulboundNFT has key { id: UID, owner: address }
// Only this module can call transfer::transfer()

// With store - public transfer enabled
public struct TransferableNFT has key, store { id: UID, owner: address }
// Any module can call transfer::public_transfer()
// Can be wrapped in other structs
// Can be stored in dynamic fields
```

**Decision Guide:**

```
Does the object need to be transferred by other modules?
├── Yes → Add `store`, use public_* functions
└── No → Omit `store`, keep restricted transfer
    └── Provides stronger guarantees about object lifecycle
```

### Transfer to Object (TTO)

Objects can be owned by other objects, not just addresses. The `Receiving` type enables this:

```move
use sui::transfer::{Self, Receiving};

// Object owned by another object
public struct PostBox has key {
    id: UID,
    owner: address,
}

// Send to a PostBox (object-owned)
public fun send_to_postbox(
    box_id: ID,
    item: Package,
    receiving: Receiving<Package>,  // Transaction argument
    ctx: &mut TxContext
) {
    // Get parent UID via dynamic field or direct access
    let parent_id = /* get parent UID */;
    
    // Receive the item
    let received = transfer::receive<Package>(&mut parent_id, receiving);
    
    // Process/store the received item
    // ...
}

// Transfer to object's address
public fun deliver_to_box(box: &PostBox, item: Package) {
    // Convert object ID to address
    let box_addr = object::id_address(box);
    transfer::public_transfer(item, box_addr);
}
```

**TTO Use Cases:**
- Parallel execution: Transfer to multiple objects without referencing in transaction
- Container objects: Parent object owns child objects
- PostBox-like apps: Users receive assets after account activation
- Account abstraction: Objects acting as accounts

**Important:**
- `Receiving<T>` is a special transaction argument type
- Must explicitly import `sui::transfer::Receiving`
- `receive` subject to internal constraint (T must be defined in calling module)
- `public_receive` requires `T: key + store`

### Transfer Functions Summary

| Function | Public Variant | End State | Permissions |
|----------|---------------|-----------|-------------|
| `transfer` | `public_transfer` | Address Owned | Full |
| `share_object` | `public_share_object` | Shared | Ref, Mut Ref, Delete |
| `freeze_object` | `public_freeze_object` | Frozen | Immutable Ref only |
| `receive` | `public_receive` | - | Via parent UID |

**Object States:**

| State | Description |
|-------|-------------|
| Address Owned | Full access by owner address or object |
| Shared | Anyone can access (mutable), requires consensus |
| Frozen | Immutable, read-only access |

### Ownership Conversions

| From | To | Possible? | Notes |
|------|-----|-----------|-------|
| Single Owner | Shared | ✓ | Via `share_object` |
| Single Owner | Immutable | ✓ | Via `freeze_object` |
| Shared | Single Owner | ✗ | Not allowed |
| Shared | Immutable | ✗ | Not allowed |
| Immutable | Any | ✗ | Permanent |

### Internal Constraint

Some Sui Framework functions require the type parameter `T` to be defined in the calling module (internal to the caller):

```move
// This requires T to be defined in YOUR module
event::emit<T: copy + drop>(event: T)

// This works - MyEvent is defined here
public struct MyEvent has copy, drop { value: u64 }
public fun emit_event() {
    event::emit(MyEvent { value: 42 });
}

// This FAILS - TypeName is from std
public fun fail() {
    event::emit(type_name::get<MyStruct>()); // Error!
}
```

**Functions with Internal Constraint:**

| Function | Constraint |
|----------|------------|
| `event::emit<T>` | T must be internal |
| `transfer::transfer<T>` | T must be internal |
| `transfer::share_object<T>` | T must be internal |
| `transfer::freeze_object<T>` | T must be internal |
| `transfer::receive<T>` | T must be internal |
| `display::new<T>` | T must be internal |

**Public variants** (`public_transfer`, etc.) bypass this constraint by requiring `T: store`.

## Design Patterns

### 1. Capability Pattern

Use owned objects as access tokens for privileged operations.

```move
public struct AdminCap has key { id: UID }

// Create in init function - guaranteed single instance
fun init(ctx: &mut TxContext) {
    transfer::transfer(
        AdminCap { id: object::new(ctx) }, 
        ctx.sender()
    );
}

// Gate functions with capability reference
public fun admin_action(_: &AdminCap, params: Params) {
    // Only AdminCap owner can call
}

// Multiple capability types for different permissions
public struct MinterCap has key { id: UID }
public struct BurnerCap has key { id: UID }
public struct PauserCap has key { id: UID }
```

**When to Use:**
- Admin operations
- Role-based access control
- Permissioned minting/burning
- Upgrade authorization

**Best Practices:**
- Name with `Cap` suffix
- Create in `init` for single-instance
- Use `key` only (no `store`) to control transfer
- Pass by immutable reference (`&Cap`)

### 2. Witness Pattern

Prove type ownership by constructing an instance.

```move
// Framework function requiring witness
module sui::balance;
public fun create_supply<T: drop>(_witness: T): Supply<T>;

// Your module proves ownership of MY_TOKEN
module my_app::token;
public struct MY_TOKEN has drop {}

public fun init(ctx: &mut TxContext) {
    let supply = balance::create_supply(MY_TOKEN {});
    // supply is now bound to MY_TOKEN type
}
```

**Use Cases:**
- Creating typed supplies (coins, tokens)
- Registering types with frameworks
- Type-bound initialization

### 3. One-Time Witness (OTW)

Guarantee single instantiation for critical operations.

**OTW Rules:**
1. Has only `drop` ability
2. Has no fields
3. Is not a generic type
4. Named after module in UPPERCASE

```move
module my_app::coin;

// OTW - follows all rules
public struct MY_COIN has drop {}

fun init(otw: MY_COIN, ctx: &mut TxContext) {
    // OTW can ONLY be received here, never constructed manually
    let (treasury, metadata) = coin::create_currency(
        otw, 
        6,  // decimals
        b"MY COIN", 
        b"SYMBOL", 
        b"Description", 
        option::none(),
        ctx
    );
    
    transfer::public_transfer(treasury, ctx.sender());
    transfer::public_transfer(metadata, ctx.sender());
}
```

### 4. Hot Potato Pattern

Enforce transaction completion for workflows.

```move
// No abilities = must be consumed, cannot be stored
public struct BorrowReceipt { 
    object_id: ID, 
    must_return_to: ID 
}

public fun borrow(container: &mut Container): (Item, BorrowReceipt) {
    let item = container.remove();
    let receipt = BorrowReceipt {
        object_id: object::id(&item),
        must_return_to: object::id(container),
    };
    (item, receipt)
}

public fun return_item(item: Item, receipt: BorrowReceipt, container: &mut Container) {
    // receipt is consumed here - you MUST call this function
    assert!(object::id(container) == receipt.must_return_to, EWrongContainer);
    container.add(item);
}
```

**Use Cases:**
- Flash loans (borrow and repay in same tx)
- Borrowing with guarantees
- Multi-step workflows
- Request/approval patterns

**Hot Potato with sui::borrow:**

```move
use sui::borrow;

// borrow module provides a hot potato for borrowing values
public fun borrow_value<T: drop + store>(
    container: &mut Container<T>
): (&mut T, Borrow<T>) {
    let borrowed = borrow::borrow(container);
    // Borrow<T> is a hot potato - must be returned
    (borrowed, borrowed)
}

public fun return_value<T: drop + store>(
    _returned: T,
    receipt: Borrow<T>,
    container: &mut Container<T>
) {
    borrow::return_(receipt, container);
}
```

**Flash Loan Example:**

```move
public struct FlashLoanReceipt { amount: u64, borrower: address }

public fun borrow_flash(
    pool: &mut Pool,
    amount: u64,
    ctx: &mut TxContext
): (Coin, FlashLoanReceipt) {
    let coin = pool.withdraw(amount);
    (coin, FlashLoanReceipt { amount, borrower: ctx.sender() })
}

public fn repay_flash(
    coin: Coin,
    receipt: FlashLoanReceipt,
    pool: &mut Pool
) {
    assert!(coin.value() >= receipt.amount, EInsufficientRepayment);
    pool.deposit(coin);
    // Receipt consumed - loan completed
}

// Usage: borrow, use funds, repay - all in same transaction
let (coin, receipt) = pool.borrow_flash(1000, ctx);
// ... use coin for arbitrage, etc ...
let repayment = /* get repayment coin */;
pool.repay_flash(repayment, receipt);
```
- Ensuring callback execution

### 5. Wrapper Type Pattern

Restrict or extend existing type behavior.

```move
// Immutable wrapper - read-only vector
public struct ImmutableVec<T: store> has store { inner: vector<T> }

public fun new<T: store>(v: vector<T>): ImmutableVec<T> { 
    ImmutableVec { inner: v } 
}

public fun borrow<T: store>(v: &ImmutableVec<T>, i: u64): &T { 
    &v.inner[i] 
}

public fun length<T: store>(v: &ImmutableVec<T>): u64 { 
    v.inner.length() 
}
// No push/pop methods - read-only enforced!

// Validated wrapper - ensures invariants
public struct PositiveNumber has store { value: u64 }

public fun new(value: u64): Option<PositiveNumber> {
    if (value > 0) option::some(PositiveNumber { value })
    else option::none()
}

public fun value(n: &PositiveNumber): u64 { n.value }
// Value is guaranteed positive by constructor
```

### 6. Builder Pattern (Optional, Testing Only)

> **Note**: This pattern is rarely needed. Use direct struct construction for most cases. Only consider builders for very complex test objects with many optional fields.

Construct complex test objects fluently. Only useful when:
- Object has 5+ fields, most optional
- Many test variations needed
- Direct construction becomes unwieldy

```move
// Usually, direct construction is sufficient:
let user = User { name: b"Alice".to_string(), balance: 1000, age: 25, is_active: true };

// Builder only for complex cases with many optional fields:
#[test_only]
public struct UserBuilder has drop {
    name: Option<String>,
    age: Option<u8>,
    balance: Option<u64>,
    is_active: Option<bool>,
}

#[test_only]
public fun new(): UserBuilder {
    UserBuilder {
        name: option::none(),
        age: option::none(),
        balance: option::none(),
        is_active: option::none(),
    }
}

#[test_only]
public fun name(mut self: UserBuilder, name: String): UserBuilder {
    self.name = option::some(name);
    self
}

#[test_only]
public fun build(self: UserBuilder): User {
    User {
        name: self.name.destroy_or!(b"Default".to_string()),
        age: self.age.destroy_or!(25),
        balance: self.balance.destroy_or!(0),
        is_active: self.is_active.destroy_or!(true),
    }
}

// Usage (only when complexity justifies it)
let user = user_builder::new()
    .name(b"Alice".to_string())
    .balance(1000)
    .build();
```

**When to Avoid:**
- Simple objects with few fields
- Most fields are required
- Only a few test variations needed

### 7. Publisher Pattern

Prove package authority for type-level operations.

```move
module my_app::nft;

public struct NFT_MODULE has drop {}  // OTW

fun init(otw: NFT_MODULE, ctx: &mut TxContext) {
    let publisher = package::claim(otw, ctx);
    transfer::public_transfer(publisher, ctx.sender());
}

// Later, use publisher to create Display
public fun create_display(
    publisher: &Publisher, 
    ctx: &mut TxContext
): Display<NFT> {
    display::new_with_fields(publisher, ctx)
}
```

### 8. Versioned State Pattern

Handle upgrades with versioned shared state.

```move
const VERSION: u16 = 2;

public struct AppState has key {
    id: UID,
    version: u16,
}

public fun mutate(state: &mut AppState) {
    assert!(state.version == VERSION, EVersionMismatch);
    // ... logic
}

// Admin can upgrade version after code upgrade
public fn upgrade_version(cap: &AdminCap, state: &mut AppState) {
    state.version = VERSION;
}
```

## Anti-Patterns

### Code Organization Anti-Patterns

```move
// ❌ BAD: Module block syntax (legacy)
module my_package::my_module {
    public struct A {}
}

// ✓ GOOD: Module label syntax
module my_package::my_module;
public struct A {}

// ❌ BAD: Redundant Self import
use my_package::my_module::{Self};

// ✓ GOOD
use my_package::my_module;

// ❌ BAD: Separate imports for same module
use my_package::my_module;
use my_package::my_module::Helper;

// ✓ GOOD: Grouped imports
use my_package::my_module::{Self, Helper};
```

### Struct Anti-Patterns

```move
// ❌ BAD: Capability without Cap suffix
public struct Admin has key { id: UID }

// ✓ GOOD
public struct AdminCap has key { id: UID }

// ❌ BAD: Event not in past tense
public struct RegisterUser has copy, drop { user: address }

// ✓ GOOD
public struct UserRegistered has copy, drop { user: address }

// ❌ BAD: Hot potato named with "Potato"
public struct PromisePotato {}

// ✓ GOOD
public struct Promise {}

// ❌ BAD: Named fields for dynamic field keys
public struct DynamicField has copy, drop, store { value: u8 }

// ✓ GOOD: Positional with Key suffix
public struct DynamicFieldKey(u8) has copy, drop, store;
```

### Function Anti-Patterns

```move
// ❌ BAD: public entry - entry is redundant
public entry fun do_something() {}

// ✓ GOOD: Use public for composability
public fun do_something(): T {}

// Or use entry intentionally for non-composable entry points
entry fun mint_and_keep(ctx: &mut TxContext) {}

// ❌ BAD: Not composable, hard to test
public fun mint_and_transfer(ctx: &mut TxContext) {
    let nft = mint(ctx);
    transfer::transfer(nft, ctx.sender());
}

// ✓ GOOD: Composable for PTBs
public fun mint(ctx: &mut TxContext): NFT {
    NFT { id: object::new(ctx) }
}

// ❌ BAD: Wrong parameter order
public fun authorize(cap: &AdminCap, app: &mut App) {}

// ✓ GOOD: Objects first, caps second (maintains method syntax)
public fun authorize(app: &mut App, cap: &AdminCap) {}

// ❌ BAD: Getter with get_ prefix
public fun get_name(u: &User): String {}

// ✓ GOOD
public fun name(u: &User): String {}
public fun name_mut(u: &mut User): &mut String {}
```

### Error Handling Anti-Patterns

```move
// ❌ BAD: All-caps for error constants
const NOT_AUTHORIZED: u64 = 0;

// ✓ GOOD: E prefix + PascalCase
const ENotAuthorized: u64 = 0;

// ❌ BAD: Assert function that aborts
public fun assert_authorized() {
    assert!(/* condition */, ENotAuthorized);
}

// ✓ GOOD: Return bool for flexibility
public fun is_authorized(): bool {
    /* condition */
}

// ❌ BAD: Same error code for different cases
public fun do_something() {
    assert!(condition_a, 0);
    assert!(condition_b, 0);  // Which failed?
}

// ✓ GOOD: Different codes for different cases
public fun do_something() {
    assert!(condition_a, EConditionAFailed);
    assert!(condition_b, EConditionBFailed);
}
```

### Testing Anti-Patterns

```move
// ❌ BAD: Separate test and expected_failure
#[test]
#[expected_failure]
fun test_abort() {}

// ✓ GOOD
#[test, expected_failure]
fun test_abort() {}

// ❌ BAD: Cleaning up in expected_failure test
#[test, expected_failure(abort_code = EError)]
fun test_fail() {
    let test = test_scenario::begin(@0);
    my_app::call(test.ctx());
    test.end();  // Won't reach here!
}

// ✓ GOOD: Abort happens before cleanup
#[test, expected_failure(abort_code = EError)]
fun test_fail() {
    let test = test_scenario::begin(@0);
    my_app::call(test.ctx());
    abort  // Clearly shows where failure expected
}

// ❌ BAD: Using TestScenario when not needed
let test = test_scenario::begin(@0);
let obj = my_app::create(test.ctx());
my_app::destroy(obj);
test.end();

// ✓ GOOD: Use dummy context for simple tests
let ctx = &mut tx_context::dummy();
my_app::create(ctx).destroy();

// ❌ BAD: Test prefix in test module
module my_app::my_module_tests;
#[test]
fun test_feature() {}

// ✓ GOOD: Descriptive name without prefix
module my_app::my_module_tests;
#[test]
fun feature_works_as_expected() {}

// ❌ BAD: assert! with abort code in tests
assert!(result == expected, 0);

// ✓ GOOD
assert!(result == expected);  // No code needed
assert_eq!(result, expected);  // Better diagnostics
```

### Syntax Anti-Patterns

```move
// ❌ BAD: Manual string creation
use std::string::utf8;
let s = utf8(b"hello");

// ✓ GOOD
let s = b"hello".to_string();

// ❌ BAD: Old vector syntax
let mut v = vector::empty();
vector::push_back(&mut v, 1);

// ✓ GOOD
let mut v = vector[1];
v.push_back(2);

// ❌ BAD: Explicit loop for simple iteration
let mut i = 0;
while (i < 10) {
    process(i);
    i = i + 1;
};

// ✓ GOOD
10.do!(|i| process(i));

// ❌ BAD: Manual option handling
if (opt.is_some()) {
    let v = opt.destroy_some();
    use_value(v);
} else {
    abort EError
};

// ✓ GOOD
opt.do!(|v| use_value(v));
let v = opt.destroy_or!(abort EError);

// ❌ BAD: Manual vector iteration
let mut i = 0;
while (i < vec.length()) {
    process(&vec[i]);
    i = i + 1;
};

// ✓ GOOD
vec.do_ref!(|e| process(e));

// ❌ BAD: Manual fold
let mut sum = 0;
let mut i = 0;
while (i < nums.length()) {
    sum = sum + nums[i];
    i = i + 1;
};

// ✓ GOOD
let sum = nums.fold!(0, |acc, n| acc + n);
```

## Standard Library Reference

### Core Modules

| Module | Description | Key Functions |
|--------|-------------|---------------|
| `std::string` | UTF-8 strings | `utf8()`, `append()`, `sub_string()`, `length()` |
| `std::ascii` | ASCII strings | `to_string()`, `into_bytes()` |
| `std::option` | Optional values | `some()`, `none()`, `is_some()`, `destroy_some()` |
| `std::vector` | Native vector operations | `push_back()`, `pop_back()`, `length()`, `borrow()` |
| `std::bcs` | Binary serialization | `to_bytes()`, `peel_*()` family |
| `std::type_name` | Type reflection | `get<T>()`, `into_string()` |
| `std::hash` | Hashing functions | `sha2_256()`, `sha3_256()` |
| `std::address` | Address operations | `length()` |

### Integer Modules (Method Syntax)

Each integer type has associated methods:

```move
let max = 100u64.max(50);           // 100
let diff = 50u64.diff(30);          // 20
let div = 7u64.divide_and_round_up(3);  // 3
let sqrt = 16u64.sqrt();            // 4
let pow = 2u64.pow(10);             // 1024
```

### Option Macros

```move
// Execute if some
opt.do!(|value| process(value));

// Destroy with default
let value = opt.destroy_or!(default_value);
let value = opt.destroy_or!(abort ENotFound);

// Extract or default
let value = opt.unwrap_or(default);
```

### Vector Macros

```move
// Create from iteration
let v = vector::tabulate!(10, |i| i * 2);

// Iterate over elements
vec.do!(|e| process(e));
vec.do_ref!(|e| process_ref(e));
vec.do_mut!(|e| mutate(e));

// Destroy and process
vec.destroy!(|e| cleanup(e));

// Fold into single value
let sum = vec.fold!(0, |acc, e| acc + e);

// Filter elements
let filtered = vec.filter!(|e| e > 10);
```

### Implicit Imports

These are available without explicit `use`:

- `std::vector`
- `std::option`  
- `std::option::Option`
- `sui::object`
- `sui::object::ID`
- `sui::object::UID`
- `sui::tx_context`
- `sui::tx_context::TxContext`
- `sui::transfer`

## Sui Framework Reference

### Core Modules

| Module | Description | Key Types/Functions |
|--------|-------------|---------------------|
| `sui::transfer` | Storage operations | `transfer()`, `share_object()`, `freeze_object()` |
| `sui::object` | Object utilities | `new()`, `delete()`, `id()`, `to_inner()` |
| `sui::tx_context` | Transaction context | `sender()`, `epoch()`, `fresh_object_address()` |
| `sui::event` | Event emission | `emit<T>()` |
| `sui::clock` | Time access | `timestamp_ms()` |
| `sui::package` | Publisher authority | `claim()`, `Publisher` |
| `sui::display` | Object metadata | `new()`, `add_field()`, `update()` |

### Collection Modules

| Module | Type | Use Case |
|--------|------|----------|
| `sui::vec_set` | `VecSet<T>` | Unique values |
| `sui::vec_map` | `VecMap<K,V>` | Small key-value maps |
| `sui::table` | `Table<K,V>` | Large typed key-value |
| `sui::bag` | `Bag` | Heterogeneous key-value |
| `sui::linked_table` | `LinkedTable<K,V>` | Ordered key-value |
| `sui::object_table` | `ObjectTable<K,V>` | Object values by ID |
| `sui::object_bag` | `ObjectBag` | Heterogeneous objects |

### Dynamic Fields

| Module | Description | Value Constraint |
|--------|-------------|------------------|
| `sui::dynamic_field` | Key-value attachments | `store` |
| `sui::dynamic_object_field` | Object attachments | `key + store` |

```move
// Dynamic field
dynamic_field::add<Name: copy + drop + store, Value: store>(
    &mut parent.id, name, value
);

// Dynamic object field (value is object, remains discoverable by ID)
dynamic_object_field::add<Name: copy + drop + store, Value: key + store>(
    &mut parent.id, name, object_value
);
```

### Coin & Balance

```move
module sui::coin;

// Create a coin (requires TreasuryCap)
public fun mint<T>(cap: &mut TreasuryCap<T>, value: u64, ctx: &mut TxContext): Coin<T>

// Split a coin
public fun split<T>(coin: &mut Coin<T>, split_amount: u64, ctx: &mut TxContext): Coin<T>

// Join coins
public fun join<T>(self: &mut Coin<T>, other: Coin<T>)

// Get value
public fun value<T>(coin: &Coin<T>): u64

// Convert to balance
public fun into_balance<T>(coin: Coin<T>): Balance<T>

// Test utilities
coin::mint_for_testing<T>(value, ctx)
coin::burn_for_testing(coin)
```

### Utility Modules

| Module | Description |
|--------|-------------|
| `sui::bcs` | BCS encoding/decoding helpers |
| `sui::borrow` | Value borrowing with guarantees |
| `sui::hex` | Hex encoding/decoding |
| `sui::types` | Type utilities (OTW check) |
| `sui::url` | URL type for metadata |
| `sui::versioned` | Versioned types |

## Transaction Context

Every transaction has an execution context available through `TxContext`. It's created by the system and passed to functions automatically.

### TxContext Structure

```move
module sui::tx_context;

public struct TxContext has drop {
    sender: address,           // Transaction sender
    tx_hash: vector<u8>,       // 32-byte transaction hash
    epoch: u64,                // Current epoch number
    epoch_timestamp_ms: u64,   // Epoch start timestamp
    ids_created: u64,          // Counter for UID generation
}
```

### Reading Context

```move
public fun process(ctx: &TxContext) {
    let sender = ctx.sender();
    let epoch = ctx.epoch();
    let timestamp = ctx.epoch_timestamp_ms();
    let tx_hash = ctx.digest();
    
    // Generate unique addresses
    let unique_addr = ctx.fresh_object_address();
}
```

### Creating Objects

`TxContext` is required for creating UIDs:

```move
public fun new(ctx: &mut TxContext): MyObject {
    MyObject { id: object::new(ctx) }
}

// Must be last parameter in function
public fun create_and_transfer(value: u64, ctx: &mut TxContext) {
    let obj = MyObject { id: object::new(ctx), value };
    transfer::transfer(obj, ctx.sender());
}
```

### Testing with TxContext

```move
#[test]
fun test_with_context() {
    // Simple dummy context
    let ctx = &mut tx_context::dummy();
    assert_eq!(ctx.sender(), @0);
    
    // Custom sender
    let ctx = &mut tx_context::new_from_hint(
        @0xA,   // sender
        42,     // hint for tx_hash
        5,      // epoch
        1000,   // timestamp
        0,      // ids_created
    );
    assert_eq!(ctx.sender(), @0xA);
    assert_eq!(ctx.epoch(), 5);
    
    // Track created objects
    assert_eq!(ctx.ids_created(), 0);
    let obj = MyObject { id: object::new(ctx) };
    assert_eq!(ctx.ids_created(), 1);
}
```

### UID Derivation

Create predictable UIDs for easier off-chain discovery using `sui::derived_object`:

```move
use sui::derived_object;

// Derive UID from parent + key (deterministic)
public fun create_derived(parent: &mut Base, key: address, ctx: &mut TxContext) {
    // Can only be called once per parent + key combination
    let id = derived_object::claim(&mut parent.id, key);
    
    let derived = Derived { id };
    transfer::share_object(derived);
}

// Check if derived UID exists
public fun has_derived<K: copy + drop + store>(parent: &UID, key: K): bool {
    derived_object::exists(parent, key)
}

// Get derived address without creating
public fn get_derived_address<K: copy + drop + store>(parent: ID, key: K): address {
    derived_object::derive_address(parent, key)
}
```

**Use Cases:**
- Predictable object addresses for indexing
- Child objects with known IDs
- Registry patterns

## Epoch and Time

Sui provides two time-related mechanisms: epochs for operational periods and Clock for precise timestamps.

### Epochs

Epochs are ~24-hour operational periods with a fixed validator set:

```move
public fun get_epoch_info(ctx: &TxContext): (u64, u64) {
    (ctx.epoch(), ctx.epoch_timestamp_ms())
}

// Use for epoch-dependent logic
public fun claim_reward(vault: &mut Vault, ctx: &TxContext) {
    assert!(vault.last_claim_epoch < ctx.epoch(), EAlreadyClaimed);
    vault.last_claim_epoch = ctx.epoch();
    // ... distribute reward
}
```

### Clock Object

The `Clock` shared object provides millisecond-precision timestamps:

```move
use sui::clock::Clock;

public fun time_sensitive_action(clock: &Clock, ctx: &mut TxContext) {
    let now = clock.timestamp_ms();
    
    // Check time-based conditions
    assert!(now >= SALE_START_TIME, ESaleNotStarted);
    assert!(now <= SALE_END_TIME, ESaleEnded);
    
    // ... perform action
}
```

**Clock Properties:**
- Located at address `0x6`
- Shared object (accessible by all)
- Cannot be mutated in transactions (read-only)
- Updated during checkpoints by system

### Testing with Clock

```move
#[test]
fun test_with_clock() {
    let ctx = &mut tx_context::dummy();
    let mut clock = clock::create_for_testing(ctx);
    
    assert_eq!(clock.timestamp_ms(), 0);
    
    clock.increment_for_testing(1000);  // Add 1 second
    assert_eq!(clock.timestamp_ms(), 1000);
    
    clock.set_for_testing(5000);  // Set absolute time
    assert_eq!(clock.timestamp_ms(), 5000);
    
    clock.destroy_for_testing();
}
```

## Object Display

Object Display provides a standard way to describe objects for wallets, explorers, and other clients.

### Creating Display

```move
use sui::display;

public struct MY_NFT has drop {}  // OTW

fun init(otw: MY_NFT, ctx: &mut TxContext) {
    let publisher = package::claim(otw, ctx);
    
    // Create Display for NFT type
    let mut display = display::new<NFT>(&publisher, ctx);
    
    // Set display fields
    display.add("name", "My NFT Collection");
    display.add("description", "A unique NFT collection");
    display.add("image_url", "https://example.com/nft.png");
    
    display.update(&publisher);
    
    transfer::public_transfer(publisher, ctx.sender());
    transfer::public_transfer(display, ctx.sender());
}
```

### Template Syntax

Display supports dynamic field interpolation:

```move
public struct NFT has key {
    id: UID,
    token_id: u64,
    rarity: String,
    metadata: Metadata,
}

public struct Metadata has store {
    description: String,
}

// Display templates with field interpolation
display.add("name", "NFT #{token_id}");
display.add("description", "{metadata.description}");
display.add("image_url", "https://example.com/nft/{token_id}.png");
display.add("attributes", "Rarity: {rarity}");
```

**Template Features:**
- `{field}` - Access top-level field
- `{nested.field}` - Access nested struct fields
- Supports all primitive types and strings

### Standard Display Fields

| Field | Description |
|-------|-------------|
| `name` | Object name |
| `description` | Object description |
| `image_url` | Image URL or blob |
| `thumbnail_url` | Thumbnail for previews |
| `project_url` | Project website |
| `creator` | Creator identifier |

### Multiple Display Objects

Multiple `Display<T>` objects can exist for the same type. The most recently updated one is used by full nodes.

## BCS Serialization

Binary Canonical Serialization (BCS) is Move's standard binary format for structured data.

### Encoding

```move
use sui::bcs;

// Encode any type to bytes
let num: u64 = 42;
let bytes = bcs::to_bytes(&num);

// Encode structs
public struct User has store {
    id: u64,
    name: String,
}

let user = User { id: 1, name: b"Alice".to_string() };
let encoded = bcs::to_bytes(&user);
```

### Decoding

BCS decoding uses a wrapper API with `peel_*` methods:

```move
public fun decode_user(bytes: vector<u8>): User {
    let mut bcs = bcs::new(bytes);
    
    // Peel fields in order they were encoded
    let id = bcs.peel_u64();
    let name = bcs.peel_string();
    
    // Check no remaining bytes
    bcs.into_remainder_bytes().destroy();
    
    User { id, name }
}
```

### Decoding Vectors

```move
public fun decode_vector(bytes: vector<u8>): vector<u64> {
    let mut bcs = bcs::new(bytes);
    
    // First, peel the length
    let len = bcs.peel_u64();
    
    // Then peel each element
    let mut result = vector[];
    let mut i = 0;
    while (i < len) {
        result.push_back(bcs.peel_u64());
        i = i + 1;
    };
    
    bcs.into_remainder_bytes().destroy();
    result
}

// Or use macro
let nums = bcs.peel_vec!(|bcs| bcs.peel_u64());
```

### Decoding Options

```move
// Option is encoded as 0 (None) or 1 (Some)
let mut bcs = bcs::new(bytes);
let opt = bcs.peel_option!(|bcs| bcs.peel_u64());
```

### Common Patterns

```move
// Chain multiple decodes
let (id, name, active) = (
    bcs.peel_u64(),
    bcs.peel_string(),
    bcs.peel_bool(),
);

// Decode nested struct
public fun decode_nested(bytes: vector<u8>): Outer {
    let mut bcs = bcs::new(bytes);
    Outer {
        id: bcs.peel_u64(),
        inner: Inner {
            value: bcs.peel_u64(),
            flag: bcs.peel_bool(),
        },
    }
}
```

### Use Cases

- **Off-chain data parsing**: Decode on-chain data in SDKs
- **Cross-chain communication**: Serialize data for bridges
- **Message verification**: Verify signed messages
- **Testing**: Create test fixtures from encoded data

## Data Storage Patterns

### Choosing the Right Storage

```
How much data? 
├── Small (< 100KB)
│   └── Object fields
│
└── Large or Variable
    ├── What type?
    │   ├── Single type, known structure → Table<K, V>
    │   ├── Mixed types → Bag
    │   └── Objects needing ID lookup → ObjectTable / ObjectBag
    │
    └── Need ordering?
        └── LinkedTable<K, V>
```

### Storage Comparison

| Storage Type | Max Size | Type Safety | Gas Cost | Ordered |
|--------------|----------|-------------|----------|---------|
| Object fields | ~256KB | ✓ High | Low | N/A |
| VecMap | Object limit | ✓ High | Low | No |
| VecSet | Object limit | ✓ High | Low | No |
| Table | Unlimited* | ✓ High | Medium | No |
| Bag | Unlimited* | Low | Medium | No |
| LinkedTable | Unlimited* | ✓ High | Medium | Yes |
| Dynamic fields | Unlimited* | Low | High | No |

*Per-field limit removed, but 1000 fields/tx creation limit

### Collection Usage Examples

```move
// VecMap - small, in-object storage
public struct Cache has key {
    id: UID,
    entries: VecMap<String, u64>,  // Must fit in object
}

// Table - large, dynamic key-value
let users = table::new<address, UserProfile>(ctx);
users.add(addr, profile);

// Bag - heterogeneous data
let metadata = bag::new(ctx);
metadata.add(b"name", b"NFT #1".to_string());
metadata.add(b"rarity", 5u8);
metadata.add(b"attributes", attributes);

// LinkedTable - ordered access
let queue = linked_table::new<ID, Task>(ctx);
queue.push_back(id1, task1);
let (first_id, first_task) = queue.pop_front();
```

### VecSet Usage

```move
use sui::vec_set::{Self, VecSet};

// Create and populate
let mut set = vec_set::empty<address>();
set.insert(@0xA);
set.insert(@0xB);
set.insert(@0xA);  // Would abort - duplicates not allowed

// Check membership
assert!(set.contains(@0xA));

// Remove
set.remove(@0xA);
```

### VecMap Usage

```move
use sui::vec_map::{Self, VecMap};

let mut map = vec_map::empty<String, u64>();
map.insert(b"score".to_string(), 100);
map.insert(b"level".to_string(), 5);

// Access
let score = map.get(&b"score".to_string());  // Returns Option<&u64>
let score = &map[&b"score".to_string()];      // Index syntax

// Update
map.insert(b"score".to_string(), 200);  // Overwrites

// Remove
let old_value = map.remove(&b"score".to_string());
```

### LinkedTable Operations

```move
use sui::linked_table::{Self, LinkedTable};

let mut table = linked_table::new<u64, Task>(ctx);

// Add to front/back
table.push_front(1, task1);
table.push_back(2, task2);

// Access ordered
let front_id = table.front();  // Option<&K>
let back_id = table.back();    // Option<&K>

// Remove from ends
let (id, task) = table.pop_front();
let (id, task) = table.pop_back();

// Iterate (via indices)
let mut id = table.front();
while (option::is_some(&id)) {
    let k = option::borrow(&id);
    let task = table.borrow(k);
    // process task
    id = table.next(k);
};
```

### Collection Limitations

```move
// VecMap/VecSet: Cannot compare for equality (order not guaranteed)
let map1 = vec_map::empty<u64, u64>();
let map2 = vec_map::empty<u64, u64>();
// assert!(map1 == map2);  // WARNING: May fail unexpectedly

// All collections: Size limited by object size (~256KB for in-object)
// Table/Bag: No size limit (dynamic fields)

// VecMap: Insertion order affects internal structure
// Use Table if consistent ordering needed
```

### Dynamic Fields Best Practices

```move
// ✓ GOOD: Remove before deleting parent
public fun delete_container(c: Container) {
    let Container { id } = c;
    // Clean up dynamic fields first!
    if (dynamic_field::exists_(&id, b"config")) {
        dynamic_field::remove(&mut id, b"config");
    };
    id.delete();
}

// ❌ BAD: Orphaned fields waste storage
public fun delete_container_bad(c: Container) {
    let Container { id } = c;
    id.delete();  // Dynamic fields become orphaned!
}

// Use custom types for namespacing
public struct ConfigKey has copy, drop, store {}
public struct MetadataKey has copy, drop, store {}

// Safer than raw strings
dynamic_field::add(&mut obj.id, ConfigKey {}, config);
dynamic_field::add(&mut obj.id, MetadataKey {}, metadata);
```

## Error Handling

### Error Constant Conventions

```move
// Error constants use E prefix + PascalCase
const EInvalidAmount: u64 = 1;
const EInsufficientBalance: u64 = 2;
const ENotAuthorized: u64 = 3;
const EAlreadyInitialized: u64 = 4;

// Regular constants use ALL_CAPS
const MAX_SUPPLY: u64 = 1_000_000;
const DECIMALS: u8 = 9;

// Move 2024: Descriptive error messages
#[error]
const EInvalidAmount: vector<u8> = b"Amount must be greater than zero";

#[error]
const EInsufficientBalance: vector<u8> = b"Insufficient balance for transfer";
```

### Error Handling Rules

**Rule 1: Provide check functions**

```move
// ✓ GOOD: Boolean check lets caller control error
public fun can_transfer(account: &Account, amount: u64): bool {
    account.balance >= amount
}

public fun transfer(account: &mut Account, amount: u64) {
    assert!(can_transfer(account, amount), EInsufficientBalance);
    // ...
}
```

**Rule 2: Use different error codes**

```move
// ✓ GOOD: Each case has unique code
public fun process(a: u64, b: u64) {
    assert!(a > 0, EInvalidA);
    assert!(b > 0, EInvalidB);
    assert!(a != b, EValuesEqual);
    // ...
}
```

**Rule 3: Return bool instead of assert helper**

```move
// ❌ BAD: Assert function removes control
public fun assert_authorized(addr: address) {
    assert!(is_admin(addr), ENotAuthorized);
}

// ✓ GOOD: Boolean function for flexibility
public fun is_authorized(addr: address): bool {
    /* check logic */
}

// Caller decides error handling
public fun admin_action(addr: address) {
    assert!(is_authorized(addr), ENotAuthorized);
}
```

### No Catch Mechanism

Transactions are atomic - they succeed completely or fail entirely:

```move
// Transaction aborts, ALL changes reverted
public fun transfer(from: &mut Account, to: &mut Account, amount: u64) {
    assert!(from.balance >= amount, EInsufficientBalance);
    from.balance = from.balance - amount;
    // If anything fails after this point, 
    // from.balance is restored to original value
    to.balance = to.balance + amount;
}
```

## Events

### Event Definition

```move
// Events: copy + drop, named in past tense
public struct TransferEvent has copy, drop {
    from: address,
    to: address,
    amount: u64,
    coin_type: String,  // Useful for indexing
}

public struct MintEvent has copy, drop {
    recipient: address,
    amount: u64,
    total_supply: u64,
}
```

### Event Emission

```move
public fun transfer(
    coin: Coin<T>, 
    to: address, 
    ctx: &mut TxContext
) {
    let from = ctx.sender();
    let amount = coin.value();
    
    transfer::public_transfer(coin, to);
    
    // Emit after successful transfer
    event::emit(TransferEvent {
        from,
        to,
        amount,
        coin_type: type_name::get<T>().into_string(),
    });
}
```

### Event Best Practices

```move
// ✓ Include relevant IDs for indexing
public struct ListEvent has copy, drop {
    listing_id: ID,      // Index by listing
    nft_id: ID,          // Index by NFT
    seller: address,     // Index by seller
    price: u64,
}

// ✓ Don't include sender/timestamp (auto-added)
// ❌ BAD
public struct BadEvent has copy, drop {
    sender: address,      // Redundant
    timestamp: u64,       // Redundant
    value: u64,
}

// ✓ GOOD
public struct GoodEvent has copy, drop {
    value: u64,
}

// ✓ Type must be internal to emitting module
// This FAILS:
public fun emit_external() {
    event::emit(std::type_name::get<MyType>()); // Error!
}
```

## Testing

### Characteristics of Good Tests

1. **Concise**: Each test should be short and focused on a single behavior
2. **Readable**: Tests serve as documentation - anyone should understand what's being tested
3. **Isolated**: Test one thing per test function for easier debugging
4. **Descriptive**: Use descriptive names like `transfer_succeeds_with_sufficient_balance`

```move
// ✓ GOOD: Following AAA pattern (Arrange, Act, Assert)
#[test]
fun test_add_increases_balance_by_specified_amount() {
    // Arrange: set up initial state
    let mut balance = balance::new(100);
    
    // Act: perform the operation being tested
    balance.add(50);
    
    // Assert: verify the expected outcome
    assert_eq!(balance.value(), 150);
}
```

### Test Annotations

```move
// Basic test
#[test]
fun test_basic() { /* ... */ }

// Expected failure
#[test, expected_failure]
fun test_aborts() { abort 1 }

// Expected failure with specific code
#[test, expected_failure(abort_code = EInvalidInput)]
fun test_invalid_input() { validate(0); }

// Expected failure at specific location
#[test, expected_failure(abort_code = 1, location = Self)]
fun test_local_abort() { abort 1 }

// Test-only code
#[test_only]
const TEST_ADDRESS: address = @0xCAFE;

#[test_only]
public fun setup_test(): TestState { /* ... */ }

#[test_only]
module my_app::test_helpers;
```

### Random Input Testing

```move
// Random primitive inputs
#[random_test]
fun test_safe_add_never_overflows(a: u64, b: u64) {
    let result = safe_add(a, b);
    assert!(result >= a && result >= b);
}

// Random vectors
#[random_test]
fun test_vector_length(v: vector<u8>) {
    assert!(v.length() >= 0);
}
```

**Supported Random Types:**

| Type | Range |
|------|-------|
| `u8`, `u16`, `u32`, `u64`, `u128`, `u256` | Full range |
| `bool` | true/false |
| `address` | Random 32-byte address |
| `vector<T>` | Random length, T must be primitive |

**Controlling Random Tests:**

```bash
# Number of iterations (default: multiple)
sui move test --rand-num-iters 100

# Reproduce failure with seed
sui move test test_name --seed 2033439370411573084
```

**Best Practices for Random Tests:**
- Constrain large integers: use smaller types and cast
- Avoid unbounded vectors: construct manually from primitives
- Use `assert_eq!` for better failure diagnostics
- Complement targeted tests, don't replace them

### Test Utilities

```move
use std::unit_test::{assert_eq, assert_ref_eq, destroy};

// Better diagnostics with assert_eq
assert_eq!(result, expected);  // Prints both values on failure

// Compare by reference
assert_ref_eq!(&user, &expected_user);

// Destroy any value (black hole)
destroy(non_droppable_value);

// No abort code needed in tests
assert!(condition);  // ✓ GOOD in tests
assert!(condition, 0);  // ❌ BAD - may conflict with app codes

// Dummy context
let ctx = &mut tx_context::dummy();

// Custom context
let ctx = &mut tx_context::new_from_hint(
    @0xA,    // sender
    42,      // unique hint
    5,       // epoch
    1000,    // timestamp ms
    0,       // ids created
);

// Full context control
let ctx = &mut tx_context::create(
    @0xA,                    // sender
    tx_context::dummy_tx_hash_with_hint(1),
    10,                      // epoch
    1700000000000,           // timestamp
    0,                       // ids_created
    1000,                    // reference_gas_price
    1500,                    // gas_price
    10_000_000,              // gas_budget
    option::none(),          // sponsor
);
```

### Test Scenario (Multi-Transaction)

```move
use sui::test_scenario;

#[test]
fun test_transfer_flow() {
    let admin = @0xAD;
    let alice = @0xA;
    let bob = @0xB;
    
    // Start scenario
    let mut scenario = test_scenario::begin(admin);
    
    // Transaction 1: Admin creates
    {
        let token = mint(1000, scenario.ctx());
        transfer::public_transfer(token, alice);
    };
    
    // Transaction 2: Alice transfers to Bob
    scenario.next_tx(alice);
    {
        let token = scenario.take_from_sender<Token>();
        assert_eq!(token.value(), 1000);
        transfer::public_transfer(token, bob);
    };
    
    // Transaction 3: Bob receives
    scenario.next_tx(bob);
    {
        assert!(scenario.has_most_recent_for_sender<Token>());
        let token = scenario.take_from_sender<Token>();
        assert_eq!(token.value(), 1000);
        scenario.return_to_sender(token);
    };
    
    scenario.end();
}
```

**Test Scenario API:**

| Function | Purpose |
|----------|---------|
| `begin(sender)` | Start scenario |
| `end()` | End and get effects |
| `next_tx(sender)` | Advance to next transaction |
| `ctx()` | Get mutable TxContext |
| `take_from_sender<T>()` | Take owned object |
| `take_from_sender_by_id<T>(id)` | Take specific object |
| `return_to_sender(obj)` | Return object |
| `take_shared<T>()` | Take shared object |
| `return_shared(obj)` | Return shared object |
| `take_immutable<T>()` | Take immutable object |
| `return_immutable(obj)` | Return immutable object |
| `has_most_recent_for_sender<T>()` | Check object exists |

**Epoch Advancement in Scenarios:**

```move
#[test]
fun test_epoch_advancement() {
    let mut scenario = test_scenario::begin(@0xA);
    
    assert_eq!(scenario.ctx().epoch(), 0);
    
    // Advance epoch
    scenario.next_epoch(@0xA);
    assert_eq!(scenario.ctx().epoch(), 1);
    
    // Advance with time (milliseconds)
    scenario.later_epoch(86400000, @0xA);  // 1 day
    assert_eq!(scenario.ctx().epoch(), 2);
    assert_eq!(scenario.ctx().epoch_timestamp_ms(), 86400000);
    
    scenario.end();
}
```

**Reading Transaction Effects:**

```move
#[test]
fun test_effects() {
    let mut scenario = test_scenario::begin(@0xA);
    
    // Create objects
    {
        let obj = create(scenario.ctx());
        transfer::public_transfer(obj, @0xB);
    };
    
    let effects = scenario.next_tx(@0xA);
    
    // Check what happened
    assert_eq!(effects.created().length(), 1);
    assert_eq!(effects.transferred_to_account().size(), 1);
    assert_eq!(effects.num_user_events(), 0);
    
    scenario.end();
}
```

**Effects API:**

| Method | Returns | Description |
|--------|---------|-------------|
| `created()` | `vector<ID>` | Objects created |
| `written()` | `vector<ID>` | Objects modified |
| `deleted()` | `vector<ID>` | Objects deleted |
| `transferred_to_account()` | `VecMap<ID, address>` | Transfers to addresses |
| `transferred_to_object()` | `VecMap<ID, ID>` | Transfers to objects |
| `shared()` | `vector<ID>` | Objects shared |
| `frozen()` | `vector<ID>` | Objects frozen |
| `num_user_events()` | `u64` | Events emitted |

### Testing with System Objects

```move
use sui::clock::Clock;
use sui::random::{Self, Random, RandomGenerator};
use sui::deny_list::DenyList;

#[test]
fun test_with_clock() {
    let mut scenario = test_scenario::begin(@0xA);
    
    // Create system objects (Clock, Random, DenyList)
    scenario.create_system_objects();
    scenario.next_tx(@0xA);
    
    {
        let clock = scenario.take_shared<Clock>();
        assert!(clock.timestamp_ms() >= 0);
        test_scenario::return_shared(clock);
    };
    
    scenario.end();
}

// Direct clock manipulation
#[test]
fun test_time_dependent() {
    let mut clock = clock::create_for_testing(&mut tx_context::dummy());
    assert_eq!(clock.timestamp_ms(), 0);
    
    clock.increment_for_testing(1000);
    assert_eq!(clock.timestamp_ms(), 1000);
    
    clock.set_for_testing(5000);
    assert_eq!(clock.timestamp_ms(), 5000);
    
    clock.destroy_for_testing();
}

// Random testing
#[test]
fun test_random() {
    // Simple generator (deterministic)
    let mut gen = random::new_generator_for_testing();
    let value: bool = gen.generate_bool();
    
    // Seeded generator (reproducible)
    let mut gen = random::new_generator_from_seed_for_testing(b"my seed");
    let value: u64 = gen.generate_u64();
}

// Deny list testing
#[test]
fun test_deny_list() {
    let mut scenario = test_scenario::begin(@0xA);
    deny_list::create_for_testing(scenario.ctx());
    
    scenario.next_tx(@0xA);
    {
        let deny_list = scenario.take_shared<DenyList>();
        // ... test deny list
        test_scenario::return_shared(deny_list);
    };
    
    scenario.end();
}

// Coin/Balance testing
#[test]
fun test_coins() {
    let ctx = &mut tx_context::dummy();
    
    // Mint test coin
    let coin = coin::mint_for_testing<SUI>(1000, ctx);
    assert_eq!(coin.value(), 1000);
    
    // Burn and get value back
    let value = coin.burn_for_testing();
    assert_eq!(value, 1000);
    
    // Create balance directly
    let balance = balance::create_for_testing<SUI>(500);
    let value = balance.destroy_for_testing();
    assert_eq!(value, 500);
}
```

**System Objects Summary:**

| Object | Creation | Test-only Features |
|--------|----------|-------------------|
| `Clock` | `clock::create_for_testing(ctx)` | `increment_for_testing`, `set_for_testing` |
| `Random` | `random::create_for_testing(ctx)` | `update_randomness_state_for_testing` |
| `RandomGenerator` | `random::new_generator_for_testing()` | `new_generator_from_seed_for_testing` |
| `DenyList` | `deny_list::create_for_testing(ctx)` | `new_for_testing` |
| `Coin<T>` | `coin::mint_for_testing<T>(value, ctx)` | `burn_for_testing` |
| `Balance<T>` | `balance::create_for_testing<T>(value)` | `destroy_for_testing` |

### Module Extensions

Extend foreign modules for testing:

```move
// tests/extensions/pyth_ext.move
#[test_only]
extend module pyth::price_info;

public fun new_price_info_for_testing(
    price: u64,
    ctx: &mut TxContext,
): PriceInfoObject {
    PriceInfoObject {
        id: object::new(ctx),
        price_info: PriceInfo { price },
    }
}
```

**Extension Rules:**
- Must have mode attribute (`#[test_only]`)
- Additive only (cannot modify existing items)
- Root package only (dependency extensions ignored)
- Edition 2024.alpha or later required

### Coverage Reports

```bash
# Run with coverage
sui move test --coverage

# Summary
sui move coverage summary

# Per-function coverage
sui move coverage summary --summarize-functions

# CSV output
sui move coverage summary --csv

# Source-level coverage
sui move coverage source --module my_module

# LCOV for CI/external tools
sui move test --coverage --trace
sui move coverage lcov

# Differential coverage (lines covered only by specific test)
sui move coverage lcov --differential-test test_name

# Single test coverage
sui move coverage lcov --only-test test_name
```

### Gas Profiling

```bash
# Show gas per test
sui move test -s

# CSV output
sui move test -s csv

# Gas limit (timeout if exceeded)
sui move test -i 100  # Max 100 gas units

# Trace analysis with speedscope
sui move test --trace
sui analyze-trace -p traces/<file> gas-profile
speedscope gas_profile_<file>.json
```

**Comparing Implementations:**

```move
// Two ways to sum 1..n
public fun sum_loop(n: u64): u64 { /* iterative */ }
public fun sum_formula(n: u64): u64 { n * (n - 1) / 2 }

#[test]
fun test_sum_loop() { assert_eq!(sum_loop(100), 4950); }

#[test]  
fun test_sum_formula() { assert_eq!(sum_formula(100), 4950); }

// Run: sui move test -s
// Results show formula is ~60x more gas efficient
```

### Testing Anti-Patterns

```move
// ❌ BAD: Separate test and expected_failure
#[test]
#[expected_failure]
fun test_abort() {}

// ✓ GOOD
#[test, expected_failure]
fun test_abort() {}

// ❌ BAD: Cleaning up in expected_failure test
#[test, expected_failure(abort_code = EError)]
fun test_fail() {
    let test = test_scenario::begin(@0);
    my_app::call(test.ctx());
    test.end();  // Won't reach here!
}

// ✓ GOOD: Clearly show where failure expected
#[test, expected_failure(abort_code = EError)]
fun test_fail() {
    let test = test_scenario::begin(@0);
    my_app::call(test.ctx());
    abort  // Marks expected failure point
}

// ❌ BAD: TestScenario when not needed
let test = test_scenario::begin(@0);
let obj = create(test.ctx());
destroy(obj);
test.end();

// ✓ GOOD: Use dummy context
let ctx = &mut tx_context::dummy();
destroy(create(ctx));

// ❌ BAD: Test prefix in test module
module my_app::my_module_tests;
#[test]
fun test_feature() {}

// ✓ GOOD: Descriptive name
module my_app::my_module_tests;
#[test]
fun feature_works_as_expected() {}

// ❌ BAD: Using abort codes in test asserts
assert!(result == expected, 0);

// ✓ GOOD: No code needed
assert!(result == expected);
assert_eq!(result, expected);

// ❌ BAD: Using *_for_testing for cleanup
nft.destroy_for_testing();

// ✓ GOOD: Use destroy black hole
destroy(nft);
```

## Security Considerations

### Access Control Patterns

```move
// ✓ GOOD: Capability-based (flexible, upgrade-safe)
public fun mint(_: &MinterCap, amount: u64, ctx: &mut TxContext): Coin<T> {
    // Only MinterCap owner can call
}

// ⚠ USE CAREFULLY: Address-based (rigid, upgrade issues)
public fun mint(ctx: &mut TxContext): Coin<T> {
    assert!(ctx.sender() == @admin, ENotAuthorized);
}

// ✓ GOOD: Publisher-based (for framework operations)
public fn create_display(publisher: &Publisher, ctx: &mut TxContext) {
    assert!(publisher.from_module("my_nft"), ENotAuthorized);
}
```

### Upgrade Security

```move
// init only runs on first publish, not upgrades
fun init(ctx: &mut TxContext) {
    // Creates initial capabilities
}

// New functions can be added in upgrades
public fn new_feature() {}  // Added in v2

// Capabilities created in init are still valid after upgrade
// Consider: Can new functions bypass capability checks?
```

### Shared Object Considerations

```move
// Shared objects require consensus (slower)
// Use when truly needed (marketplaces, pools)

// ✓ GOOD: Single owner when possible
public fun transfer(item: Item, to: address) {
    transfer::public_transfer(item, to);
}

// ✓ GOOD: Shared when multiple actors needed
public fun list(shared: &mut Marketplace, item: Item) {
    shared.listings.push_back(item);
}
```

### Type Safety

```move
// Phantom types prevent mixing
public struct Coin<phantom T> has key, store { id: UID, value: u64 }

public struct USD {}
public struct BTC {}

// Compile error: can't mix coin types
fn wrong(c: Coin<USD>, expected: Coin<BTC>) { /* ... */ }
```

## Performance & Gas

### Network Limits

| Limit | Value | Notes |
|-------|-------|-------|
| Transaction size | 128 KB | Includes signature + metadata |
| Object size | 256 KB | Use dynamic fields for larger |
| Single pure argument | 16 KB | ~500 addresses max; join vectors dynamically |
| Objects created/tx | 2048 | |
| Dynamic fields created/tx | 1000 | Key AND value both count as objects |
| Dynamic fields accessed/tx | 1000 | |
| Events emitted/tx | 1024 | |

**Workarounds for Limits:**

```move
// Bypass 16KB pure argument limit by joining vectors
public fun join_addresses(a: vector<address>, b: vector<address>): vector<address> {
    let mut result = a;
    vector::append(&mut result, b);
    result
}

// Use dynamic fields for large data (>256KB)
public struct LargeData has store { /* ... */ }

public fun store_large(parent: &mut UID, data: LargeData) {
    dynamic_field::add(parent, b"large_data", data);
}
```

### Optimization Strategies

1. **Minimize shared object usage**
   - Single-owner: Fast path (no consensus)
   - Shared: Consensus required

2. **Batch operations**
   ```move
   // ❌ BAD: Multiple transactions
   for item in items { transfer(item, recipient); }
   
   // ✓ GOOD: Single transaction
   public fun transfer_all(items: vector<Item>, recipient: address) {
       items.do!(|item| transfer::public_transfer(item, recipient));
   }
   ```

3. **Use appropriate collections**
   - VecMap/Table: Typed, efficient
   - Bag: Heterogeneous, more overhead
   - Dynamic fields: Maximum flexibility, highest cost

4. **Avoid deep nesting**
   - Each nested access adds gas

5. **Lazy initialization**
   ```move
   // Only create what's needed
   public fun get_or_create_config(parent: &mut Parent, ctx: &mut TxContext): &mut Config {
       if (!dynamic_field::exists_(&parent.id, ConfigKey {})) {
           dynamic_field::add(&mut parent.id, ConfigKey {}, Config::default(ctx));
       };
       dynamic_field::borrow_mut(&mut parent.id, ConfigKey {})
   }
   ```

## Upgradeability

### What Can Change in Upgrades

| Element | Can Remove? | Can Modify? | Notes |
|---------|-------------|-------------|-------|
| Module | ✗ | Implementation only | Must keep all modules |
| Public struct | ✗ | ✗ | Signature frozen |
| Public function | ✗ | Implementation only | Signature frozen |
| Private function | ✓ | ✓ | Internal only |
| `public(package)` function | ✓ | ✓ | Package-internal |
| Entry function | ✓ | ✓ | Not called from packages |

### Versioned State Pattern

```move
const VERSION: u16 = 2;

public struct SharedState has key {
    id: UID,
    version: u16,
    // ... state fields
}

public fun mutate(state: &mut SharedState) {
    assert!(state.version == VERSION, EVersionMismatch);
    // ... new logic
}

// Admin upgrades version after code upgrade
public fn upgrade_version(cap: &AdminCap, state: &mut SharedState) {
    assert!(state.version < VERSION, EAlreadyUpgraded);
    state.version = VERSION;
}
```

### Versioned Config Pattern

```move
public struct Config has key {
    id: UID,
    version: u16,
}

// V1 config structure
public struct ConfigV1 has store {
    threshold: u64,
}

// V2 config structure (added in upgrade)
public struct ConfigV2 has store {
    threshold: u64,
    new_field: bool,
}

public fun get_config(config: &Config): &ConfigV2 {
    assert!(config.version >= 2, EOldVersion);
    dynamic_field::borrow(&config.id, ConfigKey {})
}
```

## Code Quality Checklist

### Package Manifest

- [ ] Use `edition = "2024"` or `"2024.beta"`
- [ ] Prefix named addresses (e.g., `my_protocol_math`)
- [ ] Remove explicit framework dependencies (auto-included since CLI 1.45)

### Code Style

- [ ] Module label syntax (not block syntax)
- [ ] Group imports with `Self`
- [ ] No redundant `{Self}` alone in imports
- [ ] Error constants: `EPascalCase`
- [ ] Regular constants: `ALL_CAPS`

### Structs

- [ ] Capabilities have `Cap` suffix
- [ ] Events named in past tense
- [ ] No "Potato" in hot potato names
- [ ] Dynamic field keys: positional + `Key` suffix

### Functions

- [ ] Use `public` or `entry`, not both
- [ ] Write composable functions
- [ ] Objects first, capabilities second
- [ ] Getters named after field (no `get_` prefix)

### Modern Syntax

- [ ] Use `.to_string()` not `utf8()`
- [ ] Vector literals `vector[1, 2, 3]`
- [ ] Index syntax `map[&key]`
- [ ] Macros: `.do!()`, `.fold!()`, `.filter!()`
- [ ] `ctx.sender()` instead of `tx_context::sender(ctx)`
- [ ] `id.delete()` instead of `object::delete(id)`
- [ ] Method chaining: `coin.split(amount, ctx).into_balance()`

### Other Style

- [ ] Spread in unpack: `let MyStruct { id, .. } = value;`
- [ ] Doc comments start with `///`
- [ ] Leave comments `//` for complex logic

### Testing

- [ ] Combine `#[test, expected_failure]`
- [ ] No cleanup in expected_failure tests
- [ ] Use `assert_eq!` for diagnostics
- [ ] Use `destroy` instead of `*_for_testing()`
- [ ] Use `tx_context::dummy()` when TestScenario not needed
- [ ] No `test_` prefix in test module (module name already ends with `_tests`)
- [ ] No abort codes in test `assert!` (may conflict with app codes)

## CLI Reference

### Package Commands

```bash
# Create new package
sui move new <package_name>

# Build package
sui move build

# Run tests
sui move test
sui move test <filter>
sui move test --coverage
sui move test -s              # Gas statistics

# Migrate to new edition
sui move migrate

# Coverage commands
sui move coverage summary
sui move coverage summary --summarize-functions
sui move coverage source --module <name>
sui move coverage lcov
```

### Client Commands

```bash
# Publish package
sui client publish --gas-budget <budget>

# Upgrade package  
sui client upgrade --gas-budget <budget>

# Object operations
sui client object <object_id>
sui client objects <address>

# Transaction operations
sui client tx-block <digest>
sui client effects <digest>
```

### Debug Commands

```bash
# Trace analysis
sui move test --trace
sui analyze-trace -p traces/<file> gas-profile

# View bytecode
sui move coverage bytecode --module <name>
```

## Move 2024 Migration

### Key Changes

| Feature | Move 2020 | Move 2024 |
|---------|-----------|-----------|
| Mutable variables | `let x = 1; x = 2;` | `let mut x = 1; x = 2;` |
| Friend visibility | `friend` + `public(friend)` | `public(package)` |
| Struct visibility | `struct S {}` | `public struct S {}` |
| Module syntax | `module m { }` | `module m;` |

### Automatic Migration

```bash
sui move migrate
```

### Manual Migration Checklist

```move
// 1. Mutable bindings
let mut counter = 0;
counter = counter + 1;

// 2. Replace friend with public(package)
// Before:
friend my_package::helper;
public(friend) fn internal() {}

// After:
public(package) fn internal() {}

// 3. Add public to structs
public struct MyStruct {}

// 4. Use module label syntax
module my_package::my_module;
// ... content without braces
```

### New Features

```move
// Method syntax
let value = obj.method();
let count = vec.length();

// Index syntax
let item = &map[&key];
let item_mut = &mut map[&key];

// Method chaining
let balance = coin.split(amount, ctx).into_balance();

// Spread in destructuring
let MyStruct { id, .. } = obj;  // Ignore other fields
```

## Further Resources

- **Official Documentation**: [https://docs.sui.io](https://docs.sui.io)
- **Move Reference**: [https://move-book.com/reference](https://move-book.com/reference)
- **Sui Framework Source**: [GitHub - Sui Framework](https://github.com/MystenLabs/sui/tree/main/crates/sui-framework/packages/sui-framework/sources)
- **Sui by Example**: [https://examples.sui.io](https://examples.sui.io)
- **Move Standard Library**: [GitHub - Move Stdlib](https://github.com/MystenLabs/sui/tree/main/crates/sui-framework/packages/move-stdlib/sources)
