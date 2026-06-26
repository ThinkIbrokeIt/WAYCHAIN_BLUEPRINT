# Bitcoin Integration — Native, No Wrapping v0.1

**Bitcoin stays on Bitcoin. WayChain verifies it.**

No WBTC. No synthetic token. No bridge with a multisig that can steal
your coins. The smart contract interacts with Bitcoin directly.

---

## 0. Why This Works Now

Every attempt to use Bitcoin in DeFi has required wrapping:

| Solution | How It Works | Problem |
|----------|-------------|---------|
| WBTC | Custodian holds BTC, mints ERC-20 | Centralized. Custodian can freeze. |
| tBTC | Threshold ECDSA signing group | Complex. 51% of signers can steal. |
| RenBTC | Darknodes | Shut down. Same trust problem. |
| Lightning | Off-chain payment channels | Not composable with smart contracts. |

**The common failure:** all of them move the Bitcoin to a custodian and
issue a token. WayChain doesn't need to move the Bitcoin. It just needs
to **verify** it.

WayChain has three things no other L1 has:
1. **Dox_Dev identity** — Oracles are verified humans, not anonymous
2. **Real slashing** — Lie once, lose your badge permanently
3. **Programmable verification** — EVM contracts can verify Bitcoin state

---

## 1. How It Works

### 1.1 Commitment Phase (Bitcoin → WayChain)

```
┌─────────────────────────────────────────────────────────────────┐
│                       BITCOIN CHAIN                              │
│                                                                  │
│  User creates a Bitcoin transaction:                             │
│  - Sends a small amount of BTC to a "commitment address"         │
│    (a Taproot output whose script commits to a WayChain addr)    │
│  - The transaction includes an OP_RETURN with:                   │
│    "WC:{waychain_address}:{contract_address}"                    │
│                                                                  │
│  ┌────────────────────────────┐                                  │
│  │ UTXO now committed to      │                                  │
│  │ WayChain contract          │                                  │
│  └────────────────────────────┘                                  │
└────────────────────────────────────────────────────────────────┘
           │
           │ (Bitcoin block propagates)
           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    WAYCHAIN ORACLE LAYER                          │
│                                                                  │
│  Dox_Dev-verified attesters run Bitcoin light clients:           │
│  1. Detect the transaction                                       │
│  2. Verify it's in a Bitcoin block (check block header)          │
│  3. Verify the OP_RETURN matches valid WayChain addresses        │
│  4. Attest on WayChain: "Commitment verified"                    │
│                                                                  │
│  Multiple attesters (3-5) attest independently.                  │
│  If any attester lies → badge revoked + bond slashed.            │
└────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      WAYCHAIN CONTRACT                            │
│                                                                  │
│  BitcoinRegistry.verifyCommitment(                               │
│    utxo: "txid:vout",                                            │
│    amount: 5.0 BTC,                                              │
│    user: waychain_address,                                       │
│    attestations: [sig1, sig2, sig3]                              │
│  )                                                               │
│                                                                  │
│  → User now has 5 BTC "balance" on WayChain                      │
│  → The Bitcoin never moved. It's still on Bitcoin.               │
│  → The contract knows the user controls it.                      │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 Usage Phase (DeFi on WayChain)

Once a user has a committed balance, they can use it in any WayChain
smart contract:

| Use Case | How |
|----------|-----|
| **LP Pair** | Commit BTC as liquidity in a BTC/WAY pair on WayChain DEX |
| **Collateral** | Borrow WAY against committed BTC |
| **Swaps** | Trade committed BTC for any WayChain asset |
| **Yield** | Lend committed BTC for interest |

**The smart contract doesn't move the Bitcoin.** It just tracks
commitments and updates balances. The Bitcoin sits on Bitcoin.

### 1.3 Withdrawal Phase (WayChain → Bitcoin)

```
User on WayChain wants their BTC back:

1. User signs a withdrawal message on WayChain:
   "Send my committed BTC (UTXO: txid:vout) to bc1q..."
   
2. WayChain contract verifies the signature and reduces the balance

3. WayChain oracles witness the withdrawal event

4. Oracles coordinate to create a Bitcoin transaction:
   - Input: the committed UTXO (still on Bitcoin, still controlled)
   - Output: user's new Bitcoin address
   - Signed by the oracle quorum (N-of-M threshold signature)

5. Transaction broadcast to Bitcoin → user receives BTC
```

---

## 2. The Key Enabler: WayChain's Oracle Infrastructure

This doesn't work without WayChain's oracle model. Here's why:

| Requirement | Old Oracles | WayChain Oracles |
|-------------|-------------|-----------------|
| Verify Bitcoin transactions | Economic only | Dox_Dev identity + economic |
| Collusion resistance | Low (anonymous nodes) | High (verified humans with reputation) |
| Slashing for fraud | Bond loss (can re-enter) | Bond loss + badge revocation (cannot re-verify) |
| Threshold signing | Complex ECDSA groups | Built on attester set (already exists) |
| Audit trail | None | On-chain attestation history |

**A Bitcoin user trusting WayChain to hold their commitment is trusting
verified humans with real identity and real consequences — not anonymous
validators who can rug.**

---

## 3. Technical Components Needed

### 3.1 Bitcoin SPV Verifier (Solidity Contract)

A WayChain contract that can:
- Verify Bitcoin block headers (chain of 80-byte headers)
- Validate Merkle proofs (tx is in a block)
- Track Bitcoin chain tips (know the latest block)

```solidity
// Conceptual
interface IBitcoinSPV {
    function submitBlockHeader(bytes calldata header) external;
    function verifyTxInBlock(
        bytes32 txid,
        bytes calldata merkleProof,
        uint256 blockHeight
    ) external view returns (bool);
    function getBestBlock() external view returns (uint256 height, bytes32 hash);
}
```

Bitcoin SPV is well-understood and has been implemented in Solidity
before (BTC Relay on Ethereum, Summa's SPV verifier). The code exists,
it just needs to be deployed on WayChain.

### 5,000 gas per header verification. ~50,000 gas per tx proof.

### 3.2 Bitcoin Registry Contract

Tracks committed UTXOs and user balances:

```solidity
// Conceptual
contract BitcoinRegistry {
    // User → committed balance
    mapping(address => uint256) public balances;
    
    // UTXO → user (prevents double-commit)
    mapping(bytes32 => address) public commitments;
    
    function commit(
        bytes32 utxo,           // keccak256(txid ++ vout)
        uint256 amount,         // Satoshis
        bytes calldata txProof, // Merkle proof + block header
        bytes[] calldata oracleSigs  // Attestations from N oracles
    ) external;
    
    function withdraw(
        bytes32 utxo,
        string calldata bitcoinAddress,
        bytes calldata signature  // Signed by user's WayChain key
    ) external;
}
```

### 3.3 Oracle Bridge (Already Exists)

The oracle spec already supports witnessing external chain events.
Bitcoin is just another source chain. The attesters run Bitcoin
light clients (or full nodes) alongside their WayChain node.

### 3.4 Bitcoin DEX Pair

A BTC/WAY pair on WayChain's native DEX. LP providers commit BTC
(they never move it) and earn fees in WAY. Traders swap committed
BTC for WAY and back.

---

## 4. Security Model

### 4.1 Trust Assumptions

| Layer | Trusts | Why It's Safe |
|-------|--------|---------------|
| Bitcoin chain | Bitcoin PoW | Bitcoin's security (unmodified) |
| WayChain oracle | 3-5 verified attesters | Dox_Dev identity + slashing makes collusion irrational |
| WayChain consensus | 200 verified validators | One badge = one validator. No whale capture. |

### 4.2 Attack Scenarios

| Attack | Defense |
|--------|---------|
| Oracle lies about a Bitcoin tx | Slashing (badge revocation + bond loss). Other oracles can challenge within 100 blocks. |
| Oracle quorum colludes to steal BTC | Requires >50% of attesters to collude. Each loses their badge (irreplaceable). Reputation cost >> BTC reward. |
| User double-commits same UTXO | Registry contract prevents it. First commitment wins. |
| Bitcoin reorg | Oracles wait for 6 Bitcoin confirmations before attesting. |

### 4.3 Attester Economics

| Parameter | Value |
|-----------|-------|
| Minimum attesters per commitment | 3 (medium value), 5 (high value) |
| Attester bond | 100 WAY + Dox_Dev Level 2+ |
| Slashing for false attestation | Bond loss + badge revocation |
| Challenge window | 100 WayChain blocks |
| Attester reward per commitment | $0.01 (10× normal oracle fee — Bitcoin is higher value) |

---

## 5. Roadmap

### Phase 1: SPV + Registry (Month 1-2)

- [ ] Deploy Bitcoin SPV verifier contract (existing Solidity code, ported)
- [ ] Deploy BitcoinRegistry contract
- [ ] Run Bitcoin light client on WayChain oracle nodes
- [ ] Test: commit a Bitcoin testnet UTXO, verify on WayChain

### Phase 2: DEX Pair (Month 3)

- [ ] Deploy BTC/WAY pair on WayChain DEX
- [ ] LP providers can commit BTC as liquidity
- [ ] Traders can swap committed BTC ↔ WAY
- [ ] All liquidity is Bitcoin-native (never wrapped)

### Phase 3: Withdrawal (Month 4)

- [ ] Oracle quorum threshold signing (N-of-M)
- [ ] Withdrawal to user's Bitcoin address
- [ ] Full round-trip: Bitcoin → WayChain → Bitcoin

### Phase 4: Production (Month 5-6)

- [ ] Mainnet launch on WayChain
- [ ] Targeted marketing to Bitcoin community
- [ ] "Unlock your Bitcoin. No wrapping. No custodian."

---

## 6. Why This Brings Attention

| Audience | What They Get | Why They Care |
|----------|--------------|---------------|
| **Bitcoin holders** | Use BTC in DeFi without trusting a custodian | Finally. Real Bitcoin programmability. |
| **Bitcoin maxis** | Smart contracts without wrapping | No WBTC. No tBTC. Real native BTC. |
| **DeFi users** | Bitcoin liquidity on WayChain | The deepest liquidity pool: 1.2T BTC. |
| **Developers** | Build contracts that handle Bitcoin | First chain where Bitcoin is a first-class asset. |

**The marketing writes itself:**
- "WayChain: Bitcoin works here. No wrapping. No trust."
- "Your BTC on WayChain. Still on Bitcoin. Still yours."
- "The first smart contract chain that respects Bitcoin."

---

## 7. Summary

| Component | Status | What's Needed |
|-----------|--------|--------------|
| Oracle attester set | ✅ Exists | Already spec'd. Same attesters, light client for Bitcoin. |
| Dox_Dev identity | ✅ Exists | Attesters are verified humans. Trust is real. |
| SPV verification | 🔲 Port | Existing Solidity code. Compiles on WayChain EVM. |
| Bitcoin registry | 🔲 Build | New contract. ~200 lines. |
| DEX pair | 🔲 Build | Integrate with existing DEX. |
| Withdrawal flow | 🔲 Build | Oracle threshold signing. |

**Total new code:** ~500 lines of Solidity + Bitcoin light client config for oracle nodes.
**Timeline:** 2-3 months from go-ahead.
**Impact:** WayChain becomes the only chain where Bitcoin is a native, unwrapped, composable asset.
**Tagline:** "Your Bitcoin. Still on Bitcoin. Now on WayChain."