# Cross-Chain Attestations — WayChain Oracle Extension v1.0

WayChain's oracle attesters don't just serve the chain they run on.
They can witness events on other chains and re-anchor them on WayChain,
making WayChain's attested data available to any consumer.

---

## 0. Why Another Chain Would Trust WayChain

Every oracle network claims to be trustless. Most are anonymous.

| Property | Chainlink / Pyth | WayChain |
|----------|-----------------|----------|
| Node operator identity | Anonymous | Dox_Dev verified human |
| Slashing for lying | Economic only | Economic + badge revocation |
| Recovery after slashing | New wallet, new node | Cannot re-verify |
| Attestation finality | Depends on source chain | WayChain's 1s finality |
| Fee model | Market-driven (expensive) | Fiat-pegged (~$0.001) |

**A WayChain oracle attestation is signed by a known human.**
If they lie, they lose their badge permanently. They cannot spin up
a new identity. This is the fundamental difference.

---

## 1. How It Works

### 1.1 The Witness Flow

```
Source Chain (e.g. Ethereum, PulseChain):
  ┌─────────────────┐
  │ Event:           │
  │ Transfer(0xA→0xB)│
  │ at block 19,204,731│
  └────────┬────────┘
           │
           ▼ (WayChain attester observes via light client / RPC)
  ┌─────────────────────────────────┐
  │ Attester validates:              │
  │ 1. Block is finalized            │
  │ 2. Receipt matches event         │
  │ 3. No reorg risk (deep enough)   │
  └────────┬────────────────────────┘
           │
           ▼ (Attester submits to WayChain oracle lane)
  ┌────────────────────────────────────────────┐
  │ OracleLane.witnessEvent(                   │
  │   sourceChain: "ethereum",                  │
  │   blockNumber: 19204731,                    │
  │   txHash: 0xabcd...,                        │
  │   eventSignature: "Transfer(address,address,uint256)", │
  │   eventData: (0xA, 0xB, 1000),              │
  │   attestationProof: <attester signature>    │
  │ )                                           │
  └────────────────────────────────────────────┘
           │
           ▼ (Multiple attesters may witness the same event)
  ┌─────────────────────────────────────┐
  │ Event accretes attestations:         │
  │ Attester 1 (Dox_Dev: 0x1...) ✅     │
  │ Attester 2 (Dox_Dev: 0x2...) ✅     │
  │ Attester 3 (Dox_Dev: 0x3...) ✅     │
  │ → Confidence = 3 of N attesters     │
  └─────────────────────────────────────┘
```

### 1.2 Data Structure

```solidity
struct CrossChainAttestation {
    bytes32 sourceChain;          // Chain identifier (keccak256 of chain name)
    uint256 sourceBlockNumber;     // Block where event occurred
    bytes32 sourceTxHash;         // Transaction hash on source chain
    bytes32 eventSignature;       // keccak256 of event signature
    bytes eventData;             // ABI-encoded event parameters
    uint256 firstAttestedBlock;   // WayChain block when first attestation landed
    uint256 attestationCount;     // Number of attesters who confirmed
    uint256 confidence;           // 0-100% based on graduated trust (see oracle spec)
}
```

### 1.3 Graduated Trust

Not all attestations are equal. The more attesters, the higher the confidence.

| Attesters | Confidence Level | Use Case |
|-----------|-----------------|----------|
| 1 | Low | Low-value events, monitoring, non-critical |
| 3 | Medium | Cross-chain transfers, moderate value |
| 5 | High | Bridge operations, high-value verification |
| 10+ | Max | Critical infrastructure, governance proofs |

The consumer decides what confidence threshold to accept. A bridge may require
5+ attesters. A price feed may accept 3. An archivist may accept 1.

---

## 2. Economic Model

### 2.1 Fees

| Action | Cost (USD) | Paid To |
|--------|-----------|---------|
| Witness an event | ~$0.001 per attester | Attester (100%) |
| Query an attestation | Free (read from WayChain state) | — |
| Challenge a false attestation | ~$0.01 (bounty bond) | Challenger (if successful) |

**Witness fee is fiat-pegged,** same as WayChain gas. An attester who witnesses
an event earns $0.001 per event. If 10 attesters witness the same event,
the total cost is $0.01. Still cheaper than Chainlink.

### 2.2 Attester Economics

| Parameter | Value |
|-----------|-------|
| Minimum attester bond | 100 WAY per feed |
| Maximum feeds per attester | 5 (prevent concentration) |
| Earning per attestation | ~$0.001 in WAY |
| Slashing for false attestation | Bond + badge revocation |
| Challenge window | 100 WayChain blocks (~100 seconds) |

An attester who witnesses 1,000 events/day earns ~$1/day. At 5 feeds,
that's $5/day — meaningful side income for a verified operator.

### 2.3 The Challenge Game

If a consumer believes an attestation is false:

1. Post a challenge bond ($0.01 in WAY)
2. The attester must prove the event exists on the source chain
3. If the attester cannot prove it (event doesn't exist, block never happened):
   - Attester is slashed (bond + badge revocation)
   - Challenger receives 50% of the slashed amount
   - 50% is burned
4. If the attester proves the event exists:
   - Challenger loses their bond
   - Attester receives the bond

The challenge window is short (100 blocks) so attestations settle quickly
and consumers can rely on them.

---

## 3. What Can Be Attested

### 3.1 Event Types

| Event Type | Example | Typical Attesters Required |
|-----------|---------|--------------------------|
| **Token transfer** | ERC-20 Transfer on Ethereum | 3 |
| **Bridge deposit** | Lock event on bridge contract | 5+ |
| **Staking event** | Validator stake change | 3 |
| **Governance vote** | DAO vote result | 3 |
| **NFT mint/transfer** | ERC-721 transfer | 1-3 |
| **Price tick** | Oracle price update | 3 (sourced from our own chain) |
| **Contract deployment** | New contract at address | 1 |
| **Attestation hash** | Binary Journal TruthAnchored | 3 |

### 3.2 Source Chains (Order of Support)

| Priority | Chain | Why |
|----------|-------|-----|
| 1 | **WayChain** (native) | Already supported. Our own events. |
| 2 | **Ethereum** | Largest ecosystem. Most value to bridge. |
| 3 | **PulseChain** | Binary Journal attestations. Existing projects. |
| 4 | **Solana** | Different architecture. Cross-ecosystem proof. |
| 5 | **EVM L2s** | Optimism, Arbitrum, Base — lower fees, higher volume. |

Each source chain requires:
- A light client or RPC endpoint accessible to attesters
- Finality confirmation rules (how many blocks before an event is "safe")
- Source chain identifier in the protocol

---

## 4. Consuming Attestations

### 4.1 On-Chain Consumer

A contract on any chain can verify a WayChain attestation:

```solidity
// Pseudocode — exists on any chain with a WayChain bridge
interface IWayChainOracle {
    function getAttestation(
        bytes32 sourceChain,
        uint256 sourceBlock,
        bytes32 sourceTxHash
    ) external view returns (CrossChainAttestation memory);
    
    function verifyWithConfidence(
        bytes32 sourceChain,
        uint256 sourceBlock,
        bytes32 sourceTxHash,
        uint256 minAttesters
    ) external view returns (bool valid);
}

// Usage: a bridge contract
function processDeposit(bytes memory proof) external {
    (bytes32 sourceChain, uint256 blockNum, bytes32 txHash, uint256 minAttesters) = 
        abi.decode(proof, (bytes32, uint256, bytes32, uint256));
    
    require(
        oracle.verifyWithConfidence(sourceChain, blockNum, txHash, minAttesters),
        "Insufficient attestations"
    );
    
    // Mint tokens, release locked funds, etc.
}
```

### 4.2 Off-Chain Consumer

An app can query WayChain's RPC directly:

```typescript
const attestation = await oracleContract.getAttestation(
    ethers.utils.keccak256("ethereum"),
    19204731,
    "0xabcd..."
);

if (attestation.attestationCount >= 3) {
    // Trust the event. Act on it.
}
```

Since WayChain has 1s finality, attestations are available ~5 seconds after
the source chain event (1-2 blocks for attesters to notice + 3 blocks for
the attestation transaction to land).

---

## 5. Connection to Other Specs

### 5.1 Binary Journal

Binary Journal attestations on WayChain can be witnessed for other chains:

```
BJ user attests a hash on WayChain
  → WayChain oracle witnesses (already native)
  → If BJ wants cross-chain visibility:
    → WayChain attesters witness the TruthAnchored event
    → Re-anchor on Ethereum / PulseChain as a cross-chain attestation
    → The truth exists on both chains
```

This gives Binary Journal the option to expand without changing its core
protocol. WayChain handles the cross-chain mechanics.

### 5.2 Governance 2.0

Cross-chain attestations enable governance to verify events on other chains:
- "Did the DAO on Ethereum pass Proposal 42?"
- "Did the PulseChain bridge process a withdrawal?"
- "Is the validator set on Solana what our counterparty claims?"

Governance votes can incorporate attested external data without trusting
a third party.

### 5.3 Oracle Spec (Existing)

The cross-chain attestation layer is an extension of the existing oracle spec
(`NEW_CHAIN_ORACLE_SPEC.md`). Same attester set, same graduated trust model,
same slashing. The extension is: attesters can now witness events on external
chains, not just WayChain's own state.

---

## 6. Summary

| Feature | WayChain Native Oracle | Cross-Chain Extension |
|---------|----------------------|----------------------|
| Data source | WayChain state | External chain events |
| Attester requirement | Dox_Dev badge | Same (no additional) |
| Fee model | Fiat-pegged ~$0.001 | Same |
| Finality | 1 block (1s) | 1 block on WayChain after attestation |
| Slashing | Bond + badge | Same |
| Consumer | WayChain contracts | Any chain with a WayChain bridge |
| Challenge window | Same (100 blocks) | Same |

**Key value proposition:** WayChain's cross-chain attestations carry the
same identity-backed trust as everything else on WayChain. A verified human
signed it. If they lied, they lost their badge permanently. You can build
bridges, markets, and protocols on that.