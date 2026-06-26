# WayChain Token Economics — Protocol Spec v1.0

**Token:** WAY (native coin, no pre-mine, no pre-sale)
**Supply curve:** Dynamic — governed by protocol parameters, not fixed hard cap
**Core principle:** One verified human = one equal voice. Capital does not capture control.

---

## 0. Design Philosophy

WayChain does not optimize for:
- Token holder wealth maximization
- Whale accumulation
- Exchange dominance
- VC exit liquidity

WayChain optimizes for:
- **Participation** — Any verified human can contribute meaningfully
- **Sustainability** — Fees at cost, not market
- **Anti-capture** — Economic and governance power cannot concentrate
- **Utility** — The token is useful first, valuable second

**Token value follows from usefulness, not speculation.**
If the chain is useful, the token has value. If we build it to have value first,
we build the same system everyone is fleeing.

---

## 1. Core Identity Binding

### 1.1 One Dox_Dev Badge = One Validator

| Principle | Rule |
|-----------|------|
| Validators per human | **Exactly 1.** One Dox_Dev-verified human operates one validator node. |
| Wallets per validator | **1 project wallet.** All validator operations use a single on-chain identity. |
| Machines per validator | **1 dedicated machine.** Shared infrastructure is a security violation. |
| Badge sharing | **Not allowed.** A validator cannot be "operated by" someone else's badge. |

This fundamentally prevents:
- A whale running 50 validators through capital advantage
- An exchange controlling 40% of the active set
- A single entity hiding behind multiple identities

The validator set is 200 verified humans. Period.

### 1.2 One Dox_Dev Badge = One Oracle

A badge holder may also operate an oracle **on a separate machine** with a **separate project wallet**. Compromise of one does not compromise the other. A single human can contribute both security and data, but the infrastructure is completely isolated.

### 1.3 Badge Loss or Revocation

If a validator's Dox_Dev badge is revoked (fraud, scam, violation):
- Validator is immediately removed from the active set
- Stake enters a 30-day unbonding period
- If the revocation was fraudulent, the validator can appeal

This is real accountability. No other L1 has this.

---

## 2. Supply Curve

### 2.1 No Fixed Cap — Governed Supply

| Parameter | Value |
|-----------|-------|
| Target inflation | 7% APY at genesis |
| Range | 3-10% (governance-adjustable, 2/3 supermajority + 90-day timelock) |
| Adjustment cadence | Every 90 days (epoch-aligned) |
| Supply grows from | Validator/delegator rewards, treasury allocation |
| Supply shrinks from | Fee burns, state rent burns, slash burns, registration burns |

Unlike v0.1, inflation is **not auto-adjusted by staking ratio**. The progressive
staking curve (Section 3) handles staker incentive alignment. Inflation is
set by governance within a narrow band, with high friction to change.

### 2.2 No Pre-Mine, No Pre-Sale

```
Pre-mine:             0%
Pre-sale:             0%
VC allocation:        0%
Founder allocation:   0%
```

At genesis, **every Dox_Dev-verified human receives an equal base allocation.**
The amount is set so that a verified human can participate meaningfully
without being priced out. The exact number is finalized when the genesis
cohort size is known.

### 2.3 Genesis Supply Distribution (First Year Issuance)

| Allocation | % of Issuance | Purpose |
|------------|--------------|---------|
| Validator/delegator rewards | 85% | Block rewards, staking incentives |
| Protocol treasury | 10% | Development, audits, VPS subsidies, grants |
| Genesis validator bootstrapping | 5% | Year-1 VPS cost coverage for the first 200 |

**All issuance goes to active participants.** No dormant wallets accumulating.

---

## 3. Progressive Staking — The Anti-Whale Engine

### 3.1 Progressive Reward Brackets

Staking rewards are not linear. Smaller stakes earn higher returns, flattening
the wealth-compounding advantage of large holders.

```
Bracket 1:        1 - 1,000 tokens      → 15% APY
Bracket 2:    1,001 - 10,000 tokens     →  8% APY
Bracket 3:   10,001 - 100,000 tokens    →  4% APY
Bracket 4:  100,001 - 1,000,000 tokens  →  2% APY
Bracket 5:  1,000,001+ tokens           →  1% APY
```

**How it works:** A staker with 50,000 tokens earns:
- 15% on the first 1,000 tokens (150)
- 8% on tokens 1,001-10,000 (720)
- 4% on tokens 10,001-50,000 (1,600)
- **Total: 2,470 tokens/year = 4.94% effective APY**

A small staker with 500 tokens earns 15%. A whale with 500K tokens earns
much less per token on their excess capital.

### 3.2 Why This Matters

| Staker Size | Tokens | Effective APY | Annual Reward |
|-------------|--------|---------------|---------------|
| Small | 100 | 15.00% | 15 |
| Medium | 5,000 | 9.24% | 462 |
| Large | 50,000 | 4.94% | 2,470 |
| Whale | 500,000 | 2.22% | 11,104 |
| Mega-whale | 5,000,000 | 1.22% | 61,104 |

**Without progressive curve (flat 7%):**

| Staker Size | Tokens | Effective APY | Annual Reward |
|-------------|--------|---------------|---------------|
| Small | 100 | 7% | 7 |
| Medium | 5,000 | 7% | 350 |
| Large | 50,000 | 7% | 3,500 |
| Whale | 500,000 | 7% | 35,000 |
| Mega-whale | 5,000,000 | 7% | 350,000 |

The gap between small and whale rewards shrinks from 50,000x to 4,000x.
Whales are still rewarded — proportionally to their contribution — but cannot
use compounding to permanently outpace the rest of the community.

---

## 4. Validator Economics — 200 Equal Seats

### 4.1 Equal Base Reward

Every active validator earns the **same base reward** regardless of stake size.

```python
def distribute_block_rewards(block):
    issuance = calculate_issuance(block.number)
    
    # Split issuance
    validator_pool = issuance * 0.85
    treasury_pool = issuance * 0.10
    bootstrap_pool = issuance * 0.05
    
    # Validator rewards are EQUAL per seat, not proportional to stake
    active_count = len(active_validators)  # max 200
    base_reward = validator_pool / active_count
    
    for validator in active_validators:
        validator.reward += base_reward
        # Validator keeps commission
        # Delegators split their validator's share among themselves
        commission = base_reward * validator.commission_rate
        validator.self += commission
        
        delegator_share = base_reward - commission
        for delegator in validator.delegators:
            delegator.reward += delegator_share * (delegator.stake / validator.total_delegated)
    
    # Gas fees: 100% to validators (not split with treasury)
    # This rewards the active security providers
    gas_share = block.total_gas_fees / active_count
    for validator in active_validators:
        validator.reward += gas_share
```

### 4.2 Validator Requirements

| Requirement | Value |
|-------------|-------|
| Minimum self-bond | 100 WAY (low enough for any verified human) |
| Maximum self-bond | None (but capped utility — progressive curve applies) |
| Hardware | Dedicated machine, separate from oracle |
| Wallet | Single project wallet per validator |
| Dox_Dev badge | Active, Level 2+ |
| Uptime target | 95%+ (graced for first 90 days) |

The low minimum stake ensures that capital is not a barrier to entry.

### 4.3 Delegation

A validator can accept delegations. Delegators earn rewards through the
validator, minus commission. The progressive staking curve applies to the
**total** (self-bond + delegations) so a validator cannot bypass the curve
by splintering into multiple entities.

---

## 5. Fee Model — At Cost, Not Market

### 5.1 Fiat-Pegged Fee Calculation

Fees are calculated in **USD-equivalent** and paid in WAY at the current
oracle rate. This ensures fees stay cheap in real terms regardless of
token price volatility.

```python
def calculate_fees(block):
    # Oracle provides WAY/USD price (updated every block by attesters)
    way_usd = oracle.get_price("WAY/USD")
    
    # Target fees in USD (fiat-pegged, governance-adjustable)
    CONFIG = {
        "consensus_tx_usd": 0.001,    # $0.001 per simple tx
        "complex_tx_usd": 0.005,      # $0.005 per complex tx
        "state_rent_kb_usd": 0.0001,  # $0.0001/KB/block
        "oracle_attestation_usd": 0.01,  # $0.01 per attestation
    }
    
    # Convert to WAY at current oracle rate
    consensus_fee_way = CONFIG["consensus_tx_usd"] / way_usd
    state_rent_way = CONFIG["state_rent_kb_usd"] / way_usd
    
    return consensus_fee_way, state_rent_way
```

**If WAY = $10:** Consensus fee = 0.0001 WAY/tx ($0.001). Feels like nothing.
**If WAY = $0.001:** Consensus fee = 1 WAY/tx ($0.001). Feels like nothing.

The cost to the user is always ~$0.001. The protocol earns in WAY, which has
real value because it's useful.

### 5.2 Fee Types (v1.0)

| Fee Type | USD Target | Destination | Burn % |
|----------|-----------|-------------|--------|
| **Simple tx** | $0.001 | Validators (100%) | 0% |
| **Complex tx (contract interaction)** | $0.005 | Validators (100%) | 0% |
| **Priority tip** | Optional, capped at 2x base | Proposer (100%) | 0% |
| **State rent** | $0.0001/KB/block | Burn (60%), Validators (40%) | 60% |
| **Template deployment** | $10 (one-time) | Template author (40%), Auditor (30%), Treasury (30%) | 0% |
| **Dox_Dev Level 2** | Free (or minimal gas) | — | 0% |
| **Dox_Dev Level 3** | $50 (one-time) | Curators (30%), Endorsers (20%), Treasury (50%) | 0% |
| **Validator registration** | $5 (one-time) | Burn (100%) | 100% |
| **Oracle registration** | $5 (one-time per feed) | Burn (100%) | 100% |

### 5.3 Why Fees Go 100% to Validators

In v0.1, gas fees split 50/50 with treasury. In v1.0, validators get 100%
of gas fees. This makes the economics work during bootstrap:

- Treasury gets its share from issuance (10%) and state rent (10%)
- Validators need every dollar they can earn early on
- Once volume scales, the validator revenue from fees alone exceeds issuance

---

## 6. Bootstrap Period — Year 1

### 6.1 The Problem

At genesis, token price is unknown. If WAY starts at $0.01 with 500 tx/day,
a validator earns ~$26/month. A $50/month VPS costs nearly double that.
Validators would be running at a loss.

Without validators, the chain doesn't exist. This is the bootstrap trap.

### 6.2 The Solution — Treasury VPS Subsidy

For the first 365 days, the protocol treasury covers VPS costs for active
validators:

```python
def bootstrap_subsidy(validator, block):
    if block.number > BLOCKS_IN_YEAR:
        return 0  # Subsidy ends after year 1
    
    # Validator must have >95% uptime for the prior epoch
    if validator.uptime < 0.95:
        return 0
    
    # Fixed monthly subsidy: $50 equivalent in WAY
    subsidy_usd = 50.0 / 30 / 86400  # Per block
    way_usd = oracle.get_price("WAY/USD")
    subsidy_way = subsidy_usd / way_usd
    
    return subsidy_way
```

**Funding source:** The 5% bootstrap allocation from issuance (Section 2.3)
plus any surplus from the 10% treasury allocation. Estimated annual cost
for 200 validators:

```
200 validators × $50/month × 12 months = $120,000
```

At $0.01 WAY: 12,000,000 WAY
At $0.10 WAY: 1,200,000 WAY
At $1.00 WAY: 120,000 WAY

The annual issuance at 7% on 100M supply = 7,000,000 WAY. The 5%
bootstrap allocation = 350,000 WAY/year. This is sufficient if the token
finds $0.34+ price. Below that, the treasury may need to supplement from
the 10% general allocation.

### 6.3 Subsidy Sunset

After 365 days, the subsidy phase is complete. If the chain hasn't found
enough volume to sustain validators without subsidy, it means the chain
hasn't found product-market fit. Continuing the subsidy delays the signal.

A 90-day grace period (days 366-455) with a 50% reduced subsidy gives
validators time to decide.

---

## 7. Governance — One Human, One Vote

### 7.1 Governance Power

| Source | Weight |
|--------|--------|
| Dox_Dev badge (Level 2+) | 1 vote per human |
| Token ownership | **0 votes** |

Token weight does not touch governance. Period.

A whale with 5 million WAY has the same governance power as a small staker
with 100 WAY. Both need a Dox_Dev badge to participate.

### 7.2 What Governance Controls

| Parameter | Threshold | Timelock |
|-----------|-----------|----------|
| Inflation rate (3-10% band) | 2/3 supermajority | 90 days |
| Fee USD targets | 2/3 supermajority | 30 days |
| Progressive bracket boundaries | 2/3 supermajority | 90 days |
| Validator set size (100-300) | 2/3 supermajority | 90 days |
| Treasury allocation percentages | 3/4 supermajority | 90 days |
| Emergency parameter freeze | Simple majority | Immediate |

### 7.3 What Is Immutable (Genesis-Enforced)

- One badge = one validator (cannot be changed)
- One badge = one vote (cannot be changed)
- Progressive staking exists (brackets can be adjusted, curve cannot be removed)
- Fiat-pegged fee model (peg targets can be adjusted, peg mechanism is fixed)
- No pre-mine / no pre-sale (cannot be added)
- Burn mechanisms (cannot be removed, percentages can be adjusted)

---

## 8. Slashing Economics

### 8.1 Slash Schedule

| Violation | Amount | Distribution |
|-----------|--------|-------------|
| Equivocation (double-sign) | 5% of total staked | 50% burned, 25% reporter, 25% honest validators |
| Wrong attestation | Per-feed bond (100 WAY) | 50% burned, 25% challenger, 25% honest attesters |
| Downtime (>5% in epoch) | 0.5% of total staked | 50% burned, 50% active validators |
| Guardian fraud | Guardian bond (100 WAY) | 50% burned, 50% to recovery target |
| Badge revocation | 10% of total staked | 50% burned, 50% treasury |

**Badge revocation slashing** is unique to WayChain. If Dox_Dev revokes a
validator's badge for fraudulent behavior, the validator loses additional
stake. This creates real-world accountability backed by economic penalty.

### 8.2 Deflation from Slashing

| Slash Type | Annual Expected | Tokens Burned |
|------------|----------------|--------------|
| Equivocation | 1-2 | ~1,600-3,200 |
| Wrong attestation | 5-10 | ~2,500-5,000 |
| Downtime | 20-50 | ~3,200-8,000 |
| Guardian fraud | <5 | ~250-500 |
| Badge revocation | Rare | ~10,000+ |
| **Total** | | **~17,000+/year** |

---

## 9. Token Utility

| Utility | Who Uses It | Why They Need It | Weight |
|---------|-----------|-----------------|--------|
| **Staking** | Validators, Delegators | Earn rewards, secure the chain | Primary |
| **Gas** | All users | Execute transactions | Required |
| **State rent** | Contract owners | Keep state alive | Required |
| **Template fee** | Deployers | Deploy from audited templates | Required |
| **Dox_Dev badge** | Developers | Get verified, deploy programs | Required |
| **Governance** | Badge holders | Change protocol parameters | 1 human = 1 vote |
| **Oracle bond** | Attesters | Participate in data attestation | Security |
| **Guardian stake** | Guardians | Approve account recoveries | Security |
| **Validator registration** | Validator candidates | Register for the active set | One-time |

---

## 10. Comparison: WayChain vs. Typical L1

| Feature | Typical L1 | WayChain |
|---------|-----------|----------|
| Validators per entity | Unlimited (capital determines) | **1 per Dox_Dev badge** |
| Staking reward curve | Linear or flat | **Progressive (smaller = higher %)** |
| Governance weight | 1 token = 1 vote | **1 badge = 1 vote** |
| Fees | Market-auction (spike during demand) | **Fiat-pegged (always cheap)** |
| Pre-mine | 10-50% to team/VCs | **0%** |
| Genesis distribution | Public sale (first money in wins) | **Equal to every verified human** |
| Validator accountability | Economic only (slashing) | **Economic + identity (badge revocation)** |
| Bootstrap support | None (validators eat the loss) | **Treasury VPS subsidy (year 1)** |
| Oracle separation | Same machine or unspecified | **Separate machine + wallet required** |
| Anti-whale mechanism | None | **Progressive curve + one-seat rule** |

---

## 11. Summary

| Parameter | Value |
|-----------|-------|
| Chain name | WayChain |
| Token ticker | WAY |
| Initial inflation | 7% APY |
| Inflation range | 3-10% (2/3 supermajority + 90-day timelock) |
| Validator set size | 200 (1 per Dox_Dev badge) |
| Validator rewards | Equal per seat (not proportional to stake) |
| Minimum self-bond | 100 WAY |
| Progressive staking | 15% / 8% / 4% / 2% / 1% brackets |
| Governance | 1 badge = 1 vote (token weight = 0) |
| Simple tx fee | ~$0.001 (fiat-pegged) |
| State rent | ~$0.0001/KB/block (fiat-pegged) |
| State rent burn | 60% |
| Slash burn | 50% |
| Bootstrap subsidy | Year 1 VPS coverage ($50/mo per validator) |
| Pre-mine | 0% |
| Genesis allocation | Equal per Dox_Dev-verified human |
| Supply model | Governed dynamic (no hard cap) |

---

## 12. Remaining Questions for Expert Review

1. **Progressive curve granularity** — Are 5 brackets the right number? Should
   brackets be continuous (smooth function) instead of stepped?

2. **Bootstrap subsidy quantum** — Is $50/month enough or too much? Should it
   be regionally adjusted (lower in lower-cost areas)?

3. **Fiat-pegged oracle dependency** — The fee model depends on an accurate
   WAY/USD price from the oracle. If the oracle is compromised, fees could
   spike or collapse. Need oracle redundancy.

4. **Genesis allocation amount** — What's the right number per verified human?
   High enough to be meaningful, low enough that a whale can't buy 10,000 badges.

5. **Validator seat cap** — 200 is the initial target. Should it scale with
   adoption, or stay fixed? Fixed creates scarcity value. Scalable prevents
   governance capture by a static group.

6. **Badge cost** — If Dox_Dev Level 2 is free (just gas), badge farming is
   possible. If it costs $50, it excludes the global poor. Tension between
   Sybil resistance and accessibility.
