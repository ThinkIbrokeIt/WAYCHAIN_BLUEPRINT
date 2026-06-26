# EVM & Execution Layer — Protocol Spec v0.1

**Base:** Fully EVM-compatible at bytecode level (all Solidity compiles)
**Extensions:** 8 new opcodes, 7 new precompiles, parallel lanes, contract classification
**Gas model:** Fixed base fee + optional priority tip, separate lane limits

---

## 1. EVM Compatibility

### 1.1 What We Inherit

All standard EVM opcodes work identically to Ethereum:
- Arithmetic, bitwise, comparison
- Memory, storage, calldata
- Block context (NUMBER, TIMESTAMP, DIFFICULTY, etc.)
- Environmental (ADDRESS, BALANCE, CALLER, etc.)
- Logging (LOG0-LOG4)
- Calls (CALL, STATICCALL, DELEGATECALL, CALLCODE)
- CREATE, CREATE2
- SELFDESTRUCT (deprecated behavior preserved for compatibility)
- Precompiles 0x01-0x09 (ECRECOVER, SHA256, RIPEMD160, IDENTITY,
  MODEXP, ECADD, ECMUL, ECPAIRING, BLAKE2F)

### 1.2 What Changes

| Change | Reason |
|--------|--------|
| +8 new opcodes | Oracle, randomness, contract classification |
| +7 new precompiles | TLS verification, BLS, account recovery, state rent |
| Gas schedule modified | Storage writes cost more (state rent), reads cost less |
| Block context extended | Includes oracle state root, attester set hash |
| CREATE/CREATE2 gated | Deployer must have sufficient Dox_Dev level for the contract class |
| SELFDESTRUCT disabled | Cannot destroy contracts (preserves Binary Journal anchors) |
| DIFFICULTY repurposed | Returns the VRF random seed for current block (not PoW difficulty) |
| COINBASE repurposed | Returns the proposer's validator index, not address |

### 1.3 Solidity Compatibility

All existing Solidity compiles without modification. New opcodes are
accessed via inline assembly:

```solidity
// New opcode access patterns
assembly {
    // Read oracle value
    let price := oracle_read(sub(block.number, 1), PRICE_FEED_ID)
    
    // Get random seed
    let seed := random(block.number)
    
    // Check contract class
    let class := contract_class(address())
}
```

---

## 2. New Opcodes

### 2.1 Oracle Opcodes

**`ORACLE_READ(blockNumber, feedId) → bytes`**
- Opcode: `0xF0`
- Gas: 200 (historical), 1,000 (current block)
- Returns the finalized value for a feed at a given block
- Reverts if the feed didn't exist at that block
- Reverts if the value wasn't finalized yet

```solidity
// Solidity equivalent
uint256 price = uint256(
    ORACLE_READ(block.number - 1, PRICE_FEED_ID)
);
```

**`ORACLE_REQUEST(feedId, params) → requestId`**
- Opcode: `0xF1`
- Gas: 5,000
- Creates an oracle request and returns its ID
- Contract listens for the `DataFinalized` event to get the result

```solidity
bytes32 reqId = ORACLE_REQUEST(
    WEATHER_FEED_ID,
    abi.encode(ORACLE_TIER_STANDARD, 5000) // tier, reward amount
);
```

**`ORACLE_VERIFY(proof, value, feedId, blockNumber) → bool`**
- Opcode: `0xF2`
- Gas: 10,000
- Verifies a value was finalized for a feed at a specific block
- Used for cross-chain verification and light client proofs

```solidity
require(
    ORACLE_VERIFY(proof, price, PRICE_FEED_ID, block.number)
);
```

### 2.2 Randomness Opcode

**`RANDOM(blockNumber) → bytes32`**
- Opcode: `0xF3`
- Gas: 100
- Returns the VRF random seed for a given block
- Deterministic — same block = same seed every call
- Unpredictable before the block is proposed
- Replaces Chainlink VRF entirely

```solidity
uint256 winner = uint256(RANDOM(block.number)) % totalEntries;
```

### 2.3 Account Opcodes

**`ACCOUNT_STAGE(address) → uint8`**
- Opcode: `0xF4`
- Gas: 200
- Returns the security stage: 0 (onboarding), 1 (standard), 2 (self-custody)
- Useful for DApps that want to limit features based on user maturity

```solidity
uint8 stage = ACCOUNT_STAGE(msg.sender);
require(stage >= 1, "Upgrade your account to use this feature");
```

**`ACCOUNT_LIMIT(address) → (uint256 daily, uint256 used, uint256 maxValue)`**
- Opcode: `0xF5`
- Gas: 300
- Returns the current spending limits and usage for an account
- Used by DEXs and payment contracts to enforce Stage 0 limits

```solidity
(uint256 daily, uint256 used, uint256 maxValue) = ACCOUNT_LIMIT(msg.sender);
require(used + amount <= daily, "Daily limit exceeded");
```

### 2.4 Contract Classification Opcode

**`CONTRACT_CLASS(address) → uint8`**
- Opcode: `0xF6`
- Gas: 200
- Returns the risk class: 0 (none), 1 (A - data), 2 (B - low risk),
  3 (C - medium), 4 (D - high)
- Used by the protocol to enforce deploy permissions
- Used by wallets to warn users about contract risk level

```solidity
uint8 class = CONTRACT_CLASS(address(this));
// Class A data contracts are treated differently by wallets
```

### 2.5 Dead Man's Switch Opcode

**`DEADMAN(address, blockNumber) → bool`**
- Opcode: `0xF7`
- Gas: 200
- Returns true if the account's dead man's switch has triggered
- Used for inheritance, recovery, and time-locked actions

```solidity
if (DEADMAN(msg.sender, block.number)) {
    // Transfer to heir
    heir.transfer(address(this).balance);
}
```

### 2.6 Opcode Summary

| Opcode | Value | Stack In | Stack Out | Gas | Replaces |
|--------|-------|----------|-----------|-----|----------|
| ORACLE_READ | 0xF0 | block, feedId | value | 200/1K | Chainlink feed |
| ORACLE_REQUEST | 0xF1 | feedId, paramsPtr | requestId | 5,000 | Chainlink request |
| ORACLE_VERIFY | 0xF2 | proofPtr, value, feedId, block | bool | 10,000 | Bridge proof |
| RANDOM | 0xF3 | block | seed | 100 | Chainlink VRF |
| ACCOUNT_STAGE | 0xF4 | addr | stage | 200 | Custom check |
| ACCOUNT_LIMIT | 0xF5 | addr | daily, used, max | 300 | Custom check |
| CONTRACT_CLASS | 0xF6 | addr | class | 200 | Custom check |
| DEADMAN | 0xF7 | addr, block | bool | 200 | Custom check |

---

## 3. New Precompile Contracts

### 3.1 `0x0C` — OracleAggregator

Creates derived feeds from existing feeds:

```solidity
// PLS/USD × USD/EUR = PLS/EUR
address derivedFeed = OracleAggregator.createDerivedFeed(
    PLS_USD_FEED,   // base feed
    USD_EUR_FEED,   // quote feed
    OPERATION_MULTIPLY,
    18,             // decimals
    "PLS/EUR"
);
```

Gas: 10,000 + 1,000 per source feed
Supports: MULTIPLY, DIVIDE, ADD, SUBTRACT, MIN, MAX, AVG

### 3.2 `0x0D` — OracleScheduler

Schedules recurring oracle requests — removes keeper bots entirely:

```solidity
// Update this feed every 10 blocks
OracleScheduler.schedule(
    PRICE_FEED_ID,
    10,                 // every N blocks
    ORACLE_TIER_STANDARD,
    100                 // reward per update
);
```

Gas: 5,000 to schedule, 2,000 per execution
Supported intervals: 1 to 100,000 blocks

### 3.3 `0x0E` — OracleVerifier

Verifies cross-chain proofs (SPV, zkBridge, IBC):

```solidity
// Verify a transaction on Ethereum
(bool valid, bytes memory eventData) = OracleVerifier.verifyProof(
    CHAIN_ID_ETHEREUM,
    proof,
    requiredConfirmations
);
require(valid, "Cross-chain proof invalid");
```

Gas: 20,000 (SPV), 50,000 (zkProof), 10,000 (IBC)

### 3.4 `0x0F` — TLSVerifier

Verifies TLSNotary/DECO proofs for data source authenticity:

```solidity
(bool valid, bytes memory response) = TLSVerifier.verify(
    tlSnProof,
    "api.coingecko.com",
    "/api/v3/simple/price?ids=bitcoin"
);
require(valid, "TLS proof invalid");
```

Gas: 25,000
Verifies: server certificate, request hash, response hash, timestamp

### 3.5 `0x10` — BLSVerify

Verifies BLS aggregate signatures for consensus proofs:

```solidity
// Verify a block was finalized by 2/3+ of validators
bool valid = BLSVerify.verifyAggregate(
    validatorsPublicKeys,
    blockHeaderHash,
    aggregateSignature
);
```

Gas: 30,000 + 100 per public key

### 3.6 `0x11` — AccountRecovery

Guardian-based recovery logic for the onboarding ledger:

```solidity
// Initiate recovery
bytes32 reqId = AccountRecovery.requestRecovery(targetAccount);

// Guardian approves
AccountRecovery.approveRecovery(targetAccount, reqId);

// After enough approvals, recover
(bytes memory encryptedKey, address newDevice) = AccountRecovery.executeRecovery(reqId);
```

Gas: 5,000 per operation
Used by: the Onboarding Ledger system contract

### 3.7 `0x12` — StateRent

State rent payment and status queries:

```solidity
// Check rent status for an account
(uint256 owed, uint256 paid, bool frozen) = StateRent.status(address(this));

// Pay rent
StateRent.payRent(address(this), amount);
```

Gas: 500 to check, 2,000 to pay

---

## 4. Gas Model

### 4.1 Fixed Base Fee

Unlike Ethereum's auction-based EIP-1559:

```
baseFee = governance-set value (e.g., 1 native token per tx)
priorityTip = optional (0-100% of base fee, goes to proposer)
```

This ensures:
- Predictable costs in bear AND bull markets
- Non-financial use stays economical (Binary Journal entries cost pennies)
- No gas wars (no "I'll pay 500 gwei to get in next block")
- Validators still earn via volume + tips, not scarcity

### 4.2 Lane-Specific Gas Limits

Each execution lane has its own gas limit:

| Lane | Gas Limit per Block | Priority | Used For |
|------|--------------------|----------|----------|
| Consensus | 30,000,000 | Normal | User transactions |
| Oracle | 5,000,000 | Can't congest consensus | Attestations |

Oracle lane has its own gas pool that resets each block. Oracle
operations cannot starve user transactions and vice versa.

### 4.3 Gas Schedule Changes

Compared to Ethereum's gas schedule:

| Operation | Ethereum | This Chain | Reason |
|-----------|----------|------------|--------|
| SLOAD (cold) | 2,100 | 200 | State reads should be cheap |
| SLOAD (warm) | 100 | 200 | Same |
| SSTORE (new) | 20,000 | 25,000 | Higher storage cost (state rent) |
| SSTORE (update) | 5,000 | 8,000 | Higher storage cost |
| CALL | 2,600 | 3,000 | Slightly higher validation |
| CREATE | 32,000 | 35,000 | + classification check |
| CREATE2 | 32,000 | 35,000 | + classification check |
| LOG0-4 | 375/event | 200/event | Cheaper logging (encourage transparency) |

### 4.4 Gas Abstraction Layer

Users don't pay gas directly — it's deducted from the asset being moved:

```
User swaps 100 USDC for PLS:
  - Contract receives 100 USDC
  - Protocol deducts gas cost from the USDC (at oracle exchange rate)
  - User receives ~99.5 USDC worth of PLS
  - User never sees a "gas fee" line item
  
User sends PLS to friend:
  - Protocol deducts 0.01 PLS for gas
  - User receives "transaction complete" — no gas mentioned
```

dApps can also sponsor gas for their users (protocol-enforced, no relayers).

---

## 5. Parallel Execution Lanes

### 5.1 Two-Lane Architecture

```
Block
├── Consensus Lane (30M gas)
│   ├── tx1: Alice → Bob (transfer)
│   ├── tx2: UniswapV3 swap
│   ├── tx3: Binary Journal anchor
│   └── ...
│
├── Oracle Lane (5M gas)
│   ├── att1: PLS/USD price feed
│   ├── att2: Weather NYC attestation
│   └── ...
│
└── State Roots
    ├── consensusStateRoot: hash after consensus execution
    ├── oracleStateRoot: hash after oracle execution
    └── finalStateRoot: hash(consensusRoot, oracleRoot)
```

### 5.2 Execution Order

1. **Consensus lane executes first** (user transactions)
2. **Oracle lane executes second** (attestations from attesters)
3. Both lanes use the same pre-state
4. Oracle lane can read state written by consensus lane
5. Consensus lane CANNOT read oracle lane state within the same block
   (must wait for next block — prevents oracle manipulation)
6. Final state root = hash(consensusStateRoot, oracleStateRoot)

### 5.3 Security

This ordering prevents flash loan manipulation of oracles:
- Oracle data from block N cannot be used until block N+1
- Flash loan attacks require manipulating price within one block
- 1-block delay breaks the attack

---

## 6. Contract Classification Enforcement

### 6.1 Deployment Gate

At deploy time (CREATE/CREATE2), the EVM checks:

```python
def check_deploy_permission(deployer, contract_code, class):
    stage = account_stage(deployer)
    doxLevel = dox_dev_level(deployer)
    
    # Determine contract class from bytecode analysis
    contract_class = classify_contract(contract_code)
    
    if contract_class == CLASS_A:  # Data
        require(stage >= 1, "Stage 0 cannot deploy data contracts")
        # No Dox_Dev required
        # No value at risk
        
    elif contract_class == CLASS_B:  # Low financial risk
        require(stage >= 1, "Must be Stage 1+")
        require(doxLevel >= 1, "Dox_Dev Level 1 required")
        require(is_audited_template(contract_code), "Must use audited template")
        # Trustless Lock enforced at template level
        
    elif contract_class == CLASS_C:  # Medium financial risk
        require(stage >= 2 or (stage >= 1 and doxLevel >= 2), 
                "Dox_Dev Level 2 or Stage 2 required")
        # Custom contract allowed
        
    elif contract_class == CLASS_D:  # High financial risk
        require(stage >= 2 and doxLevel >= 3, 
                "Dox_Dev Level 3 + Stage 2 required")
        # Multi-sig governance may also be required
```

### 6.2 Bytecode Classification

The protocol classifies contracts by scanning the bytecode for known
patterns:

| Pattern | Class | Detection Method |
|---------|-------|-----------------|
| No payable functions, no value transfers | A | Static analysis |
| Simple ERC-20/BEP-20 (no tax, no reflection) | B | Template match |
| Simple ERC-721 (no royalties, no staking) | B | Template match |
| Has Trustless Lock integration | B | Import detection |
| DEX code (UniswapV2 math) | C | Bytecode signature |
| Lending pool logic | C | Bytecode signature |
| CREATE/CREATE2 in bytecode | D | Static analysis |
| Bridge logic | D | Bytecode signature |
| SELFDESTRUCT (disabled) | D | Pattern match |

If the bytecode can't be classified, it's treated as Class C by default
(requires Dox_Dev Level 2). Governance can reclassify contracts.

### 6.3 Template Registry

Class B contracts must match an audited template:

```
Template {
    id: bytes32,
    name: string,
    bytecodeHash: bytes32,      // hash of the audited bytecode
    riskClass: CLASS_B,
    trustlessLockRequired: bool,
    maxSupply: uint256,         // optional cap
    maxTvl: uint256,            // optional TVL cap
    params: Parameter[],         // user-configurable parameters
    version: uint8,
}
```

Users deploying a Class B contract choose a template and fill in
parameters. The template's bytecode is deployed with the user's
parameters injected. The Trustless Lock is enforced at the template
level — liquidity cannot be withdrawn before the lock period.

### 6.4 Class Override

Governance can override a contract's classification:
- Reclassify a suspicious contract to a higher class
- Reclassify a known-safe contract to a lower class
- Emergency reclassification takes effect in 1 block
- Standard reclassification takes 7 days

---

## 7. State Rent

### 7.1 Mechanism

State rent is separate from execution gas. It's a per-block fee for
storage, not per-transaction:

```
writeCost = 25,000 gas (one-time) + 10 native tokens per KB stored
rentPerKBPerBlock = 0.001 native tokens
```

Every block, all accounts pay rent proportional to their storage usage:
- EOA (externally owned account): 1 token/month (negligible)
- Contract with 10KB state: 0.3 tokens/day (~9 tokens/month)
- Contract with 1MB state: 30 tokens/day (~900 tokens/month)

Rent is deducted from the account's balance automatically. If the
balance hits zero:

```
1. Account enters "frozen" state
   - Reads still work (data is accessible)
   - Writes are blocked (can't modify state)
   - Calls to the contract revert

2. After 30 days frozen:
   - Account state is pruned from the active state trie
   - A hash commitment to the state remains (data can be proven)

3. Recovery:
   - Pay back rent + reinstatement fee (10 native tokens)
   - State is restored from the last pruned checkpoint
   - Contract resumes normal operation
```

### 7.2 Exemptions

Certain accounts are exempt from state rent:
- System contracts (Onboarding Ledger, Dox_Dev, Template Registry)
- Validator staking contract
- Accounts with 0 storage
- Accounts with balance < 1,000 tokens (waived, prevents dust accounts)

### 7.3 Rent Revenue

```
rentRevenue → 60% burned (supply deflation)
           → 30% to validators (extra incentive)
           → 10% to protocol treasury
```

---

## 8. Block Context

### 8.1 Extended Block Header

```
BlockHeader {
    // Standard
    parentHash: bytes32,
    number: uint64,
    timestamp: uint64,
    
    // Our chain
    proposerIndex: uint16,         // Index in active validator set
    randomSeed: bytes32,           // VRF seed for this block
    attesterSetHash: bytes32,      // Current attester set
    attesterSetSize: uint16,       // Number of active attesters
    
    // State (dual-lane)
    consensusStateRoot: bytes32,
    oracleStateRoot: bytes32,
    finalStateRoot: bytes32,
    
    // Transaction roots
    txRoot: bytes32,
    oracleTxRoot: bytes32,
}
```

### 8.2 Repurposed Opcodes

| Standard Opcode | Our Behavior | Rationale |
|----------------|--------------|-----------|
| DIFFICULTY (0x44) | Returns VRF random seed for current block | PoW difficulty is meaningless in PoS; randomness is more useful |
| COINBASE (0x41) | Returns proposer's validator index | Proposer identity is more useful than address |
| GASLIMIT (0x45) | Returns consensus lane gas limit | Contracts need to know which gas limit applies |
| CHAINID (0x46) | Returns our chain ID | Standard but needed for replay protection |

---

## 9. Account Abstraction

### 9.1 Account Model

Every account is a unified account (not EOA vs Contract split):

```
Account {
    nonce: uint64,
    balance: uint256,
    storageRoot: bytes32,
    codeHash: bytes32,
    
    // Our extensions
    stage: uint8,              // 0=onboarding, 1=standard, 2=self-custody
    doxDevBadge: uint256,      // Token ID or 0
    guardianRoot: bytes32,     // Merkle root of guardian addresses
    sessionKeyRoot: bytes32,   // Merkle root of active session keys
    recoveryContact: address,  // Who to notify on recovery
    
    // State rent
    rentPaid: uint256,
    lastRentBlock: uint64,
    frozen: bool,
}
```

### 9.2 Session Keys

Session keys are stored in the account's state as a merkle tree:

```
sessionKey = {
    appId: bytes32,
    publicKey: bytes,          // 65 bytes (uncompressed secp256k1)
    permissions: bytes32,      // Encoded permissions bitmap
    maxSpend: uint256,
    expiryBlock: uint32,
    revoked: bool,
}
```

Contracts verify session keys via a system-level precompile. Gas:
2,000 per verification.

### 9.3 Key Rotation

```
rotateKey(newKey, signature):
    - Validates signature from current key
    - Updates account's key to newKey
    - In Stage 0: also updates encrypted backup in Onboarding Ledger
    - In Stage 1-2: user manages their own key
```

---

## 10. Summary

| Feature | Value |
|---------|-------|
| EVM compatibility | Full (all Solidity compiles unmodified) |
| New opcodes | 8 (0xF0-0xF7) |
| New precompiles | 7 (0x0C-0x12) |
| Base fee model | Fixed (governance-set, not auctioned) |
| Consensus lane gas limit | 30,000,000 per block |
| Oracle lane gas limit | 5,000,000 per block |
| State rent | 0.001 native / KB / block |
| Contract classes | A (data), B (templates), C (medium), D (high) |
| Template registry | Audited bytecode templates for Class B |
| Account model | Unified (EOA + contract, with stage) |
| Session keys | Merkle tree, gas: 2,000 per verify |
| Key rotation | Supported at all stages |
| DIFFICULTY repurposed | Returns VRF random seed |
| COINBASE repurposed | Returns proposer validator index |
| CREATE/CREATE2 gated | Dox_Dev level check per contract class |
| SELFDESTRUCT | Disabled (protects Binary Journal anchors) |