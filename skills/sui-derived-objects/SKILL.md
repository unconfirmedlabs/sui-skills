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

The derivation key is a struct you define in your module. It determines how many derived objects a parent can produce and what identifies each one. There are three patterns.

### Pattern 1: Unit Struct Key (one-to-one)

Derives exactly **one** object per parent. Used for admin capabilities, metadata singletons, or any 1:1 relationship.

```move
/// Key for deriving the admin capability from the composition.
public struct CompositionAdminCapKey() has copy, drop, store;

public struct CompositionAdminCap<phantom CompositionShare> has key, store {
    id: UID,
}

// In the creation function:
let composition_admin_cap = CompositionAdminCap<CompositionShare> {
    id: claim(&mut composition.id, CompositionAdminCapKey()),
};
```

The unit struct has no fields, so BCS encodes as a single `0x00` byte (the compiler injects `dummy_field: bool = false`).

**More examples:**

```move
public struct RecordingAdminCapKey() has copy, drop, store;
public struct ReleaseAdminCapKey() has copy, drop, store;
public struct PressingAdminCapKey() has copy, drop, store;
public struct VaultAdminCapKey() has copy, drop, store;
```

### Pattern 2: Value Key (one-to-many)

Derives **multiple** objects from one parent, each identified by a value. Used for registries, collections, and indexed children.

**Address key** — one per wallet:

```move
public struct PlayerKey(address) has copy, drop, store;

// In the creation function:
let player = Player {
    id: derived_object::claim(&mut registry.id, PlayerKey(ctx.sender())),
    // ...
};
```

**Integer key** — sequential editions:

```move
public struct PressingKey(u16) has copy, drop, store;

// Verify sequential ordering before claiming
if (edition > 0) {
    assert!(
        derived_object::exists(release.uid(), PressingKey(edition - 1)),
        ENotSequentialEditionNumber,
    );
};

let pressing = Pressing<Currency> {
    id: derived_object::claim(release.uid_mut(cap), PressingKey(edition)),
    // ...
};
```

**String key** — named entries:

```move
public struct GenreKey(String) has copy, drop, store;

let genre = Genre {
    id: claim(&mut genre_registry.id, GenreKey(name)),
    name,
};
```

**Digest key** — content-addressed:

```move
public struct ReleaseKey(vector<u8>) has copy, drop, store;

// Hash the content to produce a deterministic key
fun calculate_release_digest(
    recording_ids: vector<ID>,
    track_split_values: vector<u64>,
    nonce: u256,
): vector<u8> {
    let mut hash_input = vector<u8>[];
    hash_input.append(to_bytes(&recording_ids));
    hash_input.append(to_bytes(&track_split_values));
    hash_input.append(to_bytes(&nonce));
    blake2b256(&hash_input)
}

let release_uid = claim(&mut registry.id, ReleaseKey(release_digest));
```

**TypeName key** — one per type:

```move
public struct RecordingKey(TypeName) has copy, drop, store;

let recording = Recording<RecordingShare> {
    id: claim(
        composition.uid_mut_internal(),
        RecordingKey(*master.ingester_type()),
    ),
    // ...
};
```

### Pattern 3: Phantom-Typed Key (type-namespaced)

Uses a phantom type parameter on the key struct to namespace derivations by type. Useful when the same parent needs derived objects for different currency types or token types.

```move
public struct SiloKey<phantom Currency>(u64) has copy, drop, store;

public fun new<Currency>(parent: &mut UID, idx: u64): Silo<Currency> {
    Silo {
        id: claim(parent, SiloKey<Currency>(idx)),
        balance: balance::zero(),
    }
}
```

The phantom type is included in the type hash, so `SiloKey<SUI>(0)` and `SiloKey<USDC>(0)` produce different addresses from the same parent.

### Pattern 4: Complex Key (runtime-flexible)

Uses `TypeName` values instead of phantom types for runtime flexibility. Essential when you need to iterate over heterogeneous pools or derive addresses from stored type information.

```move
public struct RewardPoolKey(TypeName, TypeName, Option<TypeName>) has copy, drop, store;

public fun new<Share, Currency>(
    parent: &mut UID,
    kind: RewardPoolKind,
): RewardPool<Share, Currency> {
    let key = RewardPoolKey(
        with_defining_ids<Share>(),
        with_defining_ids<Currency>(),
        kind.authority_type(),
    );

    let reward_pool = RewardPool<Share, Currency> {
        id: claim(parent, key),
        // ...
    };
    reward_pool
}
```

Type safety is preserved at creation time (generic parameters), while lookup remains flexible via stored `TypeName` values. An incorrect `TypeName` yields a wrong address (nothing found), not a security vulnerability.

---

## Chaining Derivations

Derived objects can themselves be parents. This creates deterministic object hierarchies:

```
Registry
  └─ Release (derived via ReleaseKey(digest))
       ├─ ReleaseAdminCap (derived via ReleaseAdminCapKey())
       └─ Pressing (derived via PressingKey(edition))
            └─ PressingAdminCap (derived via PressingAdminCapKey())
```

```move
// 1. Release derived from Registry
let release_uid = claim(&mut registry.id, ReleaseKey(release_digest));

// 2. ReleaseAdminCap derived from Release
let release_admin_cap = ReleaseAdminCap {
    id: claim(&mut release.id, ReleaseAdminCapKey()),
    release_id: release.id(),
};

// 3. Pressing derived from Release
let pressing = Pressing<Currency> {
    id: derived_object::claim(release.uid_mut(cap), PressingKey(edition)),
    // ...
};

// 4. PressingAdminCap derived from Pressing
let pressing_admin_cap = PressingAdminCap {
    id: derived_object::claim(&mut pressing.id, PressingAdminCapKey()),
    pressing_id: pressing.id(),
};
```

The entire chain is computable client-side before any object exists on-chain.

## Ownership Verification

Use `derive_address` to verify a derived object belongs to its parent without storing the relationship:

```move
public fun withdraw<Currency>(
    self: &mut Silo<Currency>,
    parent: &mut UID,
    value: Option<u64>,
): Balance<Currency> {
    assert!(
        self.id.to_address() == derive_address(parent.to_inner(), SiloKey<Currency>(idx)),
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
        derived_object::exists(release.uid(), PressingKey(edition - 1)),
        ENotSequentialEditionNumber,
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

// Derive CompositionAdminCap from Composition
function deriveCompositionAdminCapId(
  compositionId: string,
  musicOsPackageId: string
): string {
  const keyType = `${musicOsPackageId}::composition::CompositionAdminCapKey`;
  return deriveObjectID(compositionId, keyType, UNIT_STRUCT_KEY_BYTES);
}
```

### Primitive Value Keys

BCS-serialize the inner value.

```typescript
// PlayerKey(address) — derive Player from PlayerRegistry
function derivePlayerId(
  playerRegistryId: string,
  walletAddress: string,
  sonaPlayerPackageId: string
): string {
  const keyType = `${sonaPlayerPackageId}::player::PlayerKey`;
  const keyBytes = bcs.Address.serialize(walletAddress).toBytes();
  return deriveObjectID(playerRegistryId, keyType, keyBytes);
}
```

### Vector Keys

```typescript
// ReleaseKey(vector<u8>) — derive Release from ReleaseRegistry
function deriveReleaseId(
  releaseRegistryId: string,
  releaseDigest: Uint8Array,
  musicOsPackageId: string
): string {
  const keyType = `${musicOsPackageId}::release::ReleaseKey`;
  const keyBytes = bcs.vector(bcs.u8()).serialize(releaseDigest).toBytes();
  return deriveObjectID(releaseRegistryId, keyType, keyBytes);
}
```

### Complex Struct Keys

Define a BCS layout matching the Move struct and serialize.

```typescript
const TypeNameBcs = bcs.struct("TypeName", { name: bcs.string() });

const RewardPoolKeyBcs = bcs.struct("RewardPoolKey", {
  share_type: TypeNameBcs,
  currency_type: TypeNameBcs,
  authority_type: bcs.option(TypeNameBcs),
});

function deriveRewardPoolId(
  recordingId: string,
  shareType: string,
  currencyType: string,
  umiPackageId: string,
): string {
  const keyType = `${umiPackageId}::reward_pool::RewardPoolKey`;
  const keyBytes = RewardPoolKeyBcs.serialize({
    share_type: { name: toMoveTypeName(shareType) },
    currency_type: { name: toMoveTypeName(currencyType) },
    authority_type: null,
  }).toBytes();
  return deriveObjectID(recordingId, keyType, keyBytes);
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

For keys with phantom type parameters like `SiloKey<phantom Currency>(u64)`, include the full generic type in the `keyType` string. Only BCS-serialize the non-phantom fields.

```typescript
// SiloKey<phantom Currency>(u64) — only the u64 is BCS-encoded
function deriveSiloId(
  parentId: string,
  idx: number,
  currencyType: string,
  aidaPackageId: string,
): string {
  const keyType = `${aidaPackageId}::silo::SiloKey<${currencyType}>`;
  const keyBytes = bcs.u64().serialize(idx).toBytes();
  return deriveObjectID(parentId, keyType, keyBytes);
}
```

### Coin Registry (Sui Framework)

The framework's `CoinRegistry` uses derived objects for `Currency` objects. This is a built-in example of a fieldless generic key:

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
| Admin capability per object | Unit struct key | `CompositionAdminCapKey()` |
| One entry per user | Address key | `PlayerKey(address)` |
| Sequential children | Integer key | `PressingKey(u16)` |
| Named entries in registry | String key | `GenreKey(String)` |
| Content-addressed objects | Digest key | `ReleaseKey(vector<u8>)` |
| One per type | TypeName key | `RecordingKey(TypeName)` |
| Multi-currency children | Phantom-typed key | `SiloKey<phantom Currency>(u64)` |
| Runtime-flexible pools | Complex key with TypeName | `RewardPoolKey(TypeName, TypeName, Option<TypeName>)` |

## Derived Objects vs Dynamic Fields

| Property | Derived Objects | Dynamic Fields |
|----------|----------------|----------------|
| Deterministic address | Yes | No (hash-based, but not an independent object) |
| Independent after creation | Yes — no parent sequencing | No — always requires parent |
| Client-side ID computation | Yes | N/A (not standalone objects) |
| Parallelization | Full — objects are independent | Bottleneck on parent |
| Uniqueness guarantee | One per (parent, key) | One per (parent, key) |
| Discoverability | Via address derivation | Via parent object |
