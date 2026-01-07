# Shelby Code Examples

Complete working examples for common Shelby operations.

## Table of Contents

1. [React: Complete Wallet Integration](#react-complete-wallet-integration)
2. [React: File Upload Component](#react-file-upload-component)
3. [React: Blob Gallery with Downloads](#react-blob-gallery-with-downloads)
4. [Node.js: Server-Side Blob Service](#nodejs-server-side-blob-service)
5. [Vanilla JS: Balance Fetching](#vanilla-js-balance-fetching)

---

## React: Complete Wallet Integration

### WalletProvider.jsx

```jsx
import { AptosWalletAdapterProvider } from "@aptos-labs/wallet-adapter-react";
import { AptosConnectWallet } from "@aspect-build/aptos-connect-wallet-adapter";
import { Network } from "@aptos-labs/ts-sdk";

const wallets = [
  new AptosConnectWallet({
    network: Network.TESTNET,
    dappId: "your-dapp-id",
    dappName: "Your App Name",
  }),
];

export function WalletProvider({ children }) {
  return (
    <AptosWalletAdapterProvider plugins={wallets} autoConnect={true}>
      {children}
    </AptosWalletAdapterProvider>
  );
}
```

### GuestContext.jsx (Balance Management)

```jsx
import { createContext, useContext, useState, useEffect } from "react";
import { useWallet } from "@aptos-labs/wallet-adapter-react";

const SHELBY_FULLNODE = "https://api.shelbynet.shelby.xyz/v1";
const APT_METADATA = "0xa";
const SHELBY_USD_METADATA = "0x1b18363a9f1fe5e6ebf247daba5cc1c18052bb232efdc4c50f556053922d98e1";

const GuestContext = createContext();

export function GuestProvider({ children }) {
  const [balance, setBalance] = useState(null);
  const [shelbyUsdBalance, setShelbyUsdBalance] = useState(null);
  const [balanceLoading, setBalanceLoading] = useState(false);
  const { account, connected } = useWallet();

  const fetchTokenBalance = async (address, metadata) => {
    const payload = {
      function: "0x1::primary_fungible_store::balance",
      type_arguments: ["0x1::fungible_asset::Metadata"],
      arguments: [address, metadata],
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
  };

  const refetchBalance = async () => {
    if (!connected || !account?.address) return;
    setBalanceLoading(true);

    const addr = account.address.toString();

    try {
      const [apt, susd] = await Promise.all([
        fetchTokenBalance(addr, APT_METADATA),
        fetchTokenBalance(addr, SHELBY_USD_METADATA),
      ]);
      setBalance(apt);
      setShelbyUsdBalance(susd);
    } catch (err) {
      console.error("Failed to fetch balances:", err);
    }

    setBalanceLoading(false);
  };

  useEffect(() => {
    if (connected && account?.address) {
      refetchBalance();
    }
  }, [connected, account?.address]);

  return (
    <GuestContext.Provider
      value={{
        wallet: {
          address: account?.address?.toString() || null,
          balance,
          isConnected: connected,
        },
        shelbyUsdBalance,
        balanceLoading,
        refetchBalance,
      }}
    >
      {children}
    </GuestContext.Provider>
  );
}

export const useGuest = () => useContext(GuestContext);
```

---

## React: File Upload Component

### useShelbyUpload.js

```jsx
import { useState, useCallback } from "react";
import { useWallet } from "@aptos-labs/wallet-adapter-react";
import { ShelbyBlobClient, expectedTotalChunksets } from "@aspect-build/shelby-sdk";
import { AccountAddress, Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

const aptos = new Aptos(new AptosConfig({ network: Network.TESTNET }));

export function useShelbyUpload() {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);
  const [error, setError] = useState(null);
  const { account, connected, signAndSubmitTransaction } = useWallet();

  const uploadFile = useCallback(async (file) => {
    if (!connected || !account?.address) {
      throw new Error("Wallet not connected");
    }

    setUploading(true);
    setProgress(0);
    setError(null);

    try {
      const client = new ShelbyBlobClient();
      const blobData = new Uint8Array(await file.arrayBuffer());
      const accountAddr = AccountAddress.from(account.address.toString());

      setProgress(10);

      // Generate commitments
      const commitments = client.commitments(blobData);
      setProgress(30);

      // Calculate expiration (365 days)
      const expirationMicros = BigInt(Date.now() + 365 * 24 * 60 * 60 * 1000) * 1000n;

      // Create and submit registration transaction
      const payload = client.createRegisterBlobPayload({
        account: accountAddr,
        blobName: file.name,
        blobSize: commitments.raw_data_size,
        blobMerkleRoot: commitments.blob_merkle_root,
        expirationMicros,
        numChunksets: expectedTotalChunksets(commitments.raw_data_size),
      });

      setProgress(50);

      const txResponse = await signAndSubmitTransaction({ data: payload });
      await aptos.waitForTransaction({ transactionHash: txResponse.hash });

      setProgress(70);

      // Upload blob data
      await client.rpc.putBlob({
        data: blobData,
        commitments,
        account: accountAddr,
        blobName: file.name,
        expirationMicros,
      });

      setProgress(100);

      return {
        hash: Buffer.from(commitments.blob_merkle_root).toString("hex"),
        name: file.name,
        size: blobData.byteLength,
        txHash: txResponse.hash,
      };
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setUploading(false);
    }
  }, [connected, account, signAndSubmitTransaction]);

  return { uploadFile, uploading, progress, error };
}
```

### FileUploader.jsx

```jsx
import { useCallback } from "react";
import { useDropzone } from "react-dropzone";
import { useShelbyUpload } from "../hooks/useShelbyUpload";

export function FileUploader({ onUploadComplete }) {
  const { uploadFile, uploading, progress, error } = useShelbyUpload();

  const onDrop = useCallback(async (acceptedFiles) => {
    for (const file of acceptedFiles) {
      try {
        const result = await uploadFile(file);
        onUploadComplete?.(result);
      } catch (err) {
        console.error("Upload failed:", err);
      }
    }
  }, [uploadFile, onUploadComplete]);

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    disabled: uploading,
  });

  return (
    <div
      {...getRootProps()}
      className={`border-2 border-dashed rounded-lg p-8 text-center cursor-pointer
        ${isDragActive ? "border-blue-500 bg-blue-50" : "border-gray-300"}
        ${uploading ? "opacity-50 cursor-not-allowed" : ""}`}
    >
      <input {...getInputProps()} />

      {uploading ? (
        <div>
          <p>Uploading... {progress}%</p>
          <div className="w-full bg-gray-200 rounded-full h-2 mt-2">
            <div
              className="bg-blue-500 h-2 rounded-full transition-all"
              style={{ width: `${progress}%` }}
            />
          </div>
        </div>
      ) : (
        <p>
          {isDragActive
            ? "Drop files here..."
            : "Drag & drop files, or click to select"}
        </p>
      )}

      {error && <p className="text-red-500 mt-2">{error}</p>}
    </div>
  );
}
```

---

## React: Blob Gallery with Downloads

### useShelbyBlobs.js

```jsx
import { useState, useCallback, useEffect } from "react";
import { useWallet } from "@aptos-labs/wallet-adapter-react";
import { ShelbyBlobClient } from "@aspect-build/shelby-sdk";
import { AccountAddress } from "@aptos-labs/ts-sdk";

const MIME_TYPES = {
  jpg: "image/jpeg", jpeg: "image/jpeg", png: "image/png",
  gif: "image/gif", webp: "image/webp", svg: "image/svg+xml",
  mp4: "video/mp4", webm: "video/webm", mp3: "audio/mpeg",
  pdf: "application/pdf", json: "application/json", txt: "text/plain",
};

function getMimeType(filename) {
  const ext = filename.split(".").pop()?.toLowerCase();
  return MIME_TYPES[ext] || "application/octet-stream";
}

export function useShelbyBlobs() {
  const [blobs, setBlobs] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const { account, connected } = useWallet();

  const fetchBlobs = useCallback(async () => {
    if (!connected || !account?.address) return;

    setLoading(true);
    setError(null);

    try {
      const client = new ShelbyBlobClient();
      const accountAddr = AccountAddress.from(account.address.toString());
      const blobList = await client.coordination.getAccountBlobs({ account: accountAddr });

      const parsed = blobList.map((blob) => ({
        id: Buffer.from(blob.blobMerkleRoot).toString("hex"),
        name: blob.blobNameSuffix || blob.name || blob.blobName,
        hash: Buffer.from(blob.blobMerkleRoot).toString("hex"),
        size: Number(blob.size),
        mimeType: getMimeType(blob.blobNameSuffix || blob.name || ""),
        createdAt: Number(blob.creationMicros) / 1000,
        expiresAt: Number(blob.expirationMicros) / 1000,
      }));

      setBlobs(parsed);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [connected, account]);

  const downloadBlob = useCallback(async (blob) => {
    const client = new ShelbyBlobClient();
    const accountAddr = AccountAddress.from(account.address.toString());
    const data = await client.download({ account: accountAddr, blobName: blob.name });

    const blobObj = new Blob([data], { type: blob.mimeType });
    const url = URL.createObjectURL(blobObj);

    // Trigger download
    const a = document.createElement("a");
    a.href = url;
    a.download = blob.name;
    a.click();

    URL.revokeObjectURL(url);
    return data;
  }, [account]);

  useEffect(() => {
    if (connected) {
      fetchBlobs();
    }
  }, [connected, fetchBlobs]);

  return { blobs, loading, error, fetchBlobs, downloadBlob };
}
```

### BlobGallery.jsx

```jsx
import { useShelbyBlobs } from "../hooks/useShelbyBlobs";

function formatBytes(bytes) {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}

export function BlobGallery() {
  const { blobs, loading, error, fetchBlobs, downloadBlob } = useShelbyBlobs();

  if (loading) return <div>Loading blobs...</div>;
  if (error) return <div className="text-red-500">Error: {error}</div>;

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h2 className="text-xl font-bold">Your Files</h2>
        <button onClick={fetchBlobs} className="btn-secondary">
          Refresh
        </button>
      </div>

      {blobs.length === 0 ? (
        <p className="text-gray-500">No files uploaded yet.</p>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
          {blobs.map((blob) => (
            <div key={blob.id} className="border rounded-lg p-4">
              <h3 className="font-medium truncate">{blob.name}</h3>
              <p className="text-sm text-gray-500">{formatBytes(blob.size)}</p>
              <p className="text-xs text-gray-400">
                Expires: {new Date(blob.expiresAt).toLocaleDateString()}
              </p>
              <button
                onClick={() => downloadBlob(blob)}
                className="mt-2 text-blue-500 hover:underline"
              >
                Download
              </button>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## Node.js: Server-Side Blob Service

### shelby.service.js

```javascript
import { ShelbyBlobClient } from "@aspect-build/shelby-sdk";
import { AccountAddress } from "@aptos-labs/ts-sdk";

class ShelbyService {
  constructor() {
    this.client = new ShelbyBlobClient();
  }

  async listFiles(walletAddress) {
    const account = AccountAddress.from(walletAddress);
    const blobs = await this.client.coordination.getAccountBlobs({ account });

    return blobs.map((blob) => ({
      name: blob.blobNameSuffix || blob.name,
      hash: Buffer.from(blob.blobMerkleRoot).toString("hex"),
      size: Number(blob.size),
      createdAt: Number(blob.creationMicros) / 1000,
      expiresAt: Number(blob.expirationMicros) / 1000,
    }));
  }

  async downloadFile(blobName, walletAddress) {
    const account = AccountAddress.from(walletAddress);
    const data = await this.client.download({ account, blobName });
    return data;
  }

  async getFileStream(blobName, walletAddress) {
    const data = await this.downloadFile(blobName, walletAddress);
    const { Readable } = await import("stream");

    return {
      stream: Readable.from(Buffer.from(data)),
      contentLength: data.byteLength,
      name: blobName,
    };
  }
}

export const shelbyService = new ShelbyService();
```

### Express Routes

```javascript
import express from "express";
import { shelbyService } from "./services/shelby.service.js";

const router = express.Router();

router.get("/files", async (req, res) => {
  const walletAddress = req.headers["x-wallet-address"];

  if (!walletAddress) {
    return res.status(401).json({ error: "Wallet address required" });
  }

  try {
    const files = await shelbyService.listFiles(walletAddress);
    res.json(files);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

router.get("/files/:blobName/download", async (req, res) => {
  const { blobName } = req.params;
  const walletAddress = req.headers["x-wallet-address"];

  try {
    const { stream, contentLength, name } = await shelbyService.getFileStream(
      blobName,
      walletAddress
    );

    res.setHeader("Content-Disposition", `attachment; filename="${name}"`);
    res.setHeader("Content-Length", contentLength);
    stream.pipe(res);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

export default router;
```

---


## Vanilla JS: Balance Fetching

### For use without frameworks

```html
<!DOCTYPE html>
<html>
<head>
  <title>Shelby Balance Checker</title>
</head>
<body>
  <input type="text" id="address" placeholder="Wallet address (0x...)" />
  <button onclick="checkBalance()">Check Balance</button>
  <div id="result"></div>

  <script>
    const SHELBY_FULLNODE = "https://api.shelbynet.shelby.xyz/v1";
    const APT_METADATA = "0xa";
    const SUSD_METADATA = "0x1b18363a9f1fe5e6ebf247daba5cc1c18052bb232efdc4c50f556053922d98e1";

    async function fetchBalance(address, metadata) {
      const response = await fetch(`${SHELBY_FULLNODE}/view`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          function: "0x1::primary_fungible_store::balance",
          type_arguments: ["0x1::fungible_asset::Metadata"],
          arguments: [address, metadata],
        }),
      });

      const result = await response.json();
      return Array.isArray(result) ? Number(result[0]) / 1e8 : 0;
    }

    async function checkBalance() {
      const address = document.getElementById("address").value;
      const resultDiv = document.getElementById("result");

      if (!address) {
        resultDiv.textContent = "Please enter a wallet address";
        return;
      }

      resultDiv.textContent = "Loading...";

      try {
        const [apt, susd] = await Promise.all([
          fetchBalance(address, APT_METADATA),
          fetchBalance(address, SUSD_METADATA),
        ]);

        resultDiv.innerHTML = `
          <p><strong>APT Balance:</strong> ${apt.toFixed(4)} APT</p>
          <p><strong>ShelbyUSD Balance:</strong> ${susd.toFixed(4)} SUSD</p>
        `;
      } catch (err) {
        resultDiv.textContent = `Error: ${err.message}`;
      }
    }
  </script>
</body>
</html>
```

---

## Cost Calculator

```javascript
function calculateStorageCost(sizeBytes) {
  const RATE_PER_GB_MONTH = 0.05; // $0.05
  const DURATION_MONTHS = 12;

  const sizeGB = sizeBytes / (1024 ** 3);
  const totalCost = sizeGB * RATE_PER_GB_MONTH * DURATION_MONTHS;

  return {
    sizeGB: sizeGB.toFixed(6),
    monthlyRate: RATE_PER_GB_MONTH,
    duration: DURATION_MONTHS,
    totalCost: totalCost.toFixed(6),
    currency: "ShelbyUSD",
    formatted: `${totalCost.toFixed(4)} SUSD for ${DURATION_MONTHS} months`,
  };
}

// Example
console.log(calculateStorageCost(1024 * 1024 * 100)); // 100 MB file
// { sizeGB: "0.095367", totalCost: "0.057220", ... }
```
