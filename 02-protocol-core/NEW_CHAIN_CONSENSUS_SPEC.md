# Consensus Mechanism — Protocol Spec v0.1

**Algorithm:** BFT Proof-of-Stake (CometBFT-derived)
**Block time:** 1 second target
**Finality:** Instant (after 2/3+ commit, no reorgs)
**Active validator set:** Top 200 selected via weighted random lottery per epoch (anti-whale, equal voting power)
**Registered validators:** Any amount (competition without whale dominance)

---

## 1. Design Rationale

### 1.1 Why BFT PoS Over Alternatives

| Algorithm | Block Time | Finality | Pros | Cons |
|-----------|-----------|----------|------|------|
| PoW (Bitcoin) | 10 min | Probabilistic (~1hr) | Battle-tested | Slow, wasteful, no finality |
| Gasper (Ethereum) | 12s | Probabilistic (~15min) | Large validator set | Slow finality, complex |
| **CometBFT** | **1-2s** | **Instant (1 block)** | **Fast, simple, proven** | **1/3 fault tolerance** |
| Avalanche | ~1s | Probabilistic (~3s) | Fast, subnets | Complex, DAG overhead |
| HotStuff | ~1s | Instant | High throughput | Less battle-tested |

CometBFT is the right choice because:
- **1s blocks** — good UX (user doesn't wait)
- **Instant finality** — oracle attesters need to know data won't be reorged
- **Well-proven** — Cosmos, Binance Chain, Kava, Osmosis — billions under management
- **Simple light client** — epoch-based validator set checkpoints for cross-chain
- **Deterministic** — no MEV (no validator reordering if blocks are deterministic)

The 1/3 fault tolerance is acceptable because attesters are a separate
set (oracle spec v0.2). If 34% of validators are Byzantine, they can
halt the chain but cannot corrupt data. Full attack requires
compromising BOTH sets.

### 1.2 What This Chain Needs That Stock CometBFT Doesn't Have

| Need | Modification |
|------|-------------|
| Parallel oracle lane | Separate mempool + execution lane for attestations |
| Native randomness | VRF-based seed per block from validator threshold signatures |
| Attester interface | Validators elect attesters each epoch, serve as backup |
| Fixed low fees | Gas model caps block fee, not auction-based |
| State rent | Storage pricing separate from execution pricing |

These modifications are detailed in Section 7.

---

## 2. Validator Set

There are two tiers of validators:

**Registered validators** — Anyone can register by meeting the minimum
requirements. There is no cap. Thousands can register.

**Active validator set** — From the pool of registered validators, N are
selected each epoch via a weighted random lottery. Selection probability
uses `sqrt(stake)` not linear stake — whales get an advantage, but not
an overwhelming one. Once selected, all active validators have EQUAL
voting power. This prevents a single entity from dominating consensus.

### 2.1 Becoming a Validator (Registered)

```
Requirements to REGISTER as a validator:
  - Minimum self-bond: 10,000 native tokens
    (This is the floor to register. Selection into the ACTIVE SET uses
    a weighted random lottery — even with minimum stake, you have a
    real chance of being selected each epoch)
  - Node specs: 8+ cores, 32GB RAM, 500GB NVMe SSD
  - Network: 100Mbps+ symmetric, <50ms to other validators
  - Registered operator identity (Dox_Dev Level 2 or governance-approved)
  - One-time registration fee: 100 native tokens
```

**Stake composition:**
- Self-bond: validator's own tokens (minimum 10% of their total stake)
- Delegated: tokens from other holders who delegate to this validator
- Total stake = self-bond + delegated (determines active set ranking)

**Delegation mechanics:**
- Delegators earn a share of validator rewards minus commission
- Delegators can unbond at any time (14-day unbonding period)
- No minimum delegation amount
- Slashing risk is shared by all delegators proportional to contribution

### 2.2 Active Set Selection — Weighted Random Lottery

At the start of each epoch (10,000 blocks ~ 2.8 hours), the active set
is selected via a verifiable random lottery:

```
1. All REGISTERED validators who meet minimum requirements are eligible:
   - Self-bond ≥ 10,000 native tokens
   - Uptime ≥ 90% in last epoch
   - No active slashing disputes
   - Dox_Dev identity verified

2. Each eligible validator is assigned a weight:
   weight = sqrt(totalStake)
   (Using sqrt instead of linear: 4x the stake = 2x the chance,
    not 4x. A validator with 500K has ~7x the chance of one with
    10K, not 50x.)

3. Active set of N validators (governance-set, initial 200) is
   selected via weighted random selection:
   - Uses the per-block VRF random seed
   - Selection is deterministic given the seed (anyone can verify)
   - Same validator cannot be selected >1 time per epoch
   - A validator can serve consecutive epochs (natural, not prohibited)

4. ALL selected validators have EQUAL voting power:
   - One validator = one vote in BFT consensus
   - Voting power is NOT proportional to stake
   - This prevents whales from dominating block production

5. Validators not selected stay registered and eligible for next epoch
```

**Probability examples with sqrt weighting (active set = 200):**

| Validator Stake | sqrt(stake) | Relative Weight | Approximate Slots/Epoch |
|----------------|-------------|-----------------|------------------------|
| 10,000 (minimum) | 100 | 1x | ~1-2 |
| 100,000 | 316 | 3.2x | ~5-8 |
| 500,000 | 707 | 7.1x | ~12-18 |
| 1,000,000 | 1,000 | 10x | ~17-25 |
| 5,000,000 | 2,236 | 22x | ~35-50 |
| 10,000,000 | 3,162 | 32x | ~50-70 |

A validator with **1,000x the stake** of the minimum only gets
**~32x the selection probability**. Whales still have an advantage
(they SHOULD — more at risk) but they cannot dominate. A whale
controlling 50% of all staked tokens would still only get ~50-70
slots out of 200 — a majority requires 101+.

**This is the key difference from every existing PoS chain:**
- Ethereum: top 32 ETH stakers only (whale dominant)
- Cosmos: top 180 by stake (whale dominant)
- Solana: top validators by stake (whale dominant)
- **This chain:** All eligible validators get a real chance,
  selected by verifiable lottery, equal voting power once selected

**Validator diversity enforcement:**
- Binary Journal tracks validator geographic distribution
- Governance can exclude validators from the same hosting provider
- No single entity can run >1 validator (enforced by Dox_Dev identity)
- Validator identity is registered and verified

### 2.3 Validator Rewards

```
blockReward = baseInflation × totalStaked / blocksPerYear
validatorReward = blockReward × votingPower / totalVotingPower
delegatorShare = validatorReward × (1 - commissionRate)
```

| Parameter | Value |
|-----------|-------|
| Base inflation | 7% APY (target), 5-10% range governed |
| Commission range | 0-100% (validator-set, delegator-visible) |
| Reward distribution | Every block (in-protocol, not manual claim) |
| Reward source | New issuance + transaction fees |

Distribution flow:
```
Block rewards → Protocol treasury (10%)
              → Active validators (90%) proportional to voting power
                                → Validator commission (e.g., 10%)
                                → Delegators (remaining 90%)
```

### 2.4 Slashing

| Violation | Slash Amount | Jail Duration | Notes |
|-----------|-------------|---------------|-------|
| Equivocation (double-sign) | 5% of total stake | 21 days | Proof via header chain fork detection |
| Downtime (<90% uptime) | 0.5% per missed window | 1 day | 1-hour missed-window granularity |
| Governance-identified misbehavior | 1-10% (governance-set) | 0-90 days | Requires governance vote |
| Collusion with attesters (proven) | 10% of validator stake | Permanent | Evidence required |

**Slashing distribution:**
- 50% burned (reduces supply, protects token holders)
- 25% to the reporter (incentivizes monitoring)
- 25% to remaining validators (rewards honest majority)

**Unbonding period:** 14 days. Delegators who unbond during an active
slashing dispute may still be slashed (stake locked until resolution).

---

## 3. Block Production

### 3.1 Block Time & Structure

```
Block time: 1 second target (configurable, governance-controlled)
Block size: Target 1MB, max 5MB (dynamic based on network conditions)

Block structure:
Block {
    header: {
        height: uint64,
        timestamp: uint64,          // Unix ms
        proposer: address,          // Validator who proposed this block
        lastBlockHash: bytes32,
        stateRoot: bytes32,         // Post-execution state root
        txRoot: bytes32,            // Merkle root of all transactions
        oracleTxRoot: bytes32,      // Merkle root of oracle attestations
        attestationSetHash: bytes32, // Hash of current attester set
        randomSeed: bytes32,        // VRF output for this block
        consensusProof: bytes,      // BLS aggregate sig of 2/3+ validators
    },
    consensusLane: [Transaction],   // Normal transactions
    oracleLane: [Attestation],      // Oracle attestations only
    evidence: [Evidence],           // Proofs of misbehavior (equivocation, etc.)
    signatures: [CommitSignature],  // 2/3+ validator signatures
}
```

### 3.2 Proposer Selection

```
weightedRoundRobin(votingPower, totalVotingPower, height):
    1. seed = hash(lastRandomSeed, height)
    2. cumulative = 0
    3. for validator in sorted(votingPower):
         cumulative += validator.votingPower
         if cumulative > (totalVotingPower × hash/2^256):
             return validator
    4. return highest-power validator (fallback)
```

This is deterministic — every honest validator computes the same
proposer for the same height. No forks, no ambiguity.

### 3.3 Block Production Flow

```
1. Proposer selected (deterministic, from random seed)
2. Proposer builds block:
   a. Collects transactions from mempool (highest fee first)
   b. Collects oracle attestations from oracle mempool
   c. Executes both lanes (consensus first, then oracle)
   d. Computes stateRoot, txRoot, oracleTxRoot
   e. Signs + broadcasts proposed block
3. Other validators receive proposed block:
   a. Verify proposer is correct for this height
   b. Verify all transactions execute correctly
   c. Verify both state roots match local execution
   d. Verify oracle attestations are valid
   e. Sign commit or send reject
4. If 2/3+ of voting power commits:
   a. Block is finalized
   b. Proposer receives block reward
   c. Next proposer starts building next block
5. If <2/3 commits within timeout (3s):
   a. Proposer is skipped
   b. Next proposer in rotation tries
   c. Evidence of proposer's failure recorded
```

### 3.4 Timeouts & Liveness

| Event | Timeout | Action |
|-------|---------|--------|
| Proposer doesn't propose | 3s | Next proposer takes over |
| Block proposal invalid | Immediate | Validators reject, skip proposer |
| <2/3 commit within timeout | 3s | Skip round, next proposer |
| Network partition | After timeout | Validators continue on longest chain |

The chain prioritizes liveness over consistency — a single faulty
proposer cannot halt the chain. The timeout-based skip mechanism
ensures progress as long as >2/3 of validators are honest.

---

## 4. Finality

### 4.1 Instant Finality

Unlike probabilistic finality (Bitcoin: 6+ confirmations), our chain
has instant finality:

```
Block N proposed → 2/3+ validators commit → Block N is FINAL
```

Once a block is committed:
- It will never be reorganized
- Transactions in it are final
- Oracle attestations in it are final
- State transitions are irreversible

**No forks.** At any given height, there is exactly one canonical
block. This is guaranteed by the BFT consensus rules.

### 4.2 Why This Matters

| Feature | Without Instant Finality | With Instant Finality |
|---------|------------------------|----------------------|
| Oracle attestations | Must wait N blocks for safety | Final at block 1 |
| Cross-chain proofs | Must include N confirmations | Single block proof |
| User experience | "Wait for confirmations" | "Transaction confirmed" |
| DEX swaps | Risk of reorg | Safe immediately |
| Settlement finality | Hours to days | 1 second |

### 4.3 Finalization Proof

Any finalized block can be verified by a light client:

```
finalizationProof = {
    blockHeader: BlockHeader,
    commitSignatures: [{
        validatorIndex: uint16,
        signature: bytes64,          // BLS signature
        votingPower: uint64,
    }],
    totalVotingPower: uint64,        // Total at this epoch
    attesterSetRoot: bytes32,        // For cross-chain verification
}
```

A light client:
1. Verifies the validator set from the latest epoch checkpoint
2. Verifies 2/3+ voting power signed the block
3. Verifies the state root matches
4. Accepts as final

This is a single block proof — no waiting for confirmations.

---

## 5. Native Randomness

### 5.1 Why Native Randomness

The chain needs randomness for:
- **Attester selection** — unbiased selection for each feed round
- **Validator rotation** — proposer selection should be unpredictable
- **VRF opcode** — contracts need verifiable randomness (gaming, lotteries, NFT reveals)

Stock CometBFT uses deterministic proposer selection based on the
previous block hash, which is predictable. We add VRF-based randomness.

### 5.2 Randomness Generation

Each block produces a fresh random seed:

```
1. Validator set generates a BLS threshold signature each epoch
2. Each block proposer includes a random seed derived from:
   seed = hash(epochSeed, height, proposerSignature)
3. The seed is committed in the block header
4. Contracts can read ORACLE_RANDOM(blockNumber) to get the seed
```

**Security:**
- No single validator can predict or influence the seed
- The threshold signature requires 2/3+ of validators to cooperate
- Even if the proposer tries to bias, they control only 1/200 of the input
- The seed is verifiable (anyone can recompute it)

### 5.3 Randomness as an Opcode

```
OPCODE: RANDOM(blockNumber) → bytes32
Gas: 100 (same as a basic hash)

// Solidity equivalent
bytes32 seed = RANDOM(block.number);
uint256 winner = uint256(seed) % totalParticipants;
```

This replaces:
- VRF oracle calls (no extra cost, no third-party)
- Blockhash-based RNG (no manipulation, even for old blocks)
- Commit-reveal schemes (no complexity)

---

## 6. Staking & Delegation Economics

### 6.1 Token Flows

```
                     ┌──────────────┐
                     │   Inflation   │  ← 7% APY target
                     └──────┬───────┘
                            │
                            ▼
                   ┌────────────────┐
                   │ Protocol Treasury │  ← 10% of inflation
                   └────────────────┘
                            │
                            ▼
                   ┌────────────────┐
                   │  Validators +    │  ← 90% of inflation + fees
                   │   Delegators     │
                   └────────────────┘
                            │
                            ▼
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
     ┌────────────────┐         ┌────────────────┐
     │  50% Slashed    │         │    Competitors   │
     │   (burned)       │         │   (reporters +    │
     └────────────────┘         │    honest valid.)  │
                                 └────────────────┘
```

### 6.2 Delegation Mechanics

```
Delegate to Validator:
  1. User sends tokens to delegation contract
  2. Tokens are locked + assigned to chosen validator
  3. Delegator receives shares representing their portion
  4. Rewards auto-compound every block
  5. Unbonding: 14-day waiting period before withdrawal

Validator Commission:
  1. Validator sets commission rate (0-100%)
  2. Commission is taken from the validator's total reward before
     distribution to delegators
  3. Rate can be changed with 24-hour delay (prevents sudden changes)
  4. Delegators are notified of commission changes
```

### 6.3 Governance Parameters

| Parameter | Default | Range | Change Mechanism |
|-----------|---------|-------|-----------------|
| Base inflation | 7% | 5-10% | Governance vote, 7-day delay |
| Min self-bond to register | 10,000 | 5,000-50,000 | Governance vote, 14-day delay |
| Active validator set size | 200 | 50-500 | Governance vote, 14-day delay |
| Block time | 1s | 0.5-5s | Governance vote, 7-day delay |
| Max block size | 1MB | 1-10MB | Governance vote, 3-day delay |
| Unbonding period | 14 days | 7-28 days | Governance vote, 14-day delay |
| Slash percentages | See table | 0-25% | Governance vote with supermajority |
| Max voting power per validator | 20% | 10-33% | Governance vote, 14-day delay |

---

## 7. Modifications to Stock CometBFT

### 7.1 Parallel Oracle Lane

Normal transactions and oracle attestations execute in parallel lanes:

```
Block {
    height: N,
    timestamp: T,
    
    // Normal lane
    consensusLane: [
        tx1 (swap), tx2 (transfer), tx3 (contract call), ...
    ],
    
    // Oracle lane (parallel execution)
    oracleLane: [
        attestation1, attestation2, ...
    ],
    
    // Separate state roots
    stateRoot: hash(consensusExecutionState),
    oracleStateRoot: hash(oracleExecutionState),
    
    // Combined final state
    finalStateRoot: hash(stateRoot, oracleStateRoot),
}
```

Benefits:
- Oracle operations don't congest normal transactions
- Normal gas prices aren't affected by oracle demand
- Each lane has its own gas limit
- Both lanes finalize in the same block

### 7.2 Attester Set Interface

The consensus protocol exposes attester set information:

```
BlockHeader {
    ...
    attesterSetHash: bytes32,      // Hash of current attester set
    attesterSetSize: uint16,       // Number of active attesters
    validatorBackupFlag: bool,     // True if validators are serving as backup attesters
    ...
}
```

This allows:
- Oracle contracts to verify an attestation came from a current attester
- Light clients to verify attester set membership
- Emergency backup activation to be detected

### 7.3 Fixed / Capped Gas Model

Unlike Ethereum's auction-based gas (EIP-1559), this chain uses a
fixed-price model for the base fee:

```
baseFee = governance-set (e.g., 1 native token per tx)

Per-block fee cap:
  consensusCap = 100,000 native tokens
  oracleCap = 10,000 native tokens
  
Priority fee: optional tip for faster inclusion (0-100% of base fee)
```

This ensures:
- Predictable costs for users (no gas spikes during bull markets)
- Non-financial use cases remain economical (Binary Journal entries)
- Oracle attestations have dedicated cheap capacity
- Validators still earn fees (tipping for priority)

### 7.4 State Rent

Storage costs are separated from execution costs:

```
writeSlotCost = 200 gas + 10 native tokens per KB stored
readSlotCost = 200 gas (free for historical reads)
rentPerKBPerBlock = 0.001 native tokens

When account balance < accrued rent:
  - Account enters "frozen" state (reads work, writes blocked)
  - After 30 days frozen: account data pruned
  - Data recoverable by paying back rent + reinstatement fee
```

This prevents state bloat (a critical problem on Ethereum where
inactive contracts permanently burden validators).

---

## 8. Light Client Support

### 8.1 Epoch Checkpoints

Every epoch (10,000 blocks ~ 2.8 hours), the validator set is
committed to the state:

```
Checkpoint {
    epoch: uint64,
    validatorSetRoot: bytes32,    // Merkle root of validator set
    attesterSetRoot: bytes32,     // Merkle root of attester set
    blockHeight: uint64,
    consensusProof: bytes,        // 2/3+ validator BLS sig
}
```

Light clients only need to track the latest checkpoint, not every block.

### 8.2 Cross-Chain Proof

A finalized block's state can be proven to another chain:

```
CrossChainProof {
    blockHeader: BlockHeader,
    epochCheckpoint: Checkpoint,
    merkleProof: MerkleProof,      // Proof of a specific state value
    commitSignatures: [CommitSignature],
}
```

This proof is:
- ~1KB (small enough for on-chain verification on another chain)
- Verifiable in ~50K gas
- Final (no waiting for confirmations)

---

## 9. Security Model

### 9.1 Threat Model

| Attack | Feasibility | Impact | Mitigation |
|--------|-------------|--------|------------|
| **33%+ Byzantine validators halt chain** | Requires controlling 101+ of 200 active slots. With sqrt weighting, a whale with 50% of total stake gets ~60-70 slots — needs multiple independent whales to collude | Chain halts | Liveness timeout + proposer skip keeps chain moving |
| **Equivocation (double-sign)** | Single validator misbehavior | Connects two forks | Instant detection via header proof, 5% slash + jail |
| **Long-range attack** | Requires old validator set keys | Rewrites history | Epoch checkpoints in state (immutable after N epochs) |
| **Validator collusion with attesters** | Must control >33% validators + >50% attesters simultaneously | Corrupt data + blocks | Separate sets, limited overlap (20% max), economic disincentive |
| **Delegation centralization** | One entity controls multiple validators | Censorship risk | Dox_Dev identity per validator, stake cap (20% max) |
| **MEV / reordering** | Proposer reorders txs for profit | User exploitation | Deterministic block building (transactions ordered by hash, not at proposer's discretion) |

### 9.2 Economic Security

The cost to attack the chain:

```
Attack cost to control 101/200 active slots in weighted lottery:
= Must control enough stake to consistently win majority of slots
= With sqrt weighting, ~60%+ of total staked supply needed
Example: if total staked = 100M tokens, need ~60M (multiple whales)
This is significantly harder than the 33% needed on linear-stake chains
```

To profit from the attack, the attacker must extract more than
~2.1M tokens worth of value. Circuit breakers (pause trading, halt
mints) limit extractable value to well below this threshold.

### 9.3 Censorship Resistance

Validators cannot censor transactions because:
- Block building is deterministic (ordered by tx hash, not proposer preference)
- Proposer selection is random (can't predict who proposes next)
- Failed proposals skip to next proposer (no censorship via delay)
- Transactions can be submitted directly (no mempool gatekeeping)
- Dox_Dev identity means censorship is traceable and slashable

---

## 10. Summary Parameters

| Parameter | Value |
|-----------|-------|
| Consensus algorithm | BFT PoS (CometBFT-derived) |
| Block time | 1 second |
| Finality | Instant (1 block, 2/3+ commit) |
| Registered validators | Any number (compete for active slots) |
| Active validator set | 200 selected by sqrt-weighted lottery (equal voting power) |
| Min self-bond to register | 10,000 native tokens |
| Dynamic entry threshold | Weighted random lottery using sqrt(stake) — anyone eligible has a real chance |
| Max voting power per validator | Equal (1 validator = 1 vote in active set) |
| Base inflation | 7% APY (target) |
| Unbonding period | 14 days |
| Slash for equivocation | 5% of stake |
| Epoch length | 10,000 blocks (~2.8 hours) |
| Block size | 1MB target, 5MB max |
| Gas model | Fixed base fee + optional priority tip |
| State rent | 0.001 native / KB / block |
| Lanes | Consensus + Oracle (parallel) |
| Native randomness | VRF-based, per-block seed |
| Light client proof | ~1KB, verifiable in ~50K gas |
| Max block time | 3s before proposer skip |
| Governance param change | 3-14 day delay (depends on impact) |