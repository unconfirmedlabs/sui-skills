---
name: sui-derived-objects
description: "Sui Derived Objects — deterministic object IDs derived from a parent UID and a key. Covers the Move module (sui::derived_object), ObjectKey struct patterns, client-side derivation with deriveObjectID, and real-world usage patterns."
---

# Sui Derived Objects

Derived objects have deterministic addresses computed from a parent object's UID and a key. This enables offline ID computation, registry patterns without bottlenecks, and client-side discoverability.

## Core Concept

A derived object's address = `hash(parent_id, DerivedObjectKey(key))`. Given the same parent and key, you always get the same address — on-chain and off-chain.

After creation, derived objects are **independent** of their parent. No sequencing on the parent is required to use them. The parent only tracks which keys have been claimed.

## Move API (`sui::derived_object`)

```move
use sui::derived_object::{Self, claim, derive_address};
```

### Functions

| Function | Signature | Purpose |
|----------|-----------|---------|
| `claim` | `claim<K: copy + drop + store>(parent: &mut UID, key: K): UID` | Create a derived UID. Aborts if already claimed. |
| `exists` | `exists<K: copy + drop + store>(parent: &UID, key: K): bool` | Check if a key has been claimed. Returns `true` even after UID deletion. |
| `derive_address` | `derive_address<K: copy + drop + store>(parent: ID, key: K): address` | Compute the derived address without claiming. |

### Key Constraint

Keys must have `copy + drop + store`. This is satisfied by tuple structs containing primitives, `String`, `TypeName`, `ID`, `address`, `vector<u8>`, etc.

---

## ObjectKey Patterns

The derivation key is a struct you define in your module. It determines how many derived objects a parent can produce and what identifies each one.

### Pattern 1: Unit Struct Key (one-to-one)

Derives exactly **one** object per parent. Used for admin capabilities, metadata singletons, or any 1:1 relationship.

```move
/// Key for deriving an admin cap from the parent object.
public struct AdminCapKey() has copy, drop, store;

public struct AdminCap has key, store {
    id: UID,
}

// In the creation function:
let admin_cap = AdminCap {
    id: claim(&mut parent.id, AdminCapKey()),
};
```

The unit struct has no fields, so BCS encodes as a single `0x00` byte (the compiler injects `dummy_field: bool = false`).

### Pattern 2: Value Key (one-to-many)

Derives **multiple** objects from one parent, each identified by a value. Used for registries, collections, and indexed children.

**Address key** — one per wallet:

```move
public struct ProfileKey(address) has copy, drop, store;

let profile = Profile {
    id: derived_object::claim(&mut registry.id, ProfileKey(ctx.sender())),
    // ...
};
```

**Integer key** — sequential editions:

```move
public struct EditionKey(u16) has copy, drop, store;

// Verify sequential ordering before claiming
if (edition > 0) {
    assert!(
        derived_object::exists(parent.uid(), EditionKey(edition - 1)),
        ENotSequentialEdition,
    );
};

let child = Child {
    id: derived_object::claim(parent.uid_mut(cap), EditionKey(edition)),
    // ...
};
```

**String key** — named entries:

```move
public struct CategoryKey(String) has copy, drop, store;

let category = Category {
    id: claim(&mut registry.id, CategoryKey(name)),
    name,
};
```

**Digest key** — content-addressed:

```move
public struct ContentKey(vector<u8>) has copy, drop, store;

// Hash the content to produce a deterministic key
fun calculate_digest(ids: vector<ID>, values: vector<u64>, nonce: u256): vector<u8> {
    let mut hash_input = vector<u8>[];
    hash_input.append(to_bytes(&ids));
    hash_input.append(to_bytes(&values));
    hash_input.append(to_bytes(&nonce));
    blake2b256(&hash_input)
}

let child = Child {
    id: claim(&mut registry.id, ContentKey(digest)),
    // ...
};
```

**TypeName key** — one per type:

```move
public struct ChildKey(TypeName) has copy, drop, store;

let child = Child {
    id: claim(parent.uid_mut(), ChildKey(type_name::get<T>())),
    // ...
};
```

### Pattern 3: Phantom-Typed Key (type-namespaced)

Uses a phantom type parameter on the key struct to namespace derivations by type. Useful when the same parent needs derived objects for different currency types or token types.

```move
public struct VaultKey<phantom Currency>(u64) has copy, drop, store;

public fun new<Currency>(parent: &mut UID, idx: u64): Vault<Currency> {
    Vault {
        id: claim(parent, VaultKey<Currency>(idx)),
        balance: balance::zero(),
    }
}
```

The phantom type is included in the type hash, so `VaultKey<SUI>(0)` and `VaultKey<USDC>(0)` produce different addresses from the same parent.

### Pattern 4: Complex Key (runtime-flexible)

Uses `TypeName` values instead of phantom types for runtime flexibility. Essential when you need to iterate over heterogeneous pools or derive addresses from stored type information.

```move
public struct PoolKey(TypeName, TypeName, Option<TypeName>) has copy, drop, store;

public fun new<Share, Currency>(
    parent: &mut UID,
    kind: PoolKind,
): Pool<Share, Currency> {
    let key = PoolKey(
        with_defining_ids<Share>(),
        with_defining_ids<Currency>(),
        kind.authority_type(),
    );

    Pool {
        id: claim(parent, key),
        // ...
    }
}
```

Type safety is preserved at creation time (generic parameters), while lookup remains flexible via stored `TypeName` values. An incorrect `TypeName` yields a wrong address (nothing found), not a security vulnerability.

---

## Chaining Derivations

Derived objects can themselves be parents. This creates deterministic object hierarchies:

```
Registry
  └─ Parent (derived via ContentKey(digest))
       ├─ ParentAdminCap (derived via AdminCapKey())
       └─ Child (derived via EditionKey(edition))
            └─ ChildAdminCap (derived via AdminCapKey())
```

```move
// 1. Parent derived from Registry
let parent = Parent {
    id: claim(&mut registry.id, ContentKey(digest)),
    // ...
};

// 2. AdminCap derived from Parent
let parent_admin_cap = ParentAdminCap {
    id: claim(&mut parent.id, AdminCapKey()),
};

// 3. Child derived from Parent
let child = Child {
    id: claim(&mut parent.id, EditionKey(edition)),
    // ...
};

// 4. AdminCap derived from Child
let child_admin_cap = ChildAdminCap {
    id: claim(&mut child.id, AdminCapKey()),
};
```

The entire chain is computable client-side before any object exists on-chain.

## Ownership Verification

Use `derive_address` to verify a derived object belongs to its parent without storing the relationship:

```move
public fun withdraw<Currency>(
    self: &mut Vault<Currency>,
    parent: &mut UID,
    idx: u64,
    value: Option<u64>,
): Balance<Currency> {
    assert!(
        self.id.to_address() == derive_address(parent.to_inner(), VaultKey<Currency>(idx)),
        EUnauthorized,
    );
    self.balance.split(value.destroy_or!(self.balance.value()))
}
```

## Existence Checks

Use `exists` to enforce ordering or prevent duplicates:

```move
// Enforce sequential edition numbers
if (edition > 0) {
    assert!(
        derived_object::exists(parent.uid(), EditionKey(edition - 1)),
        ENotSequentialEdition,
    );
};
```

Note: `exists` returns `true` even after the derived UID is deleted, because the `Claimed` marker is a dynamic field on the parent that persists.

---

## Client-Side Derivation (TypeScript)

Use `deriveObjectID` from `@mysten/sui/utils` to compute derived addresses off-chain. The result matches `derived_object::derive_address` on-chain.

```typescript
import { deriveObjectID } from '@mysten/sui/utils';
import { bcs } from '@mysten/sui/bcs';
```

### `deriveObjectID(parentId, keyType, keyBytes)`

| Parameter | Type | Description |
|-----------|------|-------------|
| `parentId` | `string` | Object ID of the parent |
| `keyType` | `string` | Fully-qualified Move type of the key struct |
| `keyBytes` | `Uint8Array` | BCS-serialized key value |

### Unit Struct Keys

Unit structs have a compiler-injected `dummy_field: bool = false`. BCS = `0x00`.

```typescript
const UNIT_STRUCT_KEY_BYTES = new Uint8Array([0x00]);

// AdminCapKey() — derive AdminCap from parent
function deriveAdminCapId(parentId: string, packageId: string): string {
  const keyType = `${packageId}::my_module::AdminCapKey`;
  return deriveObjectID(parentId, keyType, UNIT_STRUCT_KEY_BYTES);
}
```

### Primitive Value Keys

BCS-serialize the inner value.

```typescript
// ProfileKey(address) — derive Profile from Registry
function deriveProfileId(
  registryId: string,
  walletAddress: string,
  packageId: string,
): string {
  const keyType = `${packageId}::my_module::ProfileKey`;
  const keyBytes = bcs.Address.serialize(walletAddress).toBytes();
  return deriveObjectID(registryId, keyType, keyBytes);
}

// EditionKey(u16)
function deriveEditionId(
  parentId: string,
  edition: number,
  packageId: string,
): string {
  const keyType = `${packageId}::my_module::EditionKey`;
  const keyBytes = bcs.u16().serialize(edition).toBytes();
  return deriveObjectID(parentId, keyType, keyBytes);
}
```

### Vector Keys

```typescript
// ContentKey(vector<u8>) — derive from digest bytes
function deriveContentId(
  registryId: string,
  digest: Uint8Array,
  packageId: string,
): string {
  const keyType = `${packageId}::my_module::ContentKey`;
  const keyBytes = bcs.vector(bcs.u8()).serialize(digest).toBytes();
  return deriveObjectID(registryId, keyType, keyBytes);
}
```

### Complex Struct Keys

Define a BCS layout matching the Move struct and serialize.

```typescript
// PoolKey(TypeName, TypeName, Option<TypeName>)
const TypeNameBcs = bcs.struct("TypeName", { name: bcs.string() });

const PoolKeyBcs = bcs.struct("PoolKey", {
  share_type: TypeNameBcs,
  currency_type: TypeNameBcs,
  authority_type: bcs.option(TypeNameBcs),
});

function derivePoolId(
  parentId: string,
  shareType: string,
  currencyType: string,
  packageId: string,
): string {
  const keyType = `${packageId}::pool::PoolKey`;
  const keyBytes = PoolKeyBcs.serialize({
    share_type: { name: toMoveTypeName(shareType) },
    currency_type: { name: toMoveTypeName(currencyType) },
    authority_type: null,
  }).toBytes();
  return deriveObjectID(parentId, keyType, keyBytes);
}
```

### TypeName Conversion

Move's `type_name::get<T>()` returns addresses as 64-char zero-padded hex without `0x` prefix. SDK type strings use `0x`-prefixed, non-padded addresses. Convert before serializing:

```typescript
function toMoveTypeName(typeStr: string): string {
  return typeStr.replace(/0x([0-9a-fA-F]+)/g, (_, hex: string) => hex.padStart(64, "0"));
}

// "0x2::sui::SUI" → "0000...0002::sui::SUI"
```

### Phantom-Typed Keys

For keys with phantom type parameters like `VaultKey<phantom Currency>(u64)`, include the full generic type in the `keyType` string. Only BCS-serialize the non-phantom fields.

```typescript
// VaultKey<phantom Currency>(u64) — only the u64 is BCS-encoded
function deriveVaultId(
  parentId: string,
  idx: number,
  currencyType: string,
  packageId: string,
): string {
  const keyType = `${packageId}::vault::VaultKey<${currencyType}>`;
  const keyBytes = bcs.u64().serialize(idx).toBytes();
  return deriveObjectID(parentId, keyType, keyBytes);
}
```

### Coin Registry (Sui Framework)

The framework's `CoinRegistry` uses derived objects for `Currency` objects. Built-in example of a fieldless generic key:

```typescript
// CurrencyKey<phantom T>() — fieldless, just dummy_field: bool = false
const coinType = '0x2::sui::SUI';
const currencyId = deriveObjectID(
  '0xc', // CoinRegistry object ID
  `0x2::coin_registry::CurrencyKey<${coinType}>`,
  new Uint8Array([0]),
);
```

---

## Verification

Always verify your off-chain derivation matches on-chain at least once:

```typescript
// Compute off-chain
const expectedId = deriveObjectID(parentId, keyType, keyBytes);

// Verify on-chain
const object = await client.core.getObject({ objectId: expectedId });
assert(object !== null, 'Derived object ID mismatch');
```

## When to Use Derived Objects

| Use Case | Pattern | Example |
|----------|---------|---------|
| Admin capability per object | Unit struct key | `AdminCapKey()` |
| One entry per user | Address key | `ProfileKey(address)` |
| Sequential children | Integer key | `EditionKey(u16)` |
| Named entries in registry | String key | `CategoryKey(String)` |
| Content-addressed objects | Digest key | `ContentKey(vector<u8>)` |
| One per type | TypeName key | `ChildKey(TypeName)` |
| Multi-currency children | Phantom-typed key | `VaultKey<phantom Currency>(u64)` |
| Runtime-flexible pools | Complex key with TypeName | `PoolKey(TypeName, TypeName, Option<TypeName>)` |

## Derived Objects vs Dynamic Fields

| Property | Derived Objects | Dynamic Fields |
|----------|----------------|----------------|
| Deterministic address | Yes | No (hash-based, but not an independent object) |
| Independent after creation | Yes — no parent sequencing | No — always requires parent |
| Client-side ID computation | Yes | N/A (not standalone objects) |
| Parallelization | Full — objects are independent | Bottleneck on parent |
| Uniqueness guarantee | One per (parent, key) | One per (parent, key) |
| Discoverability | Via address derivation | Via parent object |
