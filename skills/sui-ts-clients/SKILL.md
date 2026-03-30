---
name: sui-ts-clients
description: "Sui TypeScript SDK Clients — how to create, configure, and use SuiGrpcClient, SuiGraphQLClient, Core API methods, native gRPC services, custom GraphQL queries, client extensions, and type utilities from @mysten/sui."
---

# Sui TypeScript SDK Clients

Reference for `@mysten/sui` client types. Covers client selection, Core API, native gRPC services, custom GraphQL queries, extensions, and types.

## Client Selection

| Client | Import | Status | Use Case |
|--------|--------|--------|----------|
| `SuiGrpcClient` | `@mysten/sui/grpc` | **Recommended** | Default for all apps |
| `SuiGraphQLClient` | `@mysten/sui/graphql` | Standard | Advanced query patterns, filtering |
| `SuiJsonRpcClient` | `@mysten/sui/client` | **Deprecated** | Legacy only — being decommissioned |

**Decision:** Use gRPC by default. Use GraphQL only when you need query patterns that full nodes can't support (e.g., filtering transactions/events, cross-owner object queries, nested data in one request).

All clients are compatible with Mysten SDKs (`@mysten/walrus`, `@mysten/seal`, `@mysten/suins`).

## API Layers

Every client exposes two API layers:

- **Core API** (`client.core.*`) — Transport-agnostic interface. Consistent across all clients. Use first.
- **Native API** — Full access to transport-specific features. Use when Core API doesn't cover the need.

---

## SuiGrpcClient

### Setup

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';

const client = new SuiGrpcClient({
  network: 'mainnet', // 'mainnet' | 'testnet' | 'devnet' | 'localnet'
  baseUrl: 'https://fullnode.mainnet.sui.io:443',
});
```

Local development:

```typescript
const client = new SuiGrpcClient({
  network: 'localnet',
  baseUrl: 'http://127.0.0.1:9000',
});
```

Custom transport:

```typescript
import { GrpcWebFetchTransport } from '@protobuf-ts/grpcweb-transport';

const transport = new GrpcWebFetchTransport({
  baseUrl: 'https://your-custom-grpc-endpoint.com',
});

const client = new SuiGrpcClient({
  network: 'testnet',
  transport,
});
```

### Native gRPC Services

Use when Core API doesn't have the method you need. Service clients are generated via `protobuf-ts`.

| Service | Access | Purpose |
|---------|--------|---------|
| `transactionExecutionService` | `client.transactionExecutionService` | Execute transactions |
| `ledgerService` | `client.ledgerService` | Query transactions, epochs |
| `stateService` | `client.stateService` | List objects, dynamic fields |
| `movePackageService` | `client.movePackageService` | Get function metadata |
| `nameService` | `client.nameService` | SuiNS resolution |
| `signatureVerificationService` | `client.signatureVerificationService` | Verify signatures |

#### Transaction Execution Service

```typescript
const { response } = await client.transactionExecutionService.executeTransaction({
  transaction: {
    bcs: { value: transactionBytes },
  },
  signatures: signatures.map((sig) => ({
    bcs: { value: fromBase64(sig) },
    signature: { oneofKind: undefined },
  })),
});

// Always check status
if (!response.finality?.effects?.status?.success) {
  const error = response.finality?.effects?.status?.error;
  throw new Error(`Transaction failed: ${error || 'Unknown error'}`);
}
```

#### Ledger Service

```typescript
const { response } = await client.ledgerService.getTransaction({
  digest: '0x123...',
});

const { response: epochInfo } = await client.ledgerService.getEpoch({});
```

#### State Service

```typescript
const { response } = await client.stateService.listOwnedObjects({
  owner: '0xabc...',
  objectType: '0x2::coin::Coin<0x2::sui::SUI>',
});

const { response: fields } = await client.stateService.listDynamicFields({
  parent: '0x123...',
});
```

#### Move Package Service

```typescript
const { response } = await client.movePackageService.getFunction({
  packageId: '0x2',
  moduleName: 'coin',
  name: 'transfer',
});
```

#### Name Service

```typescript
const { response } = await client.nameService.reverseLookupName({
  address: '0xabc...',
});
```

#### Signature Verification Service

```typescript
const { response } = await client.signatureVerificationService.verifySignature({
  message: {
    name: 'TransactionData',
    value: messageBytes,
  },
  signature: {
    bcs: { value: signatureBytes },
    signature: { oneofKind: undefined },
  },
  jwks: [],
});
```

---

## SuiGraphQLClient

### Setup

```typescript
import { SuiGraphQLClient } from '@mysten/sui/graphql';
import { graphql } from '@mysten/sui/graphql/schema';

const client = new SuiGraphQLClient({
  url: 'https://sui-mainnet.mystenlabs.com/graphql',
  network: 'mainnet',
});
```

The client implements all Core API methods:

```typescript
const { object } = await client.core.getObject({ objectId: '0x...' });
```

### Custom GraphQL Queries

Use the `query` method for anything not in the Core API. The `graphql` function (powered by `gql.tada`) provides automatic type safety.

```typescript
const chainIdentifierQuery = graphql(`
  query {
    chainIdentifier
  }
`);

const result = await client.query({
  query: chainIdentifierQuery,
});

console.log(result.data?.chainIdentifier);
```

With variables:

```typescript
const getSuinsName = graphql(`
  query getSuiName($address: SuiAddress!) {
    address(address: $address) {
      defaultSuinsName
    }
  }
`);

const result = await client.query({
  query: getSuinsName,
  variables: { address: '0xabc...' },
});

console.log(result.data?.address?.defaultSuinsName);
```

### TypedDocumentNode Compatibility

GraphQL document nodes implement the `TypedDocumentNode` standard and work with Apollo, urql, and other popular GraphQL clients:

```typescript
import { useQuery } from '@apollo/client';

const chainIdentifierQuery = graphql(`
  query { chainIdentifier }
`);

function ChainIdentifier() {
  const { data } = useQuery(chainIdentifierQuery);
  return <div>{data?.chainIdentifier}</div>;
}
```

### When to Use GraphQL

Only use GraphQL when Core API and gRPC native methods cannot accomplish the task:

- Querying objects by type across all owners
- Complex filtering combinations (by type AND checkpoint, etc.)
- Filtering transactions or events
- Nested/related data fetching in one request
- Aggregations or queries that don't exist in Core/gRPC APIs

---

## Core API Reference

Transport-agnostic interface implemented by all clients. Access via `client.core.*`.

### ClientWithCoreApi Type

Use this type when building SDKs or libraries that accept any Sui client:

```typescript
import type { ClientWithCoreApi } from '@mysten/sui/client';

class MySDK {
  constructor(private client: ClientWithCoreApi) {}

  async doSomething() {
    return this.client.core.getObject({ objectId: '0x...' });
  }
}
```

### Object Methods

Every object always includes `objectId`, `version`, `digest`, `owner`, and `type`. Additional data requires `include`:

| Include Option | Type | Description |
|----------------|------|-------------|
| `content` | `boolean` | BCS-encoded Move struct content — pass to generated BCS type parsers |
| `previousTransaction` | `boolean` | Digest of the transaction that last mutated this object |
| `json` | `boolean` | JSON representation of the object's Move struct content |
| `objectBcs` | `boolean` | Full BCS-encoded object envelope (rarely needed) |
| `display` | `boolean` | Sui Display Standard metadata |

#### getObject

```typescript
const { object } = await client.core.getObject({
  objectId: '0x123...',
  include: {
    content: true,
    previousTransaction: true,
  },
});

console.log(object.objectId);
console.log(object.version);
console.log(object.digest);
console.log(object.type); // e.g., "0x2::coin::Coin<0x2::sui::SUI>"
```

#### getObjects

```typescript
const { objects } = await client.core.getObjects({
  objectIds: ['0x123...', '0x456...'],
  include: { content: true },
});

for (const obj of objects) {
  if (obj instanceof Error) {
    console.log('Object not found:', obj.message);
  } else {
    console.log(obj.objectId, obj.type);
  }
}
```

#### listOwnedObjects

```typescript
const result = await client.core.listOwnedObjects({
  owner: '0xabc...',
  filter: { StructType: '0x2::coin::Coin<0x2::sui::SUI>' },
  limit: 10,
});

for (const obj of result.objects) {
  console.log(obj.objectId, obj.type);
}

// Paginate
if (result.cursor) {
  const nextPage = await client.core.listOwnedObjects({
    owner: '0xabc...',
    cursor: result.cursor,
  });
}
```

#### Parsing Object Content

Use `include: { content: true }` to get BCS-encoded bytes, then parse with generated types or manual BCS definitions:

```typescript
const { object } = await client.core.getObject({
  objectId: '0x123...',
  include: { content: true },
});

const parsed = MyStruct.parse(object.content);
```

#### json Include Option

> **Warning:** The `json` field structure varies between API implementations. JSON-RPC returns UID fields as nested objects (`{ "id": { "id": "0x..." } }`), while gRPC and GraphQL flatten them (`{ "id": "0x..." }`). For consistency across clients, use `content` and parse BCS directly.

#### objectBcs Include Option

Returns the full BCS-encoded object envelope (struct content wrapped in metadata). Parse only with `bcs.Object` from `@mysten/sui/bcs`:

```typescript
const envelope = bcs.Object.parse(object.objectBcs);
```

> **Do not** pass `objectBcs` to a Move struct parser — it contains wrapping metadata that will cause parsing failures. Use `content` for parsing Move struct fields.

#### display Include Option

Fetches Sui Display Standard metadata. The `display` field is `null` when no Display template exists, `undefined` when not requested.

```typescript
const { object } = await client.core.getObject({
  objectId: '0x123...',
  include: { display: true },
});

if (object.display) {
  console.log(object.display.output?.name);
  console.log(object.display.output?.image_url);
}
```

| Field | Type | Description |
|-------|------|-------------|
| `output` | `Record<string, string> \| null` | Interpolated display fields |
| `errors` | `Record<string, string> \| null` | Per-field errors if template interpolation failed |

Works with `getObject`, `getObjects`, and `listOwnedObjects`.

### Coin and Balance Methods

#### getBalance

```typescript
const balance = await client.core.getBalance({
  owner: '0xabc...',
  coinType: '0x2::sui::SUI', // Optional, defaults to SUI
});

console.log(balance.totalBalance);    // bigint
console.log(balance.coinObjectCount); // number
```

#### listBalances

```typescript
const { balances } = await client.core.listBalances({
  owner: '0xabc...',
});

for (const balance of balances) {
  console.log(balance.coinType, balance.totalBalance);
}
```

#### listCoins

```typescript
const result = await client.core.listCoins({
  owner: '0xabc...',
  coinType: '0x2::sui::SUI',
  limit: 10,
});

for (const coin of result.coins) {
  console.log(coin.objectId, coin.balance);
}
```

#### getCoinMetadata

```typescript
const { coinMetadata } = await client.core.getCoinMetadata({
  coinType: '0x2::sui::SUI',
});

if (coinMetadata) {
  console.log(coinMetadata.name, coinMetadata.symbol, coinMetadata.decimals);
  // "Sui" "SUI" 9
}
```

### Dynamic Field Methods

#### listDynamicFields

```typescript
const result = await client.core.listDynamicFields({
  parentId: '0x123...',
  limit: 10,
});

for (const field of result.dynamicFields) {
  console.log(field.name, field.type);
}
```

#### getDynamicField

```typescript
import { bcs } from '@mysten/sui/bcs';

const { dynamicField } = await client.core.getDynamicField({
  parentId: '0x123...',
  name: {
    type: 'u64',
    bcs: bcs.u64().serialize(42).toBytes(),
  },
});

console.log(dynamicField.name);
console.log(dynamicField.value.type);
console.log(dynamicField.value.bcs); // BCS-encoded value
```

#### getDynamicObjectField

```typescript
const { object } = await client.core.getDynamicObjectField({
  parentId: '0x123...',
  name: {
    type: '0x2::object::ID',
    bcs: bcs.Address.serialize('0x456...').toBytes(),
  },
  include: { content: true },
});
```

### Transaction Methods

Transaction results always include `digest`, `signatures`, `epoch`, and `status`. Additional data requires `include`:

| Include Option | Type | Description |
|----------------|------|-------------|
| `effects` | `boolean` | Parsed transaction effects (gas used, changed objects, status) |
| `events` | `boolean` | Events emitted during execution |
| `transaction` | `boolean` | Parsed transaction data (sender, gas config, inputs, commands) |
| `balanceChanges` | `boolean` | Balance changes caused by the transaction |
| `objectTypes` | `boolean` | Map of object IDs to their types for all changed objects |
| `bcs` | `boolean` | Raw BCS-encoded transaction bytes |

`simulateTransaction` also supports:

| Include Option | Type | Description |
|----------------|------|-------------|
| `commandResults` | `boolean` | Return values and mutated references from each command |

#### executeTransaction

```typescript
const result = await client.core.executeTransaction({
  transaction: transactionBytes,
  signatures: [signature],
  include: { effects: true, events: true },
});

if (result.Transaction) {
  console.log('Success:', result.Transaction.digest);
  console.log('Effects:', result.Transaction.effects);
} else {
  console.log('Failed:', result.FailedTransaction?.status.error);
}
```

#### simulateTransaction

```typescript
const result = await client.core.simulateTransaction({
  transaction: transactionBytes,
  include: {
    effects: true,
    balanceChanges: true,
    commandResults: true,
  },
});
```

Disable checks for non-public or non-entry Move functions:

```typescript
const result = await client.core.simulateTransaction({
  transaction: transactionBytes,
  checksEnabled: false,
  include: { commandResults: true },
});
```

#### signAndExecuteTransaction

```typescript
import { Transaction } from '@mysten/sui/transactions';

const tx = new Transaction();
tx.transferObjects([tx.object('0x123...')], '0xrecipient...');

const result = await client.core.signAndExecuteTransaction({
  transaction: tx,
  signer: keypair,
  include: { effects: true },
});

if (result.FailedTransaction) {
  throw new Error(`Failed: ${result.FailedTransaction.status.error}`);
}

console.log('Digest:', result.Transaction.digest);
```

#### getTransaction

```typescript
const result = await client.core.getTransaction({
  digest: 'ABC123...',
  include: { effects: true, events: true, transaction: true },
});

console.log(result.Transaction?.digest);
console.log(result.Transaction?.effects);
console.log(result.Transaction?.transaction?.sender);
```

Fetch raw BCS bytes:

```typescript
const result = await client.core.getTransaction({
  digest: 'ABC123...',
  include: { bcs: true },
});

console.log(result.Transaction?.bcs); // Uint8Array
```

#### waitForTransaction

```typescript
const result = await client.core.waitForTransaction({
  digest: 'ABC123...',
  timeout: 60_000,
  include: { effects: true },
});
```

Pass executeTransaction result directly:

```typescript
const executeResult = await client.core.executeTransaction({ ... });

const finalResult = await client.core.waitForTransaction({
  result: executeResult,
  include: { effects: true },
});
```

### System Methods

#### getReferenceGasPrice

```typescript
const { referenceGasPrice } = await client.core.getReferenceGasPrice();
console.log(referenceGasPrice); // bigint
```

#### getCurrentSystemState

```typescript
const systemState = await client.core.getCurrentSystemState();
console.log(systemState.epoch);
console.log(systemState.systemStateVersion);
```

#### getChainIdentifier

```typescript
const { chainIdentifier } = await client.core.getChainIdentifier();
console.log(chainIdentifier); // e.g., "4c78adac"
```

### Move Methods

#### getMoveFunction

```typescript
const { function: fn } = await client.core.getMoveFunction({
  packageId: '0x2',
  moduleName: 'coin',
  name: 'transfer',
});

console.log(fn.name);
console.log(fn.parameters);
console.log(fn.typeParameters);
```

### Name Service Methods

#### defaultNameServiceName

```typescript
const { name } = await client.core.defaultNameServiceName({
  address: '0xabc...',
});

console.log(name); // e.g., "example.sui"
```

### MVR Methods

Resolve Move Registry type names:

```typescript
const { type } = await client.core.mvr.resolveType({
  type: '@mysten/sui::coin::Coin<@mysten/sui::sui::SUI>',
});

console.log(type); // "0x2::coin::Coin<0x2::sui::SUI>"
```

---

## Client Extensions

All clients support extensions via `$extend`:

```typescript
import { walrus } from '@mysten/walrus';

const client = new SuiGrpcClient({
  network: 'mainnet',
  baseUrl: 'https://fullnode.mainnet.sui.io:443',
}).$extend(walrus());

await client.walrus.writeBlob({ ... });
```

---

## SuiClientTypes Namespace

Import types for function parameters, return values, and variables:

```typescript
import type { SuiClientTypes } from '@mysten/sui/client';

function processObject(obj: SuiClientTypes.Object<{ content: true }>) {
  console.log(obj.objectId, obj.content);
}

async function fetchBalance(
  client: ClientWithCoreApi,
  owner: string,
): Promise<SuiClientTypes.CoinBalance> {
  const { balance } = await client.core.getBalance({ owner });
  return balance;
}

const options: SuiClientTypes.GetObjectOptions<{ content: true }> = {
  objectId: '0x123...',
  include: { content: true },
};
```

### Common Types

| Type | Description |
|------|-------------|
| `Object<Include>` | Fetched object with optional included data |
| `Coin` | Coin object with balance |
| `CoinBalance` | Balance summary for a coin type |
| `CoinMetadata` | Metadata for a coin type |
| `Transaction<Include>` | Executed transaction with optional data |
| `TransactionResult` | Success or failure result from execution |
| `TransactionEffects` | Detailed effects from transaction execution |
| `Event` | Emitted event from a transaction |
| `ObjectOwner` | Union of all owner types |
| `ExecutionStatus` | Success/failure status with error details |
| `DynamicFieldName` | Name identifier for dynamic fields |
| `FunctionResponse` | Move function metadata |
| `Network` | Network identifier type |

### Include Options Types

| Type | Description |
|------|-------------|
| `ObjectInclude` | Options for object data inclusion |
| `TransactionInclude` | Options for transaction data inclusion |
| `SimulateTransactionInclude` | Extended options for simulation results |

### Method Options Types

| Type | Description |
|------|-------------|
| `GetObjectOptions` | Options for `getObject` |
| `GetObjectsOptions` | Options for `getObjects` |
| `ListOwnedObjectsOptions` | Options for `listOwnedObjects` |
| `ListCoinsOptions` | Options for `listCoins` |
| `GetBalanceOptions` | Options for `getBalance` |
| `ListBalancesOptions` | Options for `listBalances` |
| `GetCoinMetadataOptions` | Options for `getCoinMetadata` |
| `ExecuteTransactionOptions` | Options for `executeTransaction` |
| `SimulateTransactionOptions` | Options for `simulateTransaction` |
| `SignAndExecuteTransactionOptions` | Options for `signAndExecuteTransaction` |
| `GetTransactionOptions` | Options for `getTransaction` |
| `WaitForTransactionOptions` | Options for `waitForTransaction` |

---

## Error Handling

Object fetches may return errors in the result:

```typescript
const { objects } = await client.core.getObjects({
  objectIds: ['0x123...', '0x456...'],
});

for (const obj of objects) {
  if (obj instanceof Error) {
    console.error('Error:', obj.message);
  } else {
    console.log(obj.objectId);
  }
}
```

Transaction execution — always check the result variant:

```typescript
const result = await client.core.executeTransaction({ ... });

if (result.Transaction) {
  console.log(result.Transaction.digest);
} else if (result.FailedTransaction) {
  throw new Error(result.FailedTransaction.status.error);
}
```

---

## Migration from JSON-RPC

| Old (JSON-RPC) | New (Core API) |
|----------------|----------------|
| `new SuiClient({ url })` | `new SuiGrpcClient({ network, baseUrl })` |
| `client.getObject()` | `client.core.getObject()` |
| `client.getOwnedObjects()` | `client.core.listOwnedObjects()` |
| `client.executeTransactionBlock()` | `client.core.executeTransaction()` |
| `client.signAndExecuteTransactionBlock()` | `client.core.signAndExecuteTransaction()` |
| `client.waitForTransactionBlock()` | `client.core.waitForTransaction()` |
| `client.getCoins()` | `client.core.listCoins()` |
| `client.getBalance()` | `client.core.getBalance()` |

JSON-RPC is deprecated and will be decommissioned. Migrate to `SuiGrpcClient` with Core API.
