# Mineral Rights Tokenization — WayChain Real-World Asset Protocol v0.1

**"Prove it's there. Prove you own it. Sell the right to leave it in the ground."**

Gold mines destroy the earth. Cyanide in water tables. Open pits visible
from space. Tailings ponds that poison for generations. And for what —
gold that sits in vaults, doing nothing, while the land is never the same.

This protocol flips the economics. **The most valuable gold is the gold
that stays underground.** The miner gets paid. The environment stays intact.
WayChain (or its partner entity) holds the mineral rights as a verifiable,
auditable, tradeable real-world asset.

---

## 0. The Problem Mining Creates

| Impact | Per Gold Mine (avg) |
|--------|---------------------|
| Land disturbed | 1,000+ acres |
| Water consumed | 200+ million gallons/year |
| Cyanide used | 1+ million tons/year (heap leaching) |
| Tailings waste | 20+ million tons/year |
| Worker fatalities | 50+/year (global, all mines) |
| CO2 emitted | 500+ tons/year per mine |
| Success rate | <1% of claims ever produce commercial gold |

**A miner with a claim has two choices:**
1. Spend $10M+ to develop a mine (90% chance of failure)
2. Sell the claim for pennies on the dollar to a major miner

**This protocol offers a third choice:**
3. Prove the gold exists, transfer the rights, get paid — no mining required.

---

## 1. How It Works

### Phase 1: Claim Ownership Verification

```
Claim Owner presents:
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. Legal title deed (government-registered mining claim)    │
  │ 2. Chain of ownership (unbroken from original staking)      │
  │ 3. Proof of payments (claim maintenance fees paid)          │
  │ 4. GPS boundaries of the claim (surveyed, georeferenced)    │
  └─────────────────────────────────────────────────────────────┘
           │
           ▼
  Dox_Dev-verified lawyer reviews the title
  Dox_Dev-verified surveyor confirms the boundaries
  Dox_Dev-verified notary attests to the transferability
           │
           ▼
  WayChain Oracle witnesses → "Claim ownership verified"
  Claim owner receives a Dox_Dev badge level (Prospector badge)
```

**Required trust level:** High. Multiple Dox_Dev verifications across
different professions (lawyer, surveyor, notary). Each carries badge
revocation risk if they attest fraudulently.

### Phase 2: Reserve Verification

```
Claim Owner commissions a geological survey:
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. Independent assay lab tests core samples                 │
  │ 2. Report includes: ounces per ton, total estimated ounces, │
  │    depth of deposit, purity (fineness), recovery rate       │
  │ 3. NI 43-101 or JORC compliant report (industry standard)   │
  └─────────────────────────────────────────────────────────────┘
           │
           ▼
  Dox_Dev-verified geologist reviews and attests to the report
  Dox_Dev-verified assay lab confirms sample chain of custody
  Second lab does a blind re-test of split samples (optional)
           │
           ▼
  WayChain Oracle witnesses → "Reserves verified"
  Estimated recoverable ounces are recorded on-chain
```

**Verification standard:**

| Reserve Class | Confidence | Attesters Required | Tokenization Rate |
|--------------|-----------|-------------------|-------------------|
| **Measured** | >90% | 3 (geologist + 2 labs) | 80% of spot |
| **Indicated** | >70% | 3 (geologist + assay lab) | 60% of spot |
| **Inferred** | >50% | 2 (geologist) | 40% of spot |

A claim with 100,000 oz of Measured gold at $2,000/oz:
- Gross value: $200M
- Tokenization rate: 80%
- Protocol valuation: $160M
- Token supply: 80,000 tokens (each = 1 oz equivalent at 80%)

### Phase 3: Mineral Rights Transfer

```
Claim Owner signs a Mineral Rights Transfer Agreement:
  ┌─────────────────────────────────────────────────────────────┐
  │ The claim owner transfers ALL mineral rights to             │
  │ WayChain Partner LLC (or similar holding entity).            │
  │                                                              │
  │ Terms:                                                       │
  │ - Perpetual transfer (or 99-year lease, jurisdiction-       │
  │   dependent)                                                 │
  │ - NO mining allowed ever (covenant runs with the land)       │
  │ - Partner holds the rights in perpetuity                     │
  │ - Claim owner receives tokenized equivalent                  │
  │ - Partner is responsible for claim maintenance fees          │
  │ - Environmental monitoring required annually                 │
  └─────────────────────────────────────────────────────────────┘
           │
           ▼
  Dox_Dev-verified lawyer reviews transfer
  Dox_Dev-verified notary executes transfer
  Government registry updated (county recorder's office)
  Transfer deed hashed and attested on WayChain
  Mineral Rights Token (MRT) is issued
```

### Phase 4: Token Issuance

```solidity
contract MineralRightsToken {
    struct Claim {
        bytes32 deedHash;      // Legal deed on-chain hash
        uint256 totalOunces;   // Verified recoverable ounces
        uint256 verifiedDate;  // When reserves were verified
        uint256 tokenSupply;   // Tokens minted for this claim
        address[] verifiers;   // Dox_Dev addresses who attested
        bool active;           // Claim is active and monitored
    }
    
    mapping(bytes32 => Claim) public claims;
    mapping(address => uint256) public balanceOf;
    
    // Token = claim share. 1 token = 1/100,000 of a specific claim.
    // Tokens trade on WayChain. Backed by real, verified, in-ground reserves.
}
```

**Token economics:**

| Component | Value |
|-----------|-------|
| **Token** | Mineral Rights Token (symbol varies by claim, e.g. MRT-SIERRA-001) |
| **Backing** | Verified in-ground gold reserves, measured in troy ounces |
| **Claim owner receives** | Tokenized value minus 2% originator fee |
| **Partner receives** | Perpetual mineral rights + 2% fee |
| **Protocol treasury** | 1% of issuance |
| **Oracle attesters** | 0.5% of issuance |
| **Verification costs** | ~$25K-$100K per claim (assay, legal, survey) — paid by claim owner or deducted |

**Example:**
- 100,000 oz claim, $160M tokenized value at 80% rate
- Claim owner gets: $160M × 96.5% = $154.4M in tokens
- Originator fee: 2% = $3.2M
- Partner: 2% = $3.2M (for holding rights + liability)
- Treasury: 1% = $1.6M
- Attesters: 0.5% = $0.8M

The claim owner gets paid today for gold they'd spend years and millions
trying to extract — with no guarantee of success.

### Phase 5: Environmental Monitoring

Once the rights are transferred, the claim must be monitored annually:

```
Each year:
1. Satellite imagery of the claim (publicly verifiable)
2. Water table sampling (groundwater quality report)
3. No excavation or disturbance detected
4. Dox_Dev-verified environmental inspector attests
5. WayChain oracle witnesses → "Claim preserved"
6. Token holders receive environmental dividend (if any)

If mining activity is detected on the claim:
1. Immediate freeze on token transfers
2. Investigation by Dox_Dev-verified inspectors
3. If violation confirmed: partner enforces the covenant
   (injunction, damages, restoration bond)
4. Token holders' asset is protected by the legal covenant
```

---

## 2. Why This Only Works on WayChain

| Requirement | Every Other L1 | WayChain |
|-------------|---------------|----------|
| Identity verification | None | Dox_Dev badge (verified humans) |
| Professional attestation | Anonymous oracles | Lawyers, geologists, surveyors with badges |
| Consequence for false attestation | Bond loss (re-enter) | Badge revocation + bond (permanent) |
| Multi-profession verification | Complex, no standard | Template: lawyer + geologist + surveyor oracles |
| Real-world legal integration | None | Transfer deeds hashed + oracle attested |
| Long-term monitoring | No incentive for oracles | Attesters earn from monitoring fees |

**The key insight:** This doesn't work with anonymous oracles because
the verifications are not just economic — they're legal. A lawyer who
falsely attests to a title deed can be disbarred in real life AND lose
their Dox_Dev badge. The identity layer bridges the on-chain and off-chain
consequences.

---

## 3. Comparison: Tokenized Gold Solutions

| Solution | Backing | Mining Required? | Trust Model | Liquidity |
|----------|---------|-----------------|-------------|-----------|
| **PAXG** | Physical gold in vault | Yes (gold was mined) | Custodian (Paxos) | High |
| **XAUT** | Physical gold in vault | Yes (gold was mined) | Custodian (Tether) | High |
| **GLD ETF** | Physical gold | Yes (gold was mined) | Custodian (State Street) | Very high |
| **Mineral Rights Token** | In-ground reserves | **No. Never.** | Dox_Dev verified oracles + legal covenant | WayChain |

**The difference:** Every existing gold token represents gold that was
already mined — meaning the environmental damage was already done.
MRTs represent gold that will never be mined. The environmental benefit
is the product, not a side effect.

---

## 4. Token Holder Rights

| Right | Detail |
|-------|--------|
| **Claim on reserves** | Each token represents a proportional share of the verified in-ground gold |
| **Environmental assurance** | Annual monitoring confirms no mining occurred |
| **Transferable** | Tokens trade freely on WayChain |
| **Buyback** | Partner commits to buy back at 70% of spot if reserves verification is refreshed every 5 years |
| **Liquidation** | If partner dissolves, mineral rights revert to token holders pro-rata |
| **Redemption** | Token holders have no right to extract the gold. The gold stays in the ground. |

**The economic promise:**
- Gold is historically a store of value ($2,000/oz+)
- In-ground gold with verified reserves has real value
- No carrying cost (no vault, no insurance, no transport)
- No environmental liability
- The partner holds the rights in perpetuity — the asset exists as long as the chain does

---

## 5. Roadmap

### Phase 1: First Claim (Months 1-6)

- [ ] Establish WayChain Partner LLC (US-based holding entity)
- [ ] Legal framework: mineral rights purchase agreement, environmental covenant
- [ ] First claim: identify a fully-permitted but undeveloped gold claim
- [ ] Verify ownership (Dox_Dev lawyer + surveyor)
- [ ] Verify reserves (NI 43-101 compliant report, oracle attested)
- [ ] Transfer rights to partner
- [ ] Issue first Mineral Rights Token
- [ ] List on WayChain DEX

### Phase 2: Scale (Months 7-12)

- [ ] Standardize verification templates (geologist, lab, surveyor, lawyer)
- [ ] 10 more claims in diverse jurisdictions (Nevada, Alaska, Australia, Canada)
- [ ] MRT index token (basket of all claims)
- [ ] Partnership with environmental NGOs (verify "no mining" status)
- [ ] Carbon credit integration (carbon sequestered by not mining)

### Phase 3: Ecosystem (Year 2+)

- [ ] Expand beyond gold: silver, copper, lithium, rare earths
- [ ] "Mining Rights as a Service" — any claim owner can tokenize
- [ ] DAO governance: token holders vote on new claim acquisitions
- [ ] Cross-chain: bridge MRTs to Ethereum/Bitcoin for liquidity

---

## 6. Summary

| Metric | Value |
|--------|-------|
| Token | Mineral Rights Token (MRT) |
| Backing | Verified in-ground gold reserves |
| Mining required? | **No. The gold stays in the ground. Forever.** |
| Claim owner gets | Tokenized value (96.5% net after fees) |
| Partner gets | Perpetual mineral rights + 2% origination fee |
| Verification | Dox_Dev lawyers + geologists + assay labs + oracles |
| Environmental monitoring | Annual satellite + water + ground inspection |
| Redemption | Partner buyback at 70% of spot (optional, 5-year refresh) |

**"The most valuable gold is the gold that was never dug up."**

WayChain can prove existence, provenance, and preservation. No other
chain has the identity layer to make this work.