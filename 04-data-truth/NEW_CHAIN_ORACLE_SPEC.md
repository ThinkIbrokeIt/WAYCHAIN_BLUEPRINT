# Native Oracle Consensus Model — Protocol Spec v0.2

**Key architectural decision:** Oracles (attesters) and validators are
separate sets. Different stake, different hardware, different slashing.
Defense in depth: an attacker must compromise BOTH to corrupt the chain
and its data simultaneously.

---

## 1. Core Architecture

### 1.1 Participants

| Role | Description | Stake | Entry Requirement |
|------|-------------|-------|-------------------|
| **Validator** | Orders and finalizes blocks | Validator stake (high) | ≥32,000 native tokens |
| **Attester** | Fetches and submits real-world data | Oracle bond (medium) | ≥5,000 native tokens |
| **Consumer** | Contract that requests/consumes data | None | Any contract |
| **Challenger** | Disputes a false attestation | Challenge bond | Any staked participant |
| **Backup Validator** | Validator who can serve as attester in emergencies | Validator stake | Auto-activated when attester count drops below threshold |

**Critical rule:** A validator CAN also be an attester by posting an
additional oracle bond. But the oracle bond is separate from the
validator stake. If an attester is slashed for bad data, only their
oracle bond is affected — their validator stake is untouched.
Conversely, if a validator is slashed for a consensus violation,
their oracle bond is untouched.

This creates defense in depth:
- **Compromise a validator** → can reorder/front-run txs, can't corrupt data
- **Compromise an attester** → can corrupt a feed, can't reorder blocks
- **Full attack** → must compromise both sets simultaneously

### 1.2 Separating the Roles

| Dimension | Validator | Attester |
|-----------|-----------|----------|
| **Primary job** | Consensus (ordering, finality) | Data (fetching, attesting) |
| **Hardware** | Fast node (SSD, 8+ core, 32GB RAM) | API node (reliable network, diverse sources) |
| **Uptime required** | ~99.9% (chain halts below threshold) | ~95% (feeds stall, chain keeps running) |
| **Stake** | High (32,000+) | Medium (5,000+) |
| **Slash for** | Double-sign, downtime | Wrong data, no-show, collusion |
| **Reward source** | Block rewards, tx fees | Oracle request fees |
| **Geographic need** | Diverse (block production) | Diverse (API source access) |
| **Max per operator** | 1 per physical node (or slashed) | 1 per feed (or slashed) |

### 1.3 Attester Set Selection

Attesters are permissionless — anyone can join by posting an oracle bond.
Selection for a specific feed:

1. **Register** — Post oracle bond, declare capabilities (API access,
   geographic region, supported data types)
2. **Qualify** — Must have: bond posted + accuracy > 80% + no active
   slashing in last 10,000 blocks + unique identity (no duplicate registration)
3. **Select** — Weighted random selection for each feed round:
   - Selected from qualified attesters
   - Weight: `(bond_size × reputation_score) / current_selected_feeds`
   - This naturally distributes work across the set
4. **Rotate** — Same attester can't serve same feed more than
   3 consecutive rounds (prevents capture)

**Minimum attester set size:** 100 globally (otherwise chain governance
sets emergency measures)

### 1.4 Data Types

| Type | Solidity Equivalent | Example |
|------|-------------------|---------|
| Numeric | int256 / uint256 | Token price: 1.2345 |
| String | string | "Candidate A wins election" |
| Bytes | bytes | Hash of a document |
| Timestamp | uint256 | 1719000000 |
| Address | address | Verified contract owner |
| Boolean | bool | "Was flight AA1234 delayed?" |
| Event | struct(topics[], data[]) | Transfer on tx 0x... |
| MerkleProof | bytes | Cross-chain state proof |

Compound types (structs) supported via ABI encoding.

### 1.5 Trust Tiers

| Tier | Min Attesters | Dispute Window | Oracle Bond | Latency | Use Case |
|------|--------------|----------------|-------------|---------|----------|
| 1 (Fast) | 3 | 10 blocks (~10s) | 500 native | 1 block | NFT floor, volume, simple feeds |
| 2 (Standard) | 10 | 100 blocks (~100s) | 1,000 native | 2 blocks | Token prices, sports outcomes |
| 3 (High) | 25 | 1,000 blocks (~1hr) | 5,000 native | 5 blocks | Insurance, identity, cross-chain |
| 4 (Legal) | 50 | 10,000 blocks (~1 day) | 10,000 native | 20 blocks | Land registry, court evidence |

**Design rule:** The cost to corrupt a feed must exceed any possible
profit from that manipulation.

---

## 2. Attestation Lifecycle

### 2.1 Request Phase

A consumer contract emits an `OracleRequest` event:

```
OracleRequest(
  requestId: bytes32,
  feedId: bytes32,
  dataSources: string[],
  dataType: uint8,
  aggregationMethod: uint8,
  minAttesters: uint8,
  disputeWindow: uint32,
  rewardAmount: uint256,
  callbackContract: address,
  callbackSelector: bytes4
)
```

If `feedId` matches an existing feed, parameters are reused. New feeds
register the first time they're requested. Multiple requests for the
same feed in the same block are batched.

### 2.2 Attestation Phase

Attesters independently fetch data and submit:

**1. Commit-Reveal (Tier 3+)**
```
commit = hash(value, sourceURI, attester, nonce, blockNumber)
reveal = (value, sourceURI, nonce)
```

**2. Direct Submit (Tier 1-2)**
```
Attestation(
  feedId: bytes32,
  value: bytes,
  sourceURI: string,
  sourceProof: bytes,           // TLS receipt or equivalent
  confidenceInterval: uint16,   // 0-10000 basis points (0.00%-100%)
  bond: uint256                 // locked per-attestation
)
```

**3. Source Verification Proof (optional, Tier 2+)**
- TLSN proof: attester proves they received a specific HTTP response
  from a specific server at a specific time (via TLSNotary or DECO)
- On-chain verification of the TLS session proof
- If the source is cryptographically verified, attestation weight is
  doubled (reward bonus for verifiable sources)

### 2.3 Aggregation Phase

**Median (default for numeric)**
- All revealed values sorted, median selected
- Resistant to outliers — one bad attester can't move the median

**Mean (low-variance data)**
- Arithmetic mean after trimming top/bottom 10%

**Mode (categorical)**
- Most common value wins (elections, binary events, classification)

**TWAP (volatile assets)**
- Time-weighted average over the attestation period
- Prevents flash loan oracle manipulation

**Custom aggregation**
- Consumer can provide a WASM aggregation function
- Gas-limited to prevent DoS

### 2.4 Dispute Window

**Who can challenge:** Any staked participant (validator or attester)
**Challenge bond:** 2x the total attestation reward for that round

**Process:**
1. `Dispute(feedId, roundNumber, evidence)`
2. Evidence: signed TLS receipt, cross-reference to another feed,
   cryptographic proof of source authenticity
3. Attesters can respond with counter-evidence
4. Validators vote on the dispute (oracle-free — validators use
   on-chain evidence only, not external data)
5. Attesters involved in the dispute cannot vote

**Outcomes:**

| Outcome | Result |
|---------|--------|
| Challenge succeeds (attester lied) | Attester's bond slashed → challenger: 50%, burned: 30%, honest attesters: 20%. Attester ejected from feed for 1,000 blocks |
| Challenge fails (attester correct) | Challenger's bond slashed → distributed to attesters. Challenger banned from disputing same feed for 10,000 blocks |
| No dispute | Attestation finalizes, bonds released |

**Frivolous dispute prevention:**
- Challenge bond = 2x total attestation reward (costs more than it's worth)
- Failed challengers banned from that feed for 10,000 blocks
- Repeat frivolous challengers (3+ failed) lose challenge privilege for 100,000 blocks

### 2.5 Finalization

1. Aggregated value committed to chain state
2. Callback contract invoked with the value
3. Attester bonds released (minus slashed amounts)
4. Rewards distributed: proportional to participation × accuracy × reliability
5. `DataFinalized(feedId, value, blockNumber)` emitted
6. Value truth-anchored in Binary Journal

---

## 3. Economic Model

### 3.1 Oracle Bond Structure

Two-layer bond:

```
registrationBond = 5,000 native tokens (one-time, global)
perFeedBond = tierMinimum + (numberOfFeeds × 100)
```

An attester serving 5 Tier 2 feeds:
```
total = 5,000 + 5 × 1,000 = 10,000 native
```

Bond is slashable per-attestation, not per-bond. An attester who
attests for 5 feeds and gets slashed on 1 loses only that
attestation's bond, not their entire registration or their other feeds.

### 3.2 Rewards

```
attesterReward = totalReward × weight / totalWeight

weight = baseWeight × accuracyMultiplier × reliabilityMultiplier × sourceVerifiedMultiplier

baseWeight = 1.0 (default), 1.5 (if source has TLS proof)
accuracyMultiplier = 1.0 (within 1σ), 0.5 (within 2σ), 0 (beyond 2σ)
reliabilityMultiplier = historical_accuracy / 0.95 (capped at 1.2)
sourceVerifiedMultiplier = 1.5 (if TLS proof provided), 1.0 (otherwise)
```

### 3.3 Slashing Conditions

| Condition | Slash | Effect |
|-----------|-------|--------|
| Value > 3σ from median | 25% of per-feed bond | Reduced selection weight |
| Successful dispute | 50% of per-feed bond | Ejected from feed 1,000 blocks |
| No-show (registered, didn't attest) | 10% of per-feed bond | Selection weight halved |
| Collusion (proven coordination) | 100% of ALL bonds + banned | Permanent ejection from attester set |
| Duplicate identity | 100% of ALL bonds + banned | Permanent ejection |

### 3.4 Fee Market

Consumer contracts pay for oracle data:

```
totalFee = baseFee × tierMultiplier × sourceCount
baseFee = 10 native tokens
tierMultiplier: 1x (T1), 2x (T2), 5x (T3), 20x (T4)
sourceCount: linear multiplier per data source
```

Fee distribution:
- 80% to attesters (proportional per attestation)
- 10% to validators (for processing oracle transactions)
- 10% to protocol treasury (R&D, dispute reserves)

---

## 4. Data Source Verification (Gap 1)

This is the hardest oracle problem: how does the protocol know an
attester actually queried the claimed source and didn't just make up
the value?

### 4.1 TLSNotary Proofs (Tier 2+)

TLSNotary (TLSN) allows an attester to prove they had a specific
TLS session with a specific server, without revealing the TLS
session keys.

**Flow:**
1. Attester connects to the data source via a modified TLS handshake
2. TLSN generates a cryptographic proof that the attester received
   a specific response from the server at a specific time
3. The proof includes:
   - Server certificate hash (verifying the domain)
   - Request/response hash (verifying the data)
   - Timestamp from the TLS handshake
4. Attester submits the TLSN proof alongside their attestation
5. The protocol's precompile contract verifies the TLSN proof

**Limitations:**
- Requires the attester to run modified TLS client software
- Only works for HTTPS APIs (not for private data sources)
- Proof generation adds ~200ms latency
- Not all APIs can be proven (Cloudflare, some CDNs block automated TLS)

### 4.2 DECO Proofs (Tier 3+)

DECO (by Chainlink Labs / Cornell) extends TLSN to work with
session-level proofs, enabling oracle data from paywalled or
authenticated APIs without revealing API keys.

**Additional capability:**
- Attester proves they queried an authenticated API endpoint
- Does not reveal the API key used
- Does not reveal the full session — only the specific response

### 4.3 Reputation-Based Trust (All Tiers)

Not all data sources support cryptographic proof. For those that
don't, trust comes from:

1. **Redundancy** — Multiple independent attesters query the same
   source. Collusion requires compromising >50% of them.
2. **Economic stake** — Attesters have real money at risk.
3. **Historical track record** — Binary Journal tracks every attester's
   accuracy over time. One bad value tanks their reputation.
4. **Unpredictable spot checks** — The protocol occasionally sends a
   known-value test to random attesters. Wrong answer = automatic slash.
5. **Source diversity requirement** — Consumer contracts can require
   attesters to use fundamentally different sources (CoinGecko AND
   Binance AND Kraken, not three CoinGecko mirrors).

### 4.4 Source Diversity Enforcement

Contracts specify minimum source diversity:

```solidity
// "At least 3 different API providers, at least 5 attesters per API"
OracleRequest({
    dataSources: ["coingecko", "binance", "kraken", "coinbase", "uniswap"],
    minAttestersPerSource: 5,
    minUniqueSources: 3
});
```

The protocol enforces this at attestation time — if all submissions
come from one source, the round is invalid and attesters must re-attest
from different sources.

### 4.5 Multi-Source Conflict Resolution

If two sources disagree:
- Cross-reference with Binary Journal's historical accuracy ratings
- De-weight sources that have been historically wrong on this feed
- Escalate to next trust tier if the deviation exceeds circuit breaker
- Pause the feed if no consensus emerges after escalation

---

## 5. Validator/Attester Separation (Gap 2)

### 5.1 Why They Must Be Separate

| Attack | If Same Set | If Separate Sets |
|--------|------------|-----------------|
| Compromise validator node | Can corrupt blocks AND data | Can only corrupt blocks |
| Compromise attester node | Can corrupt data AND reorg | Can only corrupt data |
| Bribe to manipulate a feed | One bribe compromises both | Two bribes needed |
| Sybil attack | Need one large stake | Need two separate stakes |
| Infrastructure failure | Node down = no blocks + no data | Node down = only one role affected |

### 5.2 Physical Separation

Attesters and validators should operate on different:
- **Hardware** — Different machines, different hosting providers
- **Networks** — Different internet connections
- **Jurisdictions** — Different legal frameworks for data requests
- **Software stacks** — Different node implementations

The protocol does not enforce this physically, but:
- Binary Journal tracks the IP ranges and hosting providers of attesters
- Governance can flag suspicious clustering (all attesters from one cloud)
- Validator-attester overlap is limited to 20% of the combined set

### 5.3 Economic Separation

| | Validator Stake | Attester Bond |
|---|---|---|
| Minimum entry | 32,000 native | 5,000 native |
| Slash for consensus fault | Up to 100% | Not affected |
| Slash for oracle fault | Not affected | Up to 100% per feed |
| Rewards | Block rewards + fees | Oracle request fees |
| Withdraw | Warm-up period | Immediate (minus active attestations) |

### 5.4 Cross-Role Operator

A single legal entity CAN operate both a validator and an attester,
but:

1. They must be on separate physical infrastructure
2. They must register both roles with different operational keys
3. The attester's key cannot sign validator messages and vice versa
4. Combined stake (validator + oracle bonds) is counted for governance
   voting weight, BUT the oracle bond doesn't increase validator
   consensus power

### 5.5 Emergency Fallback

If the attester set drops below 50% of minimum (e.g., < 50 attesters):

1. Validators are automatically activated as backup attesters
2. They don't need to post the oracle bond (covered by their validator stake)
3. They use a lower reward multiplier (0.5x) to incentivize dedicated
   attesters to rejoin
4. Once the attester set recovers above threshold, validators are
   deactivated as attesters

This ensures oracle liveness even if the attester market is thin,
without compromising the separation in normal operation.

---

## 6. Liveness Guarantees (Gap 3)

### 6.1 Attester Commitment

When an attester is selected for a feed round, they must commit to
attest within the window. If they fail:

| Missed commitment | Penalty |
|------------------|---------|
| 1 per 100 rounds | Warning |
| 2 per 100 rounds | 10% of per-feed bond slashed |
| 3+ per 100 rounds | Ejected from feed selection for 10,000 blocks |

### 6.2 Over-Subscription

The protocol selects 30% more attesters than the minimum required.
If the min is 10, it selects 13. This handles "no-shows" without
delaying finality.

### 6.3 Feed Heartbeats

Each feed has a maximum update interval:
- Tier 1: 10 blocks (no update = "stale" flag set)
- Tier 2: 100 blocks
- Tier 3: 1,000 blocks
- Tier 4: 10,000 blocks

If a feed misses its heartbeat:
1. Feed is marked "STALE" — consumer contracts can check this status
2. A special heartbeat attestation round is triggered automatically
3. If the feed misses 3 consecutive heartbeats, governance is notified
4. After 10 missed heartbeats, the feed is paused and attesters for
   that feed lose 10% of their per-feed bond

### 6.4 Circuit Breakers

Three automatic triggers that escalate a feed's security:

**1. Deviation Circuit Breaker**
If the aggregated value changes more than X% from the previous
finalized value in a single round:
- X = 10% for prices, 50% for volatile feeds, 100% for event data
- Triggered feed auto-escalates to next trust tier for 5 rounds
- All attesters must re-attest with source proofs

**2. Staleness Circuit Breaker**
If a feed hasn't been updated in N blocks:
- N = 2x the feed's heartbeat interval
- Feed enters "recovery mode" — maximum attesters, extended dispute
- Governance notified

**3. Consensus Failure Circuit Breaker**
If attesters can't reach consensus (e.g., split vote, no majority):
- Round invalidated, attesters lose 5% of their bond
- New round starts with higher tier selection
- After 3 consecutive failures, feed paused for governance review

### 6.5 Attester Incentives for Liveness

Attesters earn a small "heartbeat bonus" for regularly updating feeds:
```
heartbeatBonus = baseBonus × consecutiveUpdates × feedTier
baseBonus = 1 native token per heartbeat
consecutiveUpdates: multiplier from 1.0 to 3.0 (capped at 100 updates)
```

This means attesters who reliably update feeds earn up to 3x base
reward beyond the request fees. This creates a subscription-style
income for reliable attesters.

### 6.6 Emergency Attestation

Any validator can submit an emergency attestation if:
1. A feed hasn't been updated in > 2x heartbeat interval
2. The validator stakes 5x the normal attestation bond
3. The attestation goes through an expedited dispute (10 blocks)
4. Validator gets 2x normal attester reward if correct

This ensures that a feed CAN always be updated, even if all dedicated
attesters for that feed are offline simultaneously.

---

## 7. EVM Integration

### 7.1 Native Opcodes

**`ORACLE_READ(blockNumber, feedId) → bytes`**
Returns the finalized oracle value for a feed at a specific block.
200 gas (same as SLOAD) for historical reads. Current-block reads
cost 1,000 gas (merkle proof verification).

```solidity
uint256 price = uint256(ORACLE_READ(block.number - 1, PRICE_FEED_PLS_USD));
```

**`ORACLE_REQUEST(feedId, params) → requestId`**
Creates an oracle request. Returns the request ID. Contract listens
for the callback event.

**`ORACLE_VERIFY(proof, value, feedId, blockNumber) → bool`**
Verifies a value was finalized at a specific block. For L2s and
cross-chain verification.

**`ORACLE_STATUS(feedId) → (bool active, uint256 lastUpdate, uint256 value)`**
Returns the current status of a feed (is it current, stale, paused).

### 7.2 Precompile Contracts

**`0x0C` — OracleAggregator** Creates derived feeds (PLS/USD × USD/EUR = PLS/EUR).

**`0x0D` — OracleScheduler** Schedules recurring oracle requests.
"Update this feed every 10 blocks." Removes keeper bots entirely.

**`0x0E` — OracleVerifier** Verifies external chain proofs (SPV, zkBridge, IBC).

**`0x0F` — TLSVerifier** Verifies TLSNotary/DECO proofs from data sources.
Checks server certificate, request hash, response hash, timestamp.

### 7.3 Gas Costs

| Operation | Gas Cost |
|-----------|----------|
| Read finalized value (historical) | 200 |
| Read current value | 1,000 |
| Request new data | 5,000 |
| Submit attestation | 10,000 |
| Submit attestation + TLS proof | 25,000 |
| Dispute | 20,000 |
| Finalize (no dispute) | 3,000 |
| Finalize (dispute resolved) | 50,000 |
| Feed status check | 800 |

---

## 8. Performance & Scalability

### 8.1 Separate Execution Lane

Attestations execute in a dedicated oracle lane:

- **Consensus lane:** User transactions, contract execution, block
  production (validators only)
- **Oracle lane:** Attestation submissions, aggregation, dispute
  processing (attesters only)

Both lanes finalize in the same block. The oracle lane has a separate
gas limit so it can't congest normal transactions.

### 8.2 Feed Batching

Multiple requests for the same feed in the same block:
- Attesters submit ONE attestation per feed per block
- Protocol aggregates once, delivers to all consumers
- Saves 5-10x gas vs independent requests

### 8.3 Push vs Pull

| Mode | Operation | Best For |
|------|-----------|----------|
| Subscribe (push) | Attesters update on schedule or threshold change | Price feeds, time-sensitive data |
| On-demand (pull) | Attesters submit only when explicitly requested | Event-driven, infrequent data |
| Hybrid | Subscribe to updates, pull latest anytime | General purpose |

### 8.4 Latency Targets

| Tier | Block-to-Final |
|------|----------------|
| 1 | 1-2 blocks (~1-2s) |
| 2 | 5-10 blocks (~5-10s) |
| 3 | 100-200 blocks (~2-5min) |
| 4 | 10,000+ blocks (~1+ day) |

---

## 9. Security Model

### 9.1 Attack Vectors

| Attack | Mitigation |
|--------|------------|
| **Flash loan manipulation** | TWAP + min-block-delay (can't manipulate a block-old value) |
| **51% attester collusion** | Requires controlling >50% of attester bonds + source diversity enforcement makes this harder |
| **Validator-atterster collusion** | Must compromise both sets + separate infrastructure + limited overlap |
| **Data source compromise** | Multi-source + TLS proofs + source diversity requirement |
| **Front-running attestations** | Commit-reveal (Tier 3+), median aggregation (all tiers) |
| **Frivilous disputes** | 2x challenge bond + banning |
| **Long-range attack** | State checkpointing, historical data immutable in Binary Journal |
| **Censorship** | Permissionless attester entry — anyone can join with a bond |
| **Sybil attesters** | Unique identity via oracle bond (creates economic cost per identity) |

### 9.2 Economic Security

Cost to corrupt a Tier 2 price feed (10 attesters, 1,000 bond each):

```
attackCost = 10 × 1,000 × tokenPrice = 10,000 tokens
+ reputational cost (attester ejection, historical record loss)
+ challenge risk (anyone can dispute and claim the slash bounty)
```

For the attack to profit, extracted value must exceed this cost.
Circuit breakers (deviation limits, TWAP) limit extractable value to
~1-2% of the pool. For a $10M pool, max extractable = $100K-200K.
Attack cost at $1/token = $10K + rep. Attack is uneconomical.

### 9.3 Gradual Escalation

If suspicious activity is detected:
1. Feed auto-escalates to next trust tier
2. More attesters assigned
3. Commit-reveal enforced
4. Dispute window extended
5. Deviation thresholds tightened

---

## 10. Binary Journal Integration

### 10.1 Truth Anchoring

Every finalized oracle value is anchored:

```
truthRoot = hash(previousTruthRoot, blockNumber, feedId, value, attesterSetHash)
```

This creates an immutable chain of verified data. Any historical
value can be:
- Verified against the truth chain
- Proved to a court/auditor via merkle proof
- Traced back to original attesters and sources

### 10.2 Attester Reputation

Binary Journal tracks per-attester:
- Total attestations submitted (by tier, by data type)
- Dispute win/loss ratio
- Deviation from median over time (systematic bias detection)
- Source verification rate (% of attestations with TLS proofs)
- Bond status and slash history

High-reliability attesters get:
- Priority selection for high-reward feeds
- Lower bond requirements after 10,000 accurate attestations
- Higher weight in aggregation

Low-reliability attesters:
- Reduced selection probability
- Higher bond requirements after slashing events
- Automatic ejection if accuracy drops below 60%

### 10.3 3·6·9 Knowledge Accumulation

The oracle network's verified data feeds into Binary Journal's
knowledge protocol:

- **3-second** — Live feed values (current state)
- **6-minute** — Short-term TWAPs, rolling averages
- **9-hour** — Daily summaries, attester reliability rankings
- **3-day** — Source quality rankings (which APIs are most reliable)
- **6-week** — Feed performance (which tiers/attesters produce best data)
- **9-month** — Network-wide oracle health trends

This becomes a self-improving system: the protocol learns which
sources, attesters, and configurations produce the most reliable data.

---

## 11. Governance & Upgradability

### 11.1 Governance Parameters

| Parameter | Change Mechanism | Delay |
|-----------|-----------------|-------|
| Trust tier parameters | Governance vote | 7 days |
| Feed registration | Feed curator (elected) | 1 day |
| Circuit breaker thresholds | Governance vote | 3 days |
| Bond minimums | Governance vote | 14 days |
| Slash percentages | Governance vote | 14 days |
| Heartbeat intervals | Governance vote | 7 days |
| Emergency fallback threshold | Governance vote | 3 days |

### 11.2 Feed Curation

- Anyone can propose a new feed
- Feed curators (elected by token holders) approve/reject
- Approved feeds registered in the feed registry
- Inactive feeds archived after 90 days without a request

### 11.3 Emergency Actions

Governance can pause a feed, force-resolve a dispute, or freeze an
attester's bond. Requires:
- Multi-sig of 5/7 elected security council members
- Public disclosure within 24 hours
- Governance review within 7 days

---

## Next Steps

The spec now addresses all three gaps:

1. **Data source verification** → TLSNotary/DECO proofs + reputation +
   source diversity enforcement + multi-source conflict resolution
2. **Validator/attester overload** → Separate sets, separate stake,
   separate hardware, separate slashing, emergency fallback
3. **Liveness guarantees** → Over-subscription (30%), feed heartbeats,
   auto-escalation circuit breakers, emergency attestation by validators

Remaining spec documents to write:
1. **EVM spec** — Precise opcode definitions, precompile interfaces
2. **Validator client spec** — Consensus protocol modifications
3. **Attester client spec** — Data fetching, proof generation, slashing
4. **Economic model with real parameters** — Bond curves, attack cost analysis
5. **Binary Journal integration spec** — Truth anchor structure, 3·6·9 accumulation