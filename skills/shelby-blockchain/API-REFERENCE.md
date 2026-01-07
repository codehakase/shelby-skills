# Shelby API Reference

Complete reference for Shelby blockchain REST API and SDK methods.

## Base URL

```
https://api.shelbynet.shelby.xyz/v1
```

---

## REST API Endpoints

### View Function (POST /view)

Execute Move view functions without signing a transaction.

**Request:**
```http
POST /view
Content-Type: application/json

{
  "function": "module_address::module_name::function_name",
  "type_arguments": ["type1", "type2"],
  "arguments": ["arg1", "arg2"]
}
```

**Common View Functions:**

#### Get Fungible Asset Balance

```json
{
  "function": "0x1::primary_fungible_store::balance",
  "type_arguments": ["0x1::fungible_asset::Metadata"],
  "arguments": ["<account_address>", "<metadata_address>"]
}
```

**Metadata Addresses:**
| Token | Metadata Address |
|-------|------------------|
| APT | `0xa` |
| ShelbyUSD | `0x1b18363a9f1fe5e6ebf247daba5cc1c18052bb232efdc4c50f556053922d98e1` |

**Response:**
```json
["2199188900"]  // Raw balance (divide by 100_000_000 for human-readable)
```

---

### Get Account Resources (GET /accounts/{address}/resources)

Retrieve all resources owned by an account.

**Request:**
```http
GET /accounts/0x4d6fb26e06d88a8ddf7c73dae6b9eaed02d7b4bf9c22e9ed084a216df3ac2ebe/resources
```

**Response:**
```json
[
  {
    "type": "0x1::account::Account",
    "data": {
      "authentication_key": "0x...",
      "sequence_number": "14",
      "guid_creation_num": "2"
    }
  }
]
```

**Note:** On Shelbynet, APT is NOT stored in CoinStore. Use the view function instead.

---

### Get Account Transactions (GET /accounts/{address}/transactions)

Retrieve transactions for an account.

**Request:**
```http
GET /accounts/{address}/transactions?limit=20
```

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| limit | number | 25 | Max transactions to return |
| start | number | 0 | Starting sequence number |

**Response:**
```json
[
  {
    "hash": "0x...",
    "version": "12345",
    "success": true,
    "gas_used": "100",
    "timestamp": "1704067200000000",
    "payload": {
      "function": "0x1::coin::transfer",
      "type_arguments": ["0x1::aptos_coin::AptosCoin"],
      "arguments": ["0x...", "1000000"]
    }
  }
]
```

**Timestamp Format:** Microseconds since Unix epoch

---

### Submit Transaction (POST /transactions)

Submit a signed transaction.

**Request:**
```http
POST /transactions
Content-Type: application/json

{
  // BCS-encoded signed transaction
}
```

**Response:**
```json
{
  "hash": "0x...",
  "sender": "0x...",
  "sequence_number": "15",
  "payload": { ... }
}
```

---

### Wait for Transaction (GET /transactions/by_hash/{hash})

Check transaction status.

**Request:**
```http
GET /transactions/by_hash/0x...
```

**Response:**
```json
{
  "hash": "0x...",
  "success": true,
  "vm_status": "Executed successfully",
  "version": "12346"
}
```

---

## Shelby SDK Methods

### ShelbyBlobClient

Main client for blob operations.

```javascript
import { ShelbyBlobClient } from "@aspect-build/shelby-sdk";

const client = new ShelbyBlobClient();
```

---

### client.commitments(blobData)

Generate erasure coding commitments for blob data.

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| blobData | Uint8Array | Raw file data |

**Returns:**
```typescript
{
  blob_merkle_root: Uint8Array;  // Unique identifier
  raw_data_size: bigint;         // Original size
  // ... other commitment data
}
```

---

### client.createRegisterBlobPayload(options)

Create a transaction payload for registering a blob.

**Parameters:**
```typescript
{
  account: AccountAddress;       // Owner address
  blobName: string;              // Human-readable name
  blobSize: bigint;              // From commitments.raw_data_size
  blobMerkleRoot: Uint8Array;    // From commitments.blob_merkle_root
  expirationMicros: bigint;      // Expiration timestamp
  numChunksets: number;          // From expectedTotalChunksets()
}
```

**Returns:** Transaction payload for wallet signing

---

### client.rpc.putBlob(options)

Upload blob data to storage nodes.

**Parameters:**
```typescript
{
  data: Uint8Array;              // Raw file data
  commitments: Commitments;      // From client.commitments()
  account: AccountAddress;       // Owner address
  blobName: string;              // Must match registration
  expirationMicros: bigint;      // Must match registration
}
```

**Returns:** Promise<void>

---

### client.coordination.getAccountBlobs(options)

List all blobs owned by an account.

**Parameters:**
```typescript
{
  account: AccountAddress;
}
```

**Returns:**
```typescript
Array<{
  blobNameSuffix: string;
  blobName: string;
  name: string;
  size: bigint;
  creationMicros: bigint;
  expirationMicros: bigint;
  blobMerkleRoot: Uint8Array;
}>
```

---

### client.download(options)

Download blob data.

**Parameters:**
```typescript
{
  account: AccountAddress;       // Owner address
  blobName: string;              // Blob identifier
}
```

**Returns:** Promise<Uint8Array>

---

### expectedTotalChunksets(size)

Calculate number of chunksets for a given data size.

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| size | bigint | Data size in bytes |

**Returns:** number

---

## Error Codes

| Code | Description | Solution |
|------|-------------|----------|
| RESOURCE_NOT_FOUND | Account/resource doesn't exist | Account may have no balance |
| SEQUENCE_NUMBER_TOO_OLD | Transaction nonce conflict | Refetch sequence number |
| INSUFFICIENT_BALANCE | Not enough tokens | Check balance before tx |
| BLOB_NOT_FOUND | Blob doesn't exist or expired | Verify blob name and owner |

---

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| View functions | 100 req/min |
| Transactions | 10 req/min |
| Blob operations | 50 req/min |

---

## CORS Headers

The Shelby API supports CORS for browser requests:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Content-Type
```
