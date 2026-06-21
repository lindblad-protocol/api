# Lindblad Protocol — API Reference

**Base URL:** `https://lindblad.io`
**Version:** 2.1
**Networks:** Arbitrum One + Polygon (mainnet) · Arbitrum Sepolia + Robinhood Chain Testnet (testnet for M2M)

---

## Overview

The Lindblad VPS API provides access to the Spectral Ledger — the physics-anchored token transport layer of the Lindblad Protocol — and to the application layer above it (RWAFi tokenization, M2M Commerce, LindFi AMM).

All read endpoints are public and require no authentication.

---

## Table of contents

- [Chain & Blocks](#chain--blocks)
- [Accounts & Balances](#accounts--balances)
- [Nodes](#nodes)
- [Bridge](#bridge)
- [RWAFi & Energy Attestations](#rwafi--energy-attestations)
- [LindFi Pools](#lindfi-pools)
- [M2M Hardware API](#m2m-hardware-api) (runs on each node, not on lindblad.io)
- [M2M Escrow Contract](#m2m-escrow-contract) (Solidity ABI on testnet)
- [Node Address Format](#node-address-format)
- [Smart Contracts](#smart-contracts)
- [PYCO Token](#pyco-token)
- [Rate Limits](#rate-limits)

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
|-------|------|-------------|
| blockCount | integer | Total blocks in the Spectral Ledger |
| latestEpoch | integer | Most recent epoch number |
| totalMined | integer | Total PYCO mined (in microunits, divide by 1e6) |
| totalBurned | integer | Total PYCO burned (in microunits) |

### `GET /blocks`

Returns the last 50 blocks.

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

### `GET /token-balance/{LD-address}?token={TOKEN}&network={NETWORK}`

Returns the balance of a specific token (USDT, USDC, PYCO) on the specified network.

**Parameters:**
- `token` — `USDT`, `USDC`, or `PYCO`
- `network` — `arbitrum` or `polygon` (default: `arbitrum`)

### `POST /send-token`

Transfer tokens between LD addresses. Internal transfers are free and instant.

---

## Nodes

### `GET /nodes`

Returns all registered nodes with status.

**Response:**

```json
{
  "nodes": [
    {
      "nodeId": "LDxxxxxxx",
      "puf": "xxxxxxxxxxxxxxxx",
      "lastSeen": 1748200000,
      "active": true
    }
  ],
  "count": 4
}
```

| Field | Description |
|-------|-------------|
| nodeId | Short node ID (LD + 7 chars of PUF) |
| active | `true` if last seen < 120s ago |

### `GET /node-status?nodeId={id}`

Returns whether a specific node is currently online.

### `GET /device/nodes?puf={puf}`

Returns all nodes authorized for a given device PUF.

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

### `POST /request-sign`

Queues a hardware signing request for a node.

### `GET /get-sign?signId={id}`

Poll for the result of a hardware signing request.

| Status | Description |
|--------|-------------|
| `pending` | Node has not yet signed |
| `complete` | Signature ready — use r, s, v to call contract |
| `expired` | Challenge expired — request again |

### `POST /verify-pairing`

Verify a pairing code and authorize a device to a node.

---

## RWAFi & Energy Attestations

Endpoints for attesting real-world energy generation and minting native GREENKWH tokens. RWAFi (Real-World Asset Finance) is the architectural pattern of physical attestation → native minting → AMM markets, unified in a single layer with end-to-end cryptographic verification.

### `GET /attestations/energy?limit={n}`

Returns the most recent energy attestations.

**Response:**

```json
{
  "attestations": [
    {
      "id": "394289f25657",
      "source": "EIA",
      "respondent": "CAL",
      "kwh_total": 72000,
      "period": "2026-06-13T05",
      "node_id": "LDxxxxxxx",
      "mint_percentage_bps": 7500,
      "tokens_minted_display": 54000.0,
      "status": "active",
      "ts": 1781381824
    }
  ],
  "count": 1
}
```

### `POST /attestations/energy/submit`

Finalizes an attestation and mints GREENKWH tokens to the producer.

**Body:** `{signId, mintPercentage, recipient}`

- `mintPercentage` in basis points (0-10000, where 5000 = 50%)
- `recipient` is the LD-address that will receive the minted GREENKWH

### `POST /attestations/energy/recalibrate`

Increases the mint percentage of an existing attestation. Only upward recalibration is allowed (already-minted tokens cannot be retracted). Requires a fresh PUF signature from the same node that produced the original attestation, with a challenge of the form `recalibrate_<attestationId>_<newPct>_<timestamp>`.

### `GET /attestations/energy/recalibrations/{attestationId}`

Returns the recalibration history for a specific attestation (public audit trail).

### `GET /tokens/available`

Returns all tokens currently active in the system, grouped as `defi_tokens` (bridged or crypto-native) and `rwafi_tokens` (minted from physical attestations). Used by the LindFi UI to populate the Create Pool modal.

**Response:**

```json
{
  "defi_tokens": [
    {"token": "USDC", "total_supply_display": 8986.75, "origin": "bridged_or_native"},
    {"token": "USDT", "total_supply_display": 8798.96, "origin": "bridged_or_native"}
  ],
  "rwafi_tokens": [
    {"token": "GREENKWH", "total_supply_display": 53983.99, "origin": "physical_attestation"}
  ]
}
```

---

## LindFi Pools

AMM endpoints for liquidity pools. Pools are constant-product (`x·y=k`) with configurable fees in basis points. All operations that change reserves (add, remove, swap) require a PUF-signed authorization from a hardware node. Pools are automatically categorized as `defi` or `rwafi` based on the tokens involved.

### `GET /pools`

Returns all pools with reserves, LP supply, fee, price, and category. Sorted by liquidity descending.

### `GET /pool/{poolId}`

Returns full state of a specific pool.

**Example:**

```
GET /pool/GREENKWH-USDT
```

**Response:**

```json
{
  "pool_id": "GREENKWH-USDT",
  "token_a": "GREENKWH",
  "token_b": "USDT",
  "reserve_a": 16009606,
  "reserve_b": 2498500,
  "lp_supply": 6324555,
  "fee_bps": 30,
  "price_a_in_b": 0.156063,
  "price_b_in_a": 6.407687
}
```

### `POST /pool/create`

Body: `{tokenA, tokenB, ldAddress, feeBps}`. Category is detected automatically.

### `POST /pool/add` — Add liquidity (PUF-signed)

Body: `{poolId, ldAddress, amountA, amountB, signId}`.

### `POST /pool/remove` — Remove liquidity (PUF-signed)

Body: `{poolId, ldAddress, lpAmount, signId}`.

### `POST /pool/swap` — Swap tokens (PUF-signed)

Body: `{poolId, ldAddress, tokenIn, amountIn, signId}`. Response includes `level2_verified`.

### `GET /pool-lp/{poolId}?address={addr}`

Returns the LP shares held by a specific address.

### `GET /pool-swaps/{poolId}?limit={n}`

Returns the most recent swaps in a pool.

### `GET /pool-apr/{poolId}?hours={n}`

Returns an estimated APR based on swap fees collected in the last N hours.

---

## M2M Hardware API

Endpoints served directly by Lindblad hardware nodes (e.g., Heltec V3 running the Lindblad firmware). These run locally on the device's HTTP server, typically over the LAN at `http://<node-ip>`. All cryptographic operations are performed inside the node — keys are derived from PUF in real time and never leave the silicon.

### `GET /api/challenge`

Returns a unique challenge derived from PUF + Chua HSC oscillator + timestamp. Each call produces a different value — non-replayable by design.

```bash
curl http://<node-ip>/api/challenge
```

**Response:**

```json
{
  "challenge": "37f5b4ca",
  "nodeId": "LDxxxxxxx",
  "ts": 1781911619,
  "pubKey": "04..."
}
```

The Ethereum-compatible address can be derived as `keccak256(pubKey[1:])[-20:]`.

### `POST /api/sign`

Signs `SHA256(challenge || nodeId || timestamp || chuaNonce)` with the node's ECDSA P-256 private key (derived in real time from PUF via BCH fuzzy extractor). The signature is EVM-compatible and verifiable on any chain.

Body: `{challenge}`. Use `Content-Type: text/plain` to bypass browser CORS preflight (the firmware reads raw body).

**Response:**

```json
{
  "ecSignature": "...",
  "chuaNonce": "83897fc3",
  "pubKey": "04..."
}
```

### `GET /api/status`

Returns the node's identity, network info, mining status, and PUF coherence indicators.

---

## M2M Escrow Contract

The M2MEscrow contract enforces the M2M Commerce lifecycle on EVM-compatible chains. Source code is published in [github.com/lindblad-protocol/contracts](https://github.com/lindblad-protocol/contracts) under MIT License.

**Deployments:**
- Arbitrum Sepolia: `0xdeaED8e809733667D80a8E6ca40A02366598CA60`
- Robinhood Chain Testnet: `0x16a69CcdA3865a23537d46055dC6564A2813C36B`

### Core functions

| Function | Type | Description |
|----------|------|-------------|
| `isNodeRegistered(address)` | view | Check if a node is authorized |
| `getContract(uint256)` | view | Read full contract state |
| `getContractState(uint256)` | view | Read lifecycle state only |
| `requestService(provider, units, pricePerUnit, maxAmount, deadline, serviceType)` | call | Client requests service, locks payment in escrow |
| `acceptService(contractId)` | call | Provider accepts the request |
| `attestDelivery(contractId, deliveredAmount, attestationHash)` | call | Provider records delivery progress |
| `settlePayment(contractId)` | call | Manual settlement after deadline |

### Lifecycle states

```
Requested → Accepted → InProgress → Delivered → Settled
                                ↘ Cancelled
```

### Verifying a node's identity end-to-end

```javascript
// 1. Fetch challenge from hardware node
const ch = await fetch(`http://<node-ip>/api/challenge`).then(r => r.json());

// 2. Derive Ethereum address from pubKey
const pubKeyHex = "0x" + ch.pubKey.slice(2);  // strip '04' prefix
const hash = ethers.keccak256(pubKeyHex);
const ethAddress = ethers.getAddress("0x" + hash.slice(-40));

// 3. Query M2MEscrow on Arbitrum Sepolia
const isAuthorized = await m2mContract.isNodeRegistered(ethAddress);
```

A full reference implementation is available in the [contracts repository](https://github.com/lindblad-protocol/contracts).

---

## Node Address Format

Every participant on the Spectral Ledger is identified by an LDXXXXXXX address:

```
Full PUF:     xxxxxxxxxxxxxxxx   (16 hex chars, from SRAM PUF)
Short ID:     LDxxxxxxx          (LD + first 7 chars)
Full address: LD-xxxxxxxxxxxxxxxx (used in API calls)
```

- Short ID is used for display
- Full address (`LD-{PUF}`) is used in all API calls
- Each address is physically unique — generated by silicon at manufacture

---

## Smart Contracts

### Production — Arbitrum One (Chain ID: 42161)

| Contract | Address |
|----------|---------|
| LindblabUSDT v3 | `0x7e0f53f04dDc48dFdc96DFE93606a73f0dCF56A3` |
| LindblabUSDC v3 | `0x1AfC80b30cBBE50E8aBb4585f53ff530c305d416` |
| PYCO ERC-20 | `0x16a69CcdA3865a23537d46055dC6564A2813C36B` |
| Real USDT | `0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9` |
| Real USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` |

### Production — Polygon (Chain ID: 137)

| Contract | Address |
|----------|---------|
| LindblabUSDT v3 | `0x17c6d525A8D809fcBe78aBE4FCaE1F9ddb0b8fa8` |
| LindblabUSDC v3 | `0x9964c63Af739bf8b4702E243f904570b17F33ab4` |
| Real USDT | `0xc2132D05D31c914a87C6611C10748AEb04B58e8F` |
| Real USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` |

### Testnet — Arbitrum Sepolia (Chain ID: 421614)

| Contract | Address |
|----------|---------|
| M2MEscrow | `0xdeaED8e809733667D80a8E6ca40A02366598CA60` |
| MockUSDC | `0xa6Ee2f4248b447f934Aabf44aA534C6C21654F6c` |

### Testnet — Robinhood Chain Testnet (Chain ID: 46630)

| Contract | Address |
|----------|---------|
| M2MEscrow | `0x16a69CcdA3865a23537d46055dC6564A2813C36B` |

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
- Protocol: [lindblad.io/protocol](https://lindblad.io/protocol)
- RWAFi: [lindblad.io/rwafi](https://lindblad.io/rwafi)
- M2M Commerce: [lindblad.io/m2m](https://lindblad.io/m2m)
- Block Explorer: [lindblad.io/scan](https://lindblad.io/scan)
- Wallet: [lindblad.io/wallet](https://lindblad.io/wallet)
- Documentation: [lindblad.io/docs](https://lindblad.io/docs)
- GitHub: [github.com/lindblad-protocol](https://github.com/lindblad-protocol)

---

*Lindblad Protocol — The hardware decides. The physics guarantees.*
