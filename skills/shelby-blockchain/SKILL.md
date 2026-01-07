---
name: shelby-blockchain
description: Interact with Shelby decentralized storage on Aptos blockchain. Use when uploading files, downloading blobs, checking APT/ShelbyUSD balances, listing transactions, or building Shelby-based applications. Covers blob storage, fungible assets, and WebDAV integration.
allowed-tools: Read, Glob, Grep, Bash(curl:*), Bash(npm:*), WebFetch
---

# Shelby Blockchain Operations

This skill provides comprehensive guidance for interacting with Shelby, a decentralized storage network built on the Aptos blockchain.

## Quick Reference

| Operation | Method | Endpoint/SDK |
|-----------|--------|--------------|
| Upload file | SDK | `client.upload()` + wallet signing |
| Download blob | SDK | `client.download()` |
| List blobs | SDK | `client.coordination.getAccountBlobs()` |
| APT balance | REST | POST `/view` with fungible asset |
| ShelbyUSD balance | REST | POST `/view` with fungible asset |
| List transactions | REST | GET `/accounts/{addr}/transactions` |

## Network Configuration

```javascript
// Shelby Fullnode API
const SHELBY_FULLNODE = "https://api.shelbynet.shelby.xyz/v1";

// Token Metadata Addresses (Fungible Assets on Shelbynet)
const APT_METADATA = "0xa";
const SHELBY_USD_METADATA = "0x1b18363a9f1fe5e6ebf247daba5cc1c18052bb232efdc4c50f556053922d98e1";

// Decimal conversion (8 decimals)
const DECIMAL_DIVISOR = 100_000_000;
```

## Core Operations

### 1. Fetching Token Balances

On Shelbynet, both APT and ShelbyUSD are stored as **Fungible Assets**, not legacy CoinStore. Use the `primary_fungible_store::balance` view function:

```javascript
async function fetchBalance(address, metadataAddress) {
  const payload = {
    function: "0x1::primary_fungible_store::balance",
    type_arguments: ["0x1::fungible_asset::Metadata"],
    arguments: [address, metadataAddress],
  };

  const response = await fetch(`${SHELBY_FULLNODE}/view`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload),
  });

  const result = await response.json();
  return Array.isArray(result) && result.length > 0
    ? Number(result[0]) / 100_000_000
    : 0;
}

// Usage
const aptBalance = await fetchBalance(walletAddress, "0xa");
const susdBalance = await fetchBalance(walletAddress, SHELBY_USD_METADATA);
```

### 2. Uploading Files to Shelby

```javascript
import { ShelbyBlobClient, expectedTotalChunksets } from "@aspect-build/shelby-sdk";

async function uploadToShelby(file, wallet) {
  const client = new ShelbyBlobClient();
  const blobData = new Uint8Array(await file.arrayBuffer());
  const account = AccountAddress.from(wallet.account.address);

  // Calculate expiration (365 days)
  const expirationMicros = BigInt(Date.now() + 365 * 24 * 60 * 60 * 1000) * 1000n;

  // Generate commitments
  const commitments = client.commitments(blobData);

  // Create registration payload
  const payload = client.createRegisterBlobPayload({
    account,
    blobName: file.name,
    blobSize: commitments.raw_data_size,
    blobMerkleRoot: commitments.blob_merkle_root,
    expirationMicros,
    numChunksets: expectedTotalChunksets(commitments.raw_data_size),
  });

  // Sign and submit transaction
  const txResponse = await wallet.signAndSubmitTransaction({ data: payload });
  await aptos.waitForTransaction({ transactionHash: txResponse.hash });

  // Upload blob data
  await client.rpc.putBlob({
    data: blobData,
    commitments,
    account,
    blobName: file.name,
    expirationMicros,
  });

  return {
    hash: commitments.blob_merkle_root,
    name: file.name,
    size: blobData.byteLength,
    txHash: txResponse.hash,
  };
}
```

### 3. Listing Blobs

```javascript
async function listBlobs(walletAddress) {
  const client = new ShelbyBlobClient();
  const account = AccountAddress.from(walletAddress);
  const blobs = await client.coordination.getAccountBlobs({ account });

  return blobs.map(blob => ({
    name: blob.blobNameSuffix || blob.name || blob.blobName,
    hash: Buffer.from(blob.blobMerkleRoot).toString("hex"),
    size: blob.size,
    createdAt: Number(blob.creationMicros) / 1000,
    expiresAt: Number(blob.expirationMicros) / 1000,
  }));
}
```

### 4. Downloading Blobs

```javascript
async function downloadBlob(blobName, walletAddress) {
  const client = new ShelbyBlobClient();
  const account = AccountAddress.from(walletAddress);
  const data = await client.download({ account, blobName });
  return data; // Uint8Array
}

// Create downloadable URL in browser
function createBlobUrl(data, mimeType) {
  const blob = new Blob([data], { type: mimeType });
  return URL.createObjectURL(blob);
}
```

### 5. Fetching Transactions

```javascript
async function fetchTransactions(address, limit = 20) {
  const response = await fetch(
    `${SHELBY_FULLNODE}/accounts/${address}/transactions?limit=${limit}`
  );
  const txData = await response.json();

  return txData.map(tx => ({
    hash: tx.hash,
    version: tx.version,
    success: tx.success,
    gasUsed: tx.gas_used,
    timestamp: tx.timestamp,
    type: parseTransactionType(tx.payload?.function),
  }));
}

function parseTransactionType(fn) {
  if (!fn) return "unknown";
  if (fn.includes("register_blob")) return "upload";
  if (fn.includes("transfer")) return "transfer";
  if (fn.includes("mint")) return "mint";
  if (fn.includes("faucet")) return "faucet";
  return "other";
}
```

### 6. Cost Calculation

```javascript
function calculateUploadCost(sizeBytes) {
  const STORAGE_RATE_PER_GB_MONTH = 0.05; // $0.05 per GB per month
  const DURATION_MONTHS = 12;

  const sizeGB = sizeBytes / (1024 * 1024 * 1024);
  const totalCost = sizeGB * STORAGE_RATE_PER_GB_MONTH * DURATION_MONTHS;

  return {
    sizeGB,
    costPerGBMonth: STORAGE_RATE_PER_GB_MONTH,
    durationMonths: DURATION_MONTHS,
    totalCost,
    currency: "ShelbyUSD",
  };
}
```

## Required Dependencies

```json
{
  "@aspect-build/shelby-sdk": "latest",
  "@aptos-labs/ts-sdk": "^1.33.1",
  "@aptos-labs/wallet-adapter-react": "^3.x"
}
```

## Environment Variables

```bash
VITE_SHELBY_API_KEY=your_api_key    # Frontend SDK key
SHELBY_API_KEY=your_api_key          # Server-side SDK key
APTOS_NETWORK=testnet                # Network setting
```

## Common Patterns

### React Hook for Balances

```javascript
const useBalances = () => {
  const [apt, setApt] = useState(null);
  const [susd, setSusd] = useState(null);
  const { account, connected } = useWallet();

  useEffect(() => {
    if (connected && account?.address) {
      const addr = account.address.toString();
      fetchBalance(addr, "0xa").then(setApt);
      fetchBalance(addr, SHELBY_USD_METADATA).then(setSusd);
    }
  }, [connected, account?.address]);

  return { apt, susd };
};
```

### Error Handling

```javascript
try {
  const balance = await fetchBalance(address, metadata);
} catch (err) {
  if (err.message?.includes("RESOURCE_NOT_FOUND")) {
    return 0; // Account has no balance
  }
  throw err;
}
```

## Additional Resources

- [API-REFERENCE.md](API-REFERENCE.md) - Detailed endpoint documentation
- [EXAMPLES.md](EXAMPLES.md) - Complete code examples for React/Node.js

## Key Insights

1. **APT is a Fungible Asset on Shelbynet** - Don't look for CoinStore, use `primary_fungible_store::balance`
2. **All amounts use 8 decimals** - Divide raw values by 100,000,000
3. **Blobs expire automatically** - No delete operation needed; set expiration on upload
4. **Wallet signing required for uploads** - Downloads are read-only
5. **WebDAV provides filesystem access** - Mount blobs as a virtual drive (read-only)
