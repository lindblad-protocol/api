# Lindblad Protocol — API Reference

**Base URL:** `https://lindblad.io`  
**Version:** 2.0  
**Networks:** Arbitrum One + Polygon (mainnet)

---

## Overview

The Lindblad VPS API provides access to the Spectral Ledger — the physics-anchored token transport layer of the Lindblad Protocol. All endpoints are public and require no authentication for read operations.

---

## Chain & Blocks

### `GET /chain`
Returns a summary of the current chain state.

**Response:**
```json
{
  "blockCount": 9534,
  "latestEpoch": 19499,
  "earliestEpoch": 3,
  "digest": "a0e8c661",
  "totalMined": 663169812500,
  "totalBurned": 0
}
```

| Field | Type | Description |
|---|---|---|
| blockCount | integer | Total blocks in the Spectral Ledger |
| latestEpoch | integer | Most recent epoch number |
| totalMined | integer | Total PYCO mined (in microunits, divide by 1e6) |
| totalBurned | integer | Total PYCO burned (in microunits) |

---

### `GET /blocks`
Returns the last 50 blocks.

**Response:**
```json
{
  "blocks": [
    {
      "epoch": 19499,
      "digest": "a0e8c661",
      "nodeCount": 4,
      "txCount": 0,
      "timestamp": 1748200000
    }
  ]
}
```

---

### `GET /block/latest`
Returns the most recent block with full attestation data.

---

## Accounts & Balances

### `GET /account/{LD-address}`
Returns the PYCO balance for a given LD address from the Spectral Ledger.

**Example:**
```
GET /account/LD-EC6C1139FA3CD393
```

**Response:**
```json
{
  "address": "LD-EC6C1139FA3CD393",
  "balance": 299036.99
}
```

---

### `GET /token-balance/{LD-address}?token={TOKEN}&network={NETWORK}`
Returns the balance of a specific token (USDT, USDC, PYCO) on the specified network.

**Parameters:**
- `token` — `USDT`, `USDC`, or `PYCO`
- `network` — `arbitrum` or `polygon` (default: `arbitrum`)

**Example:**
```
GET /token-balance/LD-EC6C1139FA3CD393?token=USDT&network=polygon
```

**Response:**
```json
{
  "address": "LD-EC6C1139FA3CD393",
  "token": "USDT",
  "network": "polygon",
  "balance": 30000000
}
```

> Note: Balances are returned in microunits. Divide by 1,000,000 for display value.

---

### `POST /send-token`
Transfer tokens between LD addresses. Internal transfers are free and instant.

**Body:**
```json
{
  "from": "LD-EC6C1139FA3CD393",
  "to": "LD-95A821D9AC091073",
  "token": "USDT",
  "amount": 10000000,
  "sig": "..."
}
```

**Response:**
```json
{
  "ok": true,
  "txId": "abc123"
}
```

---

## Nodes

### `GET /nodes`
Returns all registered nodes with status.

**Response:**
```json
{
  "nodes": [
    {
      "id": "LD95A821D",
      "puf": "95A821D9xxxxxxxx",
      "lastSeen": 1748200000,
      "status": "active",
      "lastEpoch": 19499
    }
  ],
  "count": 4
}
```

| Field | Description |
|---|---|
| id | Short node ID (LD + 7 chars of PUF) |
| status | `active` = last seen < 120s ago, `idle` = otherwise |
| lastEpoch | Most recent epoch this node submitted |

---

### `GET /node-status?nodeId={id}`
Returns whether a specific node is currently online.

**Example:**
```
GET /node-status?nodeId=LD95A821D
```

**Response:**
```json
{
  "nodeId": "LD95A821D",
  "online": true,
  "lastSeen": 1748200000
}
```

---

### `GET /device/nodes?puf={puf}`
Returns all nodes authorized for a given device PUF.

**Example:**
```
GET /device/nodes?puf=EC6C1139FA3CD393
```

**Response:**
```json
{
  "nodes": [
    {
      "nodeId": "LD95A821D",
      "nodePuf": "95A821D9xxxxxxxx",
      "ip": "192.168.x.x",
      "lastEpoch": 19499
    }
  ],
  "count": 1
}
```

---

## Bridge

### `GET /node-challenge?nodeId={id}`
Returns a signing challenge for a node. Used before bridge withdrawals.

**Response:**
```json
{
  "challenge": "a3f8c2...",
  "expiresAt": 1748200060
}
```

> Challenge expires in 60 seconds.

---

### `POST /request-sign`
Queues a hardware signing request for a node.

**Body:**
```json
{
  "nodeId": "LD95A821D",
  "challenge": "a3f8c2...",
  "amount": 10000000,
  "token": "USDT",
  "network": "polygon",
  "toAddress": "0xYourAddress"
}
```

**Response:**
```json
{
  "ok": true,
  "signId": "sign_abc123"
}
```

---

### `GET /get-sign?signId={id}`
Poll for the result of a hardware signing request.

**Response:**
```json
{
  "status": "complete",
  "r": "0x...",
  "s": "0x...",
  "v": 27
}
```

| Status | Description |
|---|---|
| `pending` | Node has not yet signed |
| `complete` | Signature ready — use r, s, v to call contract |
| `expired` | Challenge expired — request again |

---

### `POST /verify-pairing`
Verify a pairing code and authorize a device to a node.

**Body:**
```json
{
  "nodeId": "LD95A821D",
  "code": "KERX84",
  "ldAddr": "LD-EC6C1139FA3CD393"
}
```

**Response:**
```json
{
  "ok": true,
  "nodeId": "LD95A821D",
  "puf": "95A821D9xxxxxxxx"
}
```

---

## Node Address Format

Every participant on the Spectral Ledger is identified by an LDXXXXXXX address:

```
Full PUF:     95A821D9AC091073   (16 hex chars, from SRAM PUF)
Short ID:     LD95A821D          (LD + first 7 chars)
Full address: LD-95A821D9AC091073 (used in API calls)
```

- Short ID is used for display
- Full address (`LD-{PUF}`) is used in all API calls
- Each address is physically unique — generated by silicon at manufacture

---

## Smart Contracts

### Arbitrum One (Chain ID: 42161)

| Contract | Address |
|---|---|
| LindblabUSDT v3 | `0x7e0f53f04dDc48dFdc96DFE93606a73f0dCF56A3` |
| LindblabUSDC v3 | `0x1AfC80b30cBBE50E8aBb4585f53ff530c305d416` |
| PYCO ERC-20 | `0x16a69CcdA3865a23537d46055dC6564A2813C36B` |
| Real USDT | `0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9` |
| Real USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` |

### Polygon (Chain ID: 137)

| Contract | Address |
|---|---|
| LindblabUSDT v3 | `0xDcc707CAe072A4B55678355b75ABD76489bf6985` |
| LindblabUSDC v3 | `0x9964c63Af739bf8b4702E243f904570b17F33ab4` |
| Real USDT | `0xc2132D05D31c914a87C6611C10748AEb04B58e8F` |
| Real USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` |

> PYCO remains on Arbitrum One only. It is the native token of the entire Lindblad network regardless of which chain the bridge deposit originates from.

---

## PYCO Token

- **Max supply:** 100,000,000 PYCO
- **Bridge fee:** 0.1% of withdrawal amount
- **Fee split:** 50% burned permanently, 50% to active node operators
- **Internal transfers:** Always free
- **Rewards:** Distributed by Physical Coherence Verification Score (PCV-4)

---

## Rate Limits

The API is currently open with no enforced rate limits. Please be considerate with polling frequency — recommended interval is 30 seconds for live data.

---

## Links

- Website: [lindblad.io](https://lindblad.io)
- Wallet: [lindblad.io/wallet](https://lindblad.io/wallet)
- Block Explorer: [lindblad.io/scan](https://lindblad.io/scan)
- GitHub: [github.com/lindblad-protocol](https://github.com/lindblad-protocol)

---

*Lindblad Protocol — The hardware decides. The physics guarantees.*
