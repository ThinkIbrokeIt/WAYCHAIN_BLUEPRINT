# WayChain Supply & Roadmap — v1.0

**Genesis supply:** 100,000,000 WAY
**Distribution:** Equal per Dox_Dev-verified human
**No pre-mine, no pre-sale, no VC allocation.**

---

## 1. Supply Schedule

### 1.1 Inflation Curve

Inflation starts at 7% and declines by 0.5% per year to a 3% floor.
Governance can adjust within a 1% band of the scheduled rate with 2/3
supermajority and 90-day timelock.

| Year | Inflation Rate | New Issuance | Supply (EoY, no burns) | Per Validator (85% pool / 200) |
|------|---------------|-------------|----------------------|------------------------------|
| 0    | —             | —           | 100,000,000          | —                            |
| 1    | 7.0%          | 7,000,000   | 107,000,000          | 29,750                       |
| 2    | 6.5%          | 6,955,000   | 113,955,000          | 29,559                       |
| 3    | 6.0%          | 6,837,300   | 120,792,300          | 29,059                       |
| 4    | 5.5%          | 6,643,577   | 127,435,877          | 28,235                       |
| 5    | 5.0%          | 6,371,794   | 133,807,671          | 27,080                       |
| 6    | 4.5%          | 6,021,345   | 139,829,016          | 25,591                       |
| 7    | 4.0%          | 5,593,161   | 145,422,177          | 23,771                       |
| 8    | 3.5%          | 5,089,776   | 150,511,953          | 21,632                       |
| 9    | 3.0%          | 4,515,359   | 155,027,312          | 19,190                       |
| 10   | 3.0%          | 4,650,819   | 159,678,131          | 19,766                       |
| 10+  | 3.0% flat     | ~4.8M/yr    | ~164M @ yr 15        | ~20,400                      |

**Validator income in dollar terms (assuming token appreciation):**

| Year | WAY/Validator | Value at $0.10 | Value at $0.50 | Value at $1.00 |
|------|-------------|---------------|---------------|---------------|
| 1    | 29,750      | $2,975        | $14,875       | $29,750       |
| 3    | 29,059      | $2,906        | $14,530       | $29,059       |
| 5    | 27,080      | $2,708        | $13,540       | $27,080       |
| 8    | 21,632      | $2,163        | $10,816       | $21,632       |
| 10   | 19,766      | $1,977        | $9,883        | $19,766       |

**Critical insight:** Validator income declines in raw WAY over time (as
inflation drops), but dollar income holds if the token appreciates with
adoption. A successful chain at $1.00/WAY in year 10 pays validators
~$20K/year — professional-level income. An unsuccessful chain at $0.01
pays $200/year. The economics are self-reinforcing: adoption → value →
validator retention → more adoption.

### 1.2 Genesis Distribution

**100M WAY at genesis.** Every Dox_Dev-verified human receives an
equal share. The per-human amount depends on the genesis cohort size:

| Genesis Cohort | Per Human | Usable For |
|---------------|-----------|-----------|
| 500 humans    | 200,000 WAY | Staking 1,000 → 15%. Deploy templates. |
| 1,000 humans  | 100,000 WAY | Staking 1,000 at 15%. 99K for use. |
| 2,000 humans  | 50,000 WAY  | Staking 5,000 (mixed brackets). 45K for use. |
| 5,000 humans  | 20,000 WAY  | Staking 1K at 15%. Enough to participate. |
| 10,000 humans | 10,000 WAY  | Staking 1K at 15%. 9K left for economy. |

**Target: 1,000-2,000 verified humans at genesis.** This gives each
human 50K-100K WAY, enough to stake meaningfully, deploy templates,
and participate in governance without a single human holding too much
relative power.

### 1.3 Issuance Distribution

| Allocation | % of Issuance | Annual (yr 1) | Purpose |
|------------|--------------|--------------|---------|
| Validator rewards | 85% | 5,950,000 WAY | Block rewards for 200 validators |
| Protocol treasury | 10% | 700,000 WAY | Dev, audits, subsidies, grants |
| Bootstrap subsidy | 5% | 350,000 WAY | Year 1 VPS costs |

**No dormant wallets.** All issuance goes to active participants.

---

## 2. What Drives Price

### 2.1 Staking Demand (Primary Driver)

The progressive curve creates **natural demand pressure.** Small stakers
earn 15% on their first 1,000 WAY. This incentivizes people to acquire
and stake WAY, not trade it.

| If this many people stake 1,000 WAY each | Total WAY locked | % of Genesis Supply |
|------------------------------------------|-----------------|-------------------|
| 1,000 | 1,000,000 | 1% |
| 10,000 | 10,000,000 | 10% |
| 50,000 | 50,000,000 | 50% |
| 200,000 | 200,000,000 | 200% |

At 200K small stakers, more WAY is locked in staking than exists in
circulation. This creates a supply squeeze that drives price
appreciation — **organically, through utility, not speculation.**

### 2.2 Service Demand (Secondary)

Every on-chain action consumes WAY as gas, fees, or bonds:

| Activity | WAY Cost (@ $0.10) | WAY Cost (@ $0.50) |
|----------|-------------------|-------------------|
| Simple tx (gas) | 0.01 WAY | 0.002 WAY |
| Template deploy | 100 WAY | 20 WAY |
| Dox_Dev L3 badge | 500 WAY | 100 WAY |
| Validator registration | 50 WAY | 10 WAY |

At 50K tx/day + 10 developers/day, daily service demand:
- 50,000 × 0.01 + 10 × 100 = 600 + 1,000 = 1,600 WAY/day (@ $0.10)
- 584,000 WAY/year from service demand alone

### 2.3 Burn Pressure (Deflationary)

| Source | Year 1 | Year 5 | Year 10 |
|--------|--------|--------|---------|
| State rent (60% burn) | ~$2,100/yr | ~$12,600/yr | ~$25,000/yr |
| Registration (100% burn) | ~$1,000/yr | ~$1,500/yr | ~$2,500/yr |
| Slashing (50% burn) | ~$10,000/yr | ~$20,000/yr | ~$30,000/yr |
| **Total USD burned** | **~$13,000/yr** | **~$34,000/yr** | **~$58,000/yr** |

Burns grow with adoption. In year 1, burns are tiny vs. 7M new issuance.
By year 10, burns are meaningful vs. 4.7M issuance. The deflationary
pressure increases over time.

### 2.4 Speculation (Wildcard)

"Actually decentralized" is a brand no chain currently owns. If WayChain
executes — one badge = one validator, one badge = one vote, no whale
capture — people will want WAY because they believe in the model, not
because of promises.

This is the most volatile price driver. It's also the most powerful.

---

## 3. Roadmap — Followed to the T

### Phase 0: Foundation (Months 1-6)

**Goal:** Chain exists. First 200 validators are running.

| Milestone | Target | Verification |
|-----------|--------|-------------|
| Consensus client (CometBFT fork) | Month 3 | 3-node testnet running |
| EVM execution layer | Month 4 | Solidity bytecode executes |
| Account model + Dox_Dev integration | Month 5 | Dox_Dev badge gates deploy |
| Genesis cohort recruitment | Month 6 | 200+ verified humans with hardware |
| **Mainnet genesis** | **Month 6** | **Chain launches with 200 validators** |

**Phase 0 supply:** 100M WAY genesis. Equal distribution to verified humans.
Treasury holds 10M WAY reserve. Inflation at 7%.

**Phase 0 validator econ:** Validators earn ~29,750 WAY/yr from issuance.
Treasury subsidy covers VPS ($50/mo) for all active validators.

---

### Phase 1: Safety & Stability (Months 7-12)

**Goal:** Chain is safe and usable. Memecoin deploy works.

| Milestone | Target | Verification |
|-----------|--------|-------------|
| Template registry live | Month 7 | First template deployed from registry |
| Trustless Lock templates | Month 7 | Atomic memecoin deploy flow working |
| Dox_Dev Level 2+ required for deploys | Month 8 | No anonymous contracts |
| Oracle attester set active | Month 9 | First data feed live |
| First memecoin deployed via template | Month 9 | Audited, locked, verified |
| **Phase 1 review** | **Month 12** | **Is the chain stable? Are validators staying?** |

**Phase 1 supply:** ~107M WAY. Inflation drops to 6.5%. Treasury has
received 700K WAY from year 1 issuance.

**Phase 1 validator econ:** Staking begins. Early stakers lock 1K WAY
at 15%. The 15% bracket starts attracting small stakers.

**Exit criteria:**
- Chain has run 6 months without critical failure
- 180+ validators still active (>90% retention)
- At least 10 verified templates in registry
- Oracle running with 25+ attesters

---

### Phase 2: Adoption (Year 2)

**Goal:** Real users. Real transactions. Validator income stabilizes.

| Milestone | Target | Verification |
|-----------|--------|-------------|
| UX onboarding ledger (Stage 0 accounts) | Month 13 | Normie onboarding flow |
| Session keys + gas abstraction | Month 14 | Pay gas in any token |
| Cross-chain bridge to PulseChain | Month 16 | WAY/PLS pair on PulseChain |
| Binary Journal truth anchoring | Month 18 | First anchored truth |
| Liquidity on native DEX | Month 18 | WAY/PLS pair with locked LP |
| **Phase 2 review** | **Month 24** | **Has adoption started?** |

**Phase 2 supply:** ~114M WAY. Inflation at 6%.

**Phase 2 validator econ:** VPS subsidy ends at month 18 (extended 6
months from the original 12). By month 18, token price should support
validator income without subsidy.

**Phase 2 price drivers:**
- Staking demand: target 10,000+ small stakers
- Service demand: target 100+ active developers
- Cross-chain bridge brings external liquidity

**Exit criteria:**
- 5,000+ active wallets
- 50+ templates deployed (real projects, not tests)
- Staking participation >20% of circulating supply
- Validator retention >85% without subsidy

---

### Phase 3: Independence (Year 3)

**Goal:** Chain is self-sustaining. Governance is active.

| Milestone | Target | Verification |
|-----------|--------|-------------|
| First governance vote | Month 25 | Badge holders vote on parameter |
| Treasury diversification | Month 26 | Treasury holds stable assets + WAY |
| Community grants program | Month 27 | First grant awarded by governance |
| Oracle at 50+ feeds | Month 30 | Oracles are a real data marketplace |
| **Phase 3 review** | **Month 36** | **Is the chain independent?** |

**Phase 3 supply:** ~121M WAY. Inflation at 5.5%.

**Phase 3 validator econ:** Validators earn from issuance + growing fee
volume. At $0.50+/WAY, validator income is $12K-$15K/yr — professional
level.

**Phase 3 price drivers:**
- Staking demand: target 50,000+ small stakers (50%+ supply locked)
- Service demand: target 500+ developers
- Burn pressure becomes meaningful (2-3% of issuance)
- Governance proves it works

**Exit criteria:**
- 25,000+ active wallets
- 200+ templates deployed
- State rent burn removes >1% of supply
- First governance vote passes with >60% participation

---

### Phase 4: Maturity (Years 4-5)

**Goal:** WayChain is a proven alternative. The model works.

| Milestone | Target | Verification |
|-----------|--------|-------------|
| Governance 2.0 (quadratic/liquid) | Year 4 | Enhanced voting model |
| Cross-chain attestation layer | Year 4 | WayChain oracles serve other chains |
| DeFi ecosystem (lending, DEX, stablecoin) | Year 4-5 | Native DeFi without rug risk |
| **Phase 4 review** | **Month 60** | **Did we build what we promised?** |

**Phase 4 supply:** ~128-134M WAY. Inflation at 5% declining to 4.5%.

**Phase 4 validator econ:** If the chain has succeeded, validators are
earning $15K-$25K/yr. The 200 validator seats are competitive. New
candidates must wait for a seat or replace a retiring validator.

**Phase 4 price drivers:**
- Staking demand: target 200,000+ small stakers (supply squeeze)
- DeFi activity drives fee volume
- WayChain brand as "actually decentralized L1" has real value

---

### Phase 5: Legacy (Year 6+)

**Goal:** The chain runs itself. Governance handles adjustments.

Inflation stabilizes at 3%. Supply grows at ~3% minus burns (~1-2%) =
~1-2% net growth. Validators earn from fee volume + modest issuance.

No more phases. The chain is what it is — a decentralized, human-verified
L1 that displaced the plutocratic model.

---

## 4. Risk Register — What Can Break Us

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **Can't recruit 200 validators** | Medium | Critical | Bootstrap with as few as 50. Expand over 6 months. |
| **Token never finds $0.10** | Medium | High | Subsidy covers year 1. If no value by year 2, chain pivots or folds. |
| **Badge farming (Sybil attack)** | High | Medium | Dox_Dev verification level must be strong. Adjust if needed. |
| **Governance captured by small group** | Low | Critical | 2/3 supermajority + 90-day timelock on all changes. |
| **Validator collusion** | Low | High | Badge revocation slashing (10%). Real identity is real accountability. |
| **Oracle failure (fee pegging breaks)** | Low | Medium | Fallback to fixed native fee. Graceful degradation. |
| **No developer adoption** | Medium | High | Templates make it easy. If no one builds, the chain has no use. |
| **Regulatory pressure** | Low | Medium | Code is speech. We build. We don't pre-solve for regulation. |

---

## 5. Non-Negotiable Commitments

These are locked at genesis. Governance cannot change them.

1. **One Dox_Dev badge = one validator.** Forever.
2. **One badge = one governance vote.** Token weight never touches governance.
3. **Progressive staking curve exists.** Brackets adjust. Curve cannot be removed.
4. **Fees are fiat-pegged.** Stay cheap in real terms.
5. **No pre-mine, no pre-sale.** Ever.
6. **Genesis distribution is equal per verified human.** No exceptions.
7. **Treasury is transparent.** All transactions on-chain.
8. **The roadmap above is the plan.** We follow it or we explicitly vote to change it.