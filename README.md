# Shelby Skills for Claude Code

A Claude Code plugin providing skills for building on [Shelby](https://shelby.xyz), a decentralized storage network built on the Aptos blockchain.

## Installation

### Step 1: Add the marketplace

```bash
/plugin marketplace add codehakase/shelby-skills
```

### Step 2: Install the plugin

```bash
/plugin install shelby-blockchain@shelby-skills
```

### Verify installation

```bash
/plugin list
```

You should see `shelby-blockchain` in your installed plugins.

## What's Included

### shelby-blockchain skill

Comprehensive guidance for Shelby operations:

| Operation | Description |
|-----------|-------------|
| **Balance Fetching** | APT & ShelbyUSD via fungible asset view functions |
| **File Upload** | Commitments, blob registration, wallet signing |
| **File Download** | SDK download, browser blob URLs |
| **Blob Listing** | Account blobs with metadata parsing |
| **Transactions** | Fetching & parsing transaction history |
| **Cost Calculation** | Storage pricing ($0.05/GB/month) |

## Usage

Once installed, Claude Code will automatically use this skill when you ask about:

- "Upload files to Shelby"
- "Check APT balance on Shelbynet"
- "Download blobs from Shelby"
- "Build a Shelby storage app"
- "Fetch ShelbyUSD balance"

## Key Information

**Network:** Shelbynet (testnet)
```
https://api.shelbynet.shelby.xyz/v1
```

**Token Metadata (Fungible Assets):**
- APT: `0xa`
- ShelbyUSD: `0x1b18363a9f1fe5e6ebf247daba5cc1c18052bb232efdc4c50f556053922d98e1`

**Important:** On Shelbynet, APT is stored as a Fungible Asset, not in legacy CoinStore format.

## Dependencies

Projects using this skill typically need:

```json
{
  "@aspect-build/shelby-sdk": "latest",
  "@aptos-labs/ts-sdk": "^1.33.1",
  "@aptos-labs/wallet-adapter-react": "^3.x"
}
```

## License

MIT
