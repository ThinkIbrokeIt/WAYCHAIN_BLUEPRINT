# WayChain Developer Guide

## Building on the Chain That's Actually Decentralized

---

## 1. Quick Start

### 1.1 Run a Local Node

```bash
# Clone and build
git clone <repo>
cd waychain-consensus
go build -o waychain .

# Run all demos (consensus, EVM, P2P, oracle, RPC, genesis)
./waychain demo

# Or initialize a new chain
./waychain init
./waychain start
```

### 1.2 RPC Endpoint

WayChain exposes a standard JSON-RPC interface on port 9545:

```bash
# Get chain ID
curl -X POST http://127.0.0.1:9545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'

# Get block number
curl -X POST http://127.0.0.1:9545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

---

## 2. Connecting to WayChain

### 2.1 Using ethers.js

```javascript
const { ethers } = require('ethers');

const provider = new ethers.JsonRpcProvider('http://127.0.0.1:9545');
const blockNumber = await provider.getBlockNumber();
const balance = await provider.getBalance('0x616c696365');
console.log(`Block: ${blockNumber}, Balance: ${balance}`);
```

### 2.2 Using viem

```typescript
import { createPublicClient, http } from 'viem';

const client = createPublicClient({
  chain: { id: 10008, name: 'WayChain' },
  transport: http('http://127.0.0.1:9545'),
});

const block = await client.getBlockNumber();
```

### 2.3 Using Foundry / Forge

```bash
# Set WayChain as the RPC target
export ETH_RPC_URL=http://127.0.0.1:9545

# Deploy a contract
forge create src/MyContract.sol:MyContract --rpc-url $ETH_RPC_URL --private-key <key>

# Interact with contracts
cast call 0x<contract> "balanceOf(address)(uint256)" 0x<address>
```

---

## 3. Deploying Contracts

### 3.1 Contract Classification

Every contract on WayChain has a risk class. The class determines who can deploy:

| Class | Who Can Deploy | Gas Premium | Examples |
|-------|---------------|-------------|----------|
| **A** (Safe) | Anyone | None | Attestation.sol, ERC-20 tokens |
| **B** (Managed) | Dox_Dev Level 2+ | +10% | DeadMansSwitch, DEX pairs |
| **C** (Governed) | Dox_Dev Level 3+ | +25% | Protocol-level contracts |
| **D** (Restricted) | Governance vote | +50% | System contracts |

```solidity
// WayChain native opcodes
uint8 class = CONTRACTCLASS();         // 0xC0 — current contract's class
uint8 level = DOXDEVLEVEL();           // 0xC1 — caller's Dox_Dev level
uint8 lane  = LANETYPE();              // 0xC2 — execution lane (0=consensus, 1=oracle)
```

### 3.2 Using the Template Registry

The recommended way to deploy is through WayChain's Template Registry:

```solidity
// Register your contract as a template (curator-only)
registry.registerTemplate(
  "MyContract",
  "Description of what it does",
  TemplateRegistry.ContractClass.A,
  abi.encode(keccak256(type(MyContract).creationCode))
);

// Anyone can deploy Class A templates
registry.recordDeployment(templateId, address(deployedContract));
```

### 3.3 Writing Contracts with WayChain Primitives

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract WayAwareContract {
    // Verify the caller has a Dox_Dev badge
    function verifyCaller() internal view {
        uint8 level;
        assembly {
            level := DOXDEVLEVEL()
        }
        require(level >= 2, "Dox_Dev Level 2+ required");
    }

    // Anchor a hash to WayChain's immutable record
    function anchorTruth(bytes32 hash) external {
        assembly {
            mstore(0x00, hash)
            pop(ATTEST())
        }
    }

    // Get random value
    function getRandom() internal view returns (uint256) {
        uint256 rand;
        assembly {
            rand := RANDOM()
        }
        return rand;
    }

    // Verify any address's badge
    function checkBadge(address user, uint8 minLevel) internal view returns (bool) {
        bool result;
        assembly {
            mstore(0x00, user)
            mstore(0x20, minLevel)
            result := VERIFYBADGE()
        }
        return result;
    }
}
```

---

## 4. WayChain-Specific Features

### 4.1 Dox_Dev Badges

```javascript
// Check a user's Dox_Dev level
const level = await provider.send('way_getDoxLevel', ['0x<address>']);
// Returns: "0x0" (unverified), "0x1" (basic), "0x2" (pro), "0x3" (enterprise)

// Solidity: check from any contract
DoxDevBadge badge = DoxDevBadge(doxDevAddress);
uint8 level = badge.getLevel(user);
bool verified = badge.isVerified(user);
bool hasLevel = badge.hasMinLevel(user, 2);
```

### 4.2 Precompiled Contracts

WayChain includes 7 precompiled contracts at addresses `0x0C` through `0x12`:

| Address | Name | Purpose |
|---------|------|---------|
| `0x0C` | OracleAggregator | Aggregate attestations from multiple Dox_Dev oracles |
| `0x0D` | OracleScheduler | Schedule recurring attestations |
| `0x0E` | OracleVerifier | Verify individual oracle attestations |
| `0x0F` | TLSVerifier | Verify TLS proof data from oracle data sources |
| `0x10` | BLSVerify | Verify BLS12-381 aggregate signatures |
| `0x11` | AccountRecovery | Guardian-based account recovery |
| `0x12` | StateRent | Calculate and deduct state rent |

### 4.3 Gas Costs

| Operation | Gas | Notes |
|-----------|-----|-------|
| Simple transfer (EOA→EOA) | 21,000 | Standard |
| Contract creation | 32,000 + code gas | Standard |
| SSTORE (warm) | 5,000 | Standard |
| SLOAD | 2,100 | Standard |
| CONTRACTCLASS (0xC0) | 2 | WayChain native |
| DOXDEVLEVEL (0xC1) | 20 | WayChain native |
| LANETYPE (0xC2) | 2 | WayChain native |
| ATTEST (0xC3) | 20,000 | WayChain native |
| RANDOM (0xC4) | 20 | WayChain native |
| RENTBALANCE (0xC5) | 700 | WayChain native |
| DEADMANSWITCH (0xC6) | 2,000 | WayChain native |
| VERIFYBADGE (0xC7) | 700 | WayChain native |

### 4.4 Fee Calculation

Fees are fiat-pegged at ~$0.001 per simple transaction. The actual WAY amount adjusts based on the oracle-provided WAY/USD price:

```javascript
// Estimated cost in WAY
const gasPrice = await provider.send('eth_gasPrice', []);
const gasLimit = 21000;
const txCost = BigInt(gasPrice) * BigInt(gasLimit);
// txCost ≈ $0.001 in WAY at current oracle price
```

---

## 5. Running a Validator

### 5.1 Requirements

| Requirement | Detail |
|-------------|--------|
| Dox_Dev badge | Level 2+ |
| Minimum stake | 100 WAY |
| Hardware | Dedicated machine (separate from oracle) |
| Bandwidth | 10 Mbps minimum |
| Uptime | 95%+ (grace period first 90 days) |

### 5.2 Node Setup

```bash
# Initialize
./waychain init

# Start as a validator
WAYCHAIN_NODE_ID="my-validator" \
WAYCHAIN_LISTEN=":9100" \
WAYCHAIN_PEERS="<peer1>:9100,<peer2>:9101" \
WAYCHAIN_DEVNET=1 \
./waychain
```

### 5.3 Running a Devnet

```bash
# Start a 4-node devnet
bash devnet.sh 4
```

---

## 6. WayChain RPC Reference

### 6.1 Standard Ethereum Methods

| Method | Returns |
|--------|---------|
| `eth_chainId: 10008) |
| `eth_blockNumber` | Latest block height |
| `eth_getBalance` | Account balance in WAY |
| `eth_getTransactionCount` | Account nonce |
| `eth_gasPrice` | Current gas price |
| `eth_estimateGas` | Gas estimate for a transaction |
| `eth_sendRawTransaction` | Submit a transaction |
| `eth_call` | Execute a call (no state change) |

### 6.2 WayChain-Specific Methods

| Method | Params | Returns |
|--------|--------|---------|
| `way_getDoxLevel` | `address` | Dox_Dev badge level (0-3) |
| `way_getBalance` | `address` | Balance in WAY |
| `way_getBlockCount` | — | Total blocks |
| `way_getValidatorCount` | — | Active validator count |

---

## 7. Example Projects

### 7.1 Deploy a Dead Man's Switch

```solidity
DeadMansSwitch dms = new DeadMansSwitch();
uint256 switchId = dms.createSwitch(
    DeadMansSwitch.TruthType.Dark,
    heirAddress,
    30 days,
    keccak256("my_encryption_key")
);
```

### 7.2 Anchor a Truth Hash

```solidity
Attestation att = Attestation(attestationAddress);
att.attest(keccak256("my_truth_data"));
// TruthAnchored event emitted — immutable, permanent
```

### 7.3 Provide Liquidity

```solidity
// Lock LP tokens with 98/2 revenue share
TrustlessLock lock = TrustlessLock(lockAddress);
uint256 lockId = lock.createTimeLock(
    lpTokenAddress,
    amount,
    180 days
);
// 98% of LP earnings go to you, 2% to WayChain treasury
```

---

## 8. Advanced Topics

### 8.1 Parallel Execution Lanes

WayChain supports three execution lanes:
- **Consensus lane (0)** — Standard transactions (default)
- **Oracle lane (1)** — Oracle attestations with separate gas pool
- **Private lane (2)** — Encrypted mempool transactions

Contracts can read their current lane via the `LANETYPE` opcode (0xC2).

### 8.2 State Rent

Contracts pay state rent per block based on their storage size. The rent is burned (60%) and distributed to validators (40%). Run the `STATECALC` precompile (0x12) to calculate current rent due.

### 8.3 Bitcoin Integration

WayChain contracts can verify Bitcoin transactions natively. See the Bitcoin Integration spec for details on committing UTXOs, using BTC in DeFi, and withdrawing without wrapping.

---

## 9. Security & Best Practices

1. **Always check Dox_Dev levels** before trusting external actors
2. **Use the Template Registry** for deployments — pre-audited bytecode is safer
3. **Respect class gates** — a Class B operation without Dox_Dev Level 2+ will revert
4. **Anchor hashes, not raw data** — the ATTEST opcode emits on-chain events; don't put PII on-chain
5. **Test with forge** — WayChain is EVM-compatible; all existing Foundry tooling works

---

## 10. Getting Help

- **RPC endpoint**: `http://127.0.0.1:9545` (local devnet)
- **Block explorer**: Open `waychain-dashboard.html` in a browser
- **Source**: All WayChain code is in `waychain-consensus/` and `waychain-contracts/`

---

*WayChain — Actually Decentralized. One Human. One Voice.*