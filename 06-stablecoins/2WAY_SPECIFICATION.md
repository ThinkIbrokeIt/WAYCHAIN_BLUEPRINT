# 2WAY — WayChain Multi-Collateral Stablecoin

**Version:** 1.0
**Status:** Design Specification
**Chain:** WayChain (Chain ID 10008)
**Precompile Address:** 0x18 (2WAY Vault)

---

## 1. Overview

2WAY is WayChain's second stablecoin — a multi-collateral, crypto-backed
synthetic asset that accepts cross-chain assets (BTC, ETH, SOL, MATIC, etc.)
as collateral. It opens pathways for users from other ecosystems to transact
on WayChain without selling their native assets.

**Key Properties:**
- Peg: 1 2WAY = 1 USD (soft-pegged, maintained by arbitrage)
- Collateral: Multi-asset (BTC, ETH, SOL, MATIC, stETH, and more)
- Over-collateralized: Minimum 150% C-Ratio, 175% target
- Liquidation: Stability Pool + auction fallback (soft liquidation)
- Revenue: Stability fees + liquidation penalties → protocol treasury + BIJO stakers
- Cross-chain: 2WAY is an omnichain token (LayerZero OFT or native bridge)

---

## 2. Architecture

### 2.1 Collateral Vault Model (CDP)

Each user opens a Vault and deposits collateral. They can mint 2WAY up
to their collateral ratio limit.

```
User → deposit BTC/ETH/SOL → Vault → mint 2WAY → use 2WAY on WayChain
```

**Vault State:**
```
Vault {
  owner: address
  collaterals: { asset: amount }    // Multi-asset
  debt: uint256                      // 2WAY minted
  updatedBlock: uint64
}
```

### 2.2 Collateral Types

| Asset | Min C-Ratio | Liquidation Ratio | Stability Fee | Max Debt |
|-------|-------------|-------------------|---------------|----------|
| BTC   | 150%        | 140%              | 1.5% APR      | 30% cap  |
| ETH   | 150%        | 140%              | 1.5% APR      | 30% cap  |
| stETH | 150%        | 140%              | 1.5% APR      | 20% cap  |
| SOL   | 175%        | 160%              | 2.5% APR      | 15% cap  |
| MATIC | 175%        | 160%              | 2.5% APR      | 10% cap  |
| WAY   | 200%        | 180%              | 3.0% APR      | 10% cap  |

**Rationale:** More volatile assets require higher collateral ratios. BTC/ETH
are deepest and most trusted → lowest ratios. WAY is the native token →
highest ratio to prevent reflexive depeg.

### 2.3 Collateral Ratio Calculation

```
C-Ratio = (Sum of Collateral Value in USD) / (Debt in 2WAY)
```

Example: Deposit $1,500 worth of ETH, mint 800 2WAY.
C-Ratio = 1500 / 800 = 187.5% (safe)

### 2.4 Global Collateral Cap

Each collateral type has a **debt ceiling** (max % of total system debt).
This prevents any single asset from dominating risk.

```
Total Debt = Sum of all 2WAY minted across all vaults
BTC Debt Ceiling = 30% of Total Debt
ETH Debt Ceiling = 30% of Total Debt
All Others = 10-20% each
```

---

## 3. Peg Stability Mechanism

### 3.1 Internal Redemption (Primary)

1 2WAY can always be redeemed for $1 worth of collateral at any time.
This creates a hard floor: if 2WAY trades below $1, arbitrageurs buy
and redeem for profit.

```
2WAY price < $1.00 → Arbitrageurs buy 2WAY → redeem for $1 collateral → price rises
2WAY price > $1.00 → Arbitrageurs deposit collateral → mint 2WAY → sell → price falls
```

### 3.2 Stability Pool (Secondary)

A pool of 2WAY + USDC (or 1WAY) that absorbs small depegs:
- Depositors earn yield from liquidation penalties
- Pool automatically buys 2WAY when price < $0.99
- Acts as first line of defense before liquidations trigger

### 3.3 Stability Fee (Tertiary)

Variable interest rate on outstanding debt. When 2WAY is below peg:
- Governance raises stability fee → borrowing becomes expensive →
  less minting → debt shrinks → price recovers

When 2WAY is above peg:
- Governance lowers stability fee → borrowing becomes cheaper →
  more minting → supply expands → price falls back to peg

---

## 4. Liquidation Mechanism

### 4.1 Liquidation Trigger

A vault becomes liquidatable when:
```
C-Ratio < Liquidation Ratio (e.g., 140% for BTC)
```

### 4.2 Liquidation Flow

```
1. Vault falls below Liquidation Ratio
2. Stability Pool absorbs the debt first (if sufficient 2WAY available)
3. If Stability Pool insufficient → Auction begins
4. Collateral auction: Liquidators bid 2WAY to buy discounted collateral
5. Remaining collateral returned to vault owner
6. Liquidation penalty (10%) goes to protocol treasury
```

### 4.3 Soft Liquidation (Phase 2)

For assets with deep AMM liquidity (ETH, BTC), implement LLAMMA-style
soft liquidation:
- As C-Ratio drops, collateral is gradually swapped to 2WAY
- Avoids sudden, large sell-offs that crash collateral prices
- Uses WayChain's built-in DEX liquidity (WayChainFactory/Pair contracts)

### 4.4 Stability Pool

```
Stability Pool {
  deposits: { user: 2WAY_amount }
  totalDeposits: uint256
  liquidationRevenue: uint256  // Accumulated from liquidations
}
```

- Depositors earn pro-rata share of liquidation penalties (target 5-15% APY)
- Depositors can withdraw anytime (FIFO during active liquidations)
- Acts as automatic buyer of last resort for underwater vaults

---

## 5. Oracle Design

### 5.1 Price Feeds

2WAY uses WayChain's existing oracle precompiles (0x0C-0x12):

| Precompile | Function | Assets Covered |
|------------|----------|----------------|
| 0x0C       | OracleAggregator | BTC/USD, ETH/USD |
| 0x0D       | OracleScheduler | SOL/USD, MATIC/USD |
| 0x0E       | OracleVerifier | stETH/ETH ratio |
| 0x0F       | TLSVerifier | Cross-chain verification |
| 0x10       | BLSVerify | Oracle signature aggregation |
| 0x11       | AccountRecovery | Emergency price freeze |
| 0x12       | StateRent | On-chain data indexing |

### 5.2 Price Validation Rules

- Minimum 3 oracle confirmations required
- Max 2% deviation from last price (circuit breaker)
- Staleness check: price must be < 1 hour old
- Emergency freeze: If >50% of oracles report >10% deviation, halt new mints

### 5.3 Cross-Chain Price Verification

For non-native assets (ETH, SOL, MATIC):
- Oracle network attests to source chain state
- Price = median of 7 oracle responses
- Outlier rejection: discard responses >5% from median

---

## 6. Tokenomics

### 6.1 2WAY Supply Model

```
No hard cap — supply expands/contracts based on demand and collateral
Target initial supply: 10M 2WAY (from liquidity bootstrap)
Max supply growth: 20% per year (governance-controlled)
```

### 6.2 Revenue Model

| Source | Rate | Allocation |
|--------|------|------------|
| Stability Fee | 1.5-3% APR | 80% Treasury, 20% BIJO stakers |
| Liquidation Penalty | 10% of liquidated debt | 70% Stability Pool, 30% Treasury |
| Redemption Fee | 0.5% | 100% Treasury |
| Protocol-Owned Liquidity Yield | Variable | 100% Treasury |

### 6.3 Protocol Treasury

```
Treasury {
  assets: { asset: amount }    // Accumulated fees
  insuranceFund: uint256       // Backstop for bad debt
  maxInsuranceRatio: 10%      // 10% of total debt
}
```

### 6.4 Insurance Fund

If liquidations don't cover all debt (black swan event):
- Insurance fund covers shortfall
- If insurance insufficient → protocol issues recovery tokens (dilute BIJO)
- Recovery tokens vest over 1 year to prevent dump

---

## 7. Cross-Chain 2WAY

### 7.1 Omnichain Design

2WAY is an omnichain fungible token using LayerZero OFT standard:

```
Ethereum ←→ WayChain ←→ Solana ←→ Polygon
     ↕                        ↕
  Arbitrum                 Avalanche
```

### 7.2 Bridge Mechanism

- **Lock-and-Mint:** 2WAY locked on source chain, minted on destination
- **Native burn:** 2WAY burned on destination, released on source
- **Gas:** <$0.10 per cross-chain transfer (WayChain advantage)
- **Finality:** <5 minutes (WayChain 1s finality + source chain confirmation)

### 7.3 Cross-Chain Collateral

Users can deposit collateral on any supported chain:
- Deposit BTC on Ethereum → oracle verifies → mint 2WAY on WayChain
- Deposit SOL on Solana → oracle verifies → mint 2WAY on WayChain
- Cross-chain deposit takes 2-5 minutes (source chain finality + oracle)

---

## 8. Governance Parameters

### 8.1 Adjustable Parameters

| Parameter | Initial | Range | Governance |
|-----------|---------|-------|------------|
| Min C-Ratio (BTC/ETH) | 150% | 130-200% | DAO vote |
| Min C-Ratio (SOL/MATIC) | 175% | 150-250% | DAO vote |
| Min C-Ratio (WAY) | 200% | 150-300% | DAO vote |
| Stability Fee | 1.5-3% | 0-10% | DAO vote |
| Liquidation Penalty | 10% | 5-20% | DAO vote |
| Debt Ceiling per asset | 10-30% | 0-50% | DAO vote |
| Stability Pool Reward | 5-15% APY | 3-25% | DAO vote |
| Emergency Pause | Active | On/Off | Multi-sig (3/5) |

### 8.2 Emergency Controls

- **Circuit Breaker:** Pause all mints if 2WAY depegs >5% for >1 hour
- **Oracle Freeze:** Halt if >3 oracles report stale/corrupt data
- **Global Settlement:** In extreme scenarios, redeem all 2WAY at current collateral value
- **Admin Key:** 3-of-5 multi-sig (curators + dev team) for emergency pause

---

## 9. Security Model

### 9.1 Attack Vectors & Mitigations

| Attack | Mitigation |
|--------|------------|
| Oracle manipulation | 7-oracle median, 2% deviation cap, staleness check |
| Flash loan price manipulation | TWAP (time-weighted average price) for collateral valuation |
| Collateral depeg death spiral | Global collateral caps, insurance fund, emergency pause |
| Governance attack | 90-day timelock, 2/3 supermajority for critical params |
| Smart contract bug | Formal verification, audit, bug bounty |
| Cross-chain bridge exploit | Rate limiting, daily caps, multi-sig confirmation |

### 9.2 Collateral Risk Tiers

| Tier | Assets | Max Debt Ceiling | Oracle Requirement |
|------|--------|------------------|-------------------|
| 1 (Safest) | BTC, ETH | 30% each | 7/7 oracles |
| 2 (Safe) | stETH, WBTC | 20% each | 5/7 oracles |
| 3 (Moderate) | SOL, MATIC | 15% each | 5/7 oracles |
| 4 (Risky) | WAY, alt-L1s | 10% each | 7/7 oracles + TWAP |

---

## 10. Initial Deployment Plan

### Phase 1: Single-Collateral (Week 1-2)
- 2WAY vaults accept BTC only (via BitcoinRegistry precompile)
- Stability Pool with 2WAY/USDC
- 7 oracle price feeds for BTC/USD
- Initial supply cap: 1M 2WAY

### Phase 2: Multi-Collateral (Week 3-4)
- Add ETH, stETH collateral
- Add SOL, MATIC via cross-chain oracle attestation
- Enable cross-chain 2WAY (Ethereum ↔ WayChain)
- Supply cap: 10M 2WAY

### Phase 3: Scale (Month 2+)
- Add remaining collateral types
- Enable soft liquidation (AMM-based)
- Cross-chain 2WAY to Solana, Polygon, Avalanche
- Remove supply cap (governance-controlled growth)

---

## 11. Precompile Integration

### 11.1 2WAY Vault Precompile (0x18)

Extends the existing 12 precompiles with a 13th:

```
Precompile 0x18 — TwoWayVault
  - deposit(asset, amount, vaultID)
  - mint(vaultID, amount)
  - withdraw(asset, amount, vaultID)
  - repay(vaultID, amount)
  - liquidate(vaultID)
  - getVault(vaultID) → (collaterals, debt, ratio)
  - getPrice(asset) → (price, timestamp, confidence)
```

### 11.2 Integration with Existing Precompiles

| Precompile | 2WAY Usage |
|------------|-----------|
| 0x0C-0x12 (Oracles) | Price feeds for all collateral assets |
| 0x13 (DoxDevBadge) | Curator role for emergency controls |
| 0x14 (BIJO) | Revenue distribution to BIJO stakers |
| 0x15 (DeadMansSwitch) | Inheritance for vault positions |
| 0x16 (BitcoinRegistry) | BTC collateral verification |
| 0x17 (StorageEndowment) | Protocol-owned liquidity yield |

---

## 12. Economic Projections

### 12.1 Scenario: $10M TVL (Year 1)

```
Collateral deposited: $10,000,000
Average C-Ratio: 180%
2WAY minted: $5,555,555
Stability fee (2% APR): $111,111/year revenue
Liquidation penalties (est. 2% of TVL/yr): $200,000/year
Total protocol revenue: ~$311,000/year
```

### 12.2 Scenario: $100M TVL (Year 3)

```
Collateral deposited: $100,000,000
Average C-Ratio: 175%
2WAY minted: $57,142,857
Stability fee (2% APR): $1,142,857/year
Liquidation penalties: $2,000,000/year
Cross-chain fees: $500,000/year
Total protocol revenue: ~$3,642,857/year
```

### 12.3 2WAY Demand Drivers

1. **Arbitrage:** When 1WAY (native stablecoin) and 2WAY both trade at $1,
   arbitrageurs use 2WAY as collateral to mint 1WAY (leveraged yield)
2. **Cross-chain onboarding:** Users bridge BTC/ETH to WayChain → deposit →
   mint 2WAY → participate in WayChain DeFi
3. **Leveraged long:** Deposit ETH → mint 2WAY → buy more ETH → repeat
4. **Working capital:** Hold BTC, borrow 2WAY against it, spend 2WAY

---

## 13. Risks & Honest Assessment

### 13.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Oracle failure | Low | Critical | 7-oracle median, circuit breaker |
| Collateral 10+% drop in 1 hour | Medium | High | Liquidation + insurance fund |
| Smart contract bug | Low | Critical | 2 audits + formal verification |
| Cross-chain bridge exploit | Low | High | Rate limits, daily caps |

### 13.2 Economic Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| 2WAY depeg below $0.95 | Medium | High | Stability fee increase, Stability Pool |
| Collateral death spiral (ETH 50% drop) | Low-Med | Critical | Global collateral caps, emergency pause |
| Insufficient liquidity for redemptions | Medium | Medium | Protocol-owned liquidity, POL strategy |
| Regulatory (security classification) | Unknown | Unknown | Decentralized governance, no issuer |

### 13.3 What Could Go Wrong

If BTC/ETH drops 50%+ in a single day:
1. Mass liquidations trigger → collateral dumped → further price drops
2. Stability Pool absorbs first losses (target: 5-15% of TVL)
3. Insurance fund covers next layer
4. If both exhausted → recovery tokens issued (dilutive but survivable)
5. 2WAY may trade at $0.80-0.90 temporarily until collateral recovers

**This is the same risk profile as MakerDAO, Liquity, and Synthetix.
WayChain's advantage: 1s finality means faster liquidation processing.**

---

## 14. Implementation Checklist

- [ ] 2WAY Vault precompile (0x18) — deposit/mint/withdraw/liquidate
- [ ] Stability Pool contract — deposit/withdraw/liquidation absorption
- [ ] Oracle integration — 7 feeds for BTC/ETH/stETH/SOL/MATIC/WAY
- [ ] Liquidation engine — Stability Pool first, auction fallback
- [ ] Cross-chain bridge — LayerZero OFT for 2WAY omnichain
- [ ] Governance module — parameter adjustment, emergency controls
- [ ] Insurance fund — automatic allocation from protocol revenue
- [ ] UI — vault management, 2WAY mint/burn, cross-chain interface
- [ ] Audit — 2 independent firms before mainnet
- [ ] Bug bounty — $50K+ program

---

## 15. Summary

2WAY transforms WayChain from a single-collateral chain into a multi-asset
settlement layer. By accepting BTC, ETH, SOL, MATIC and others as collateral:

1. **Users from other chains** can bring their assets to WayChain
2. **2WAY becomes the unit of account** for multi-chain DeFi on WayChain
3. **Protocol revenue** flows to BIJO stakers and the treasury
4. **Network effects** compound as more assets → more users → more liquidity

The math works at every scale. $10M TVL generates $300K/year revenue.
$100M TVL generates $3.6M/year. $1B TVL generates $36M/year — enough
to fund protocol development indefinitely.

**2WAY is the gateway drug for cross-chain liquidity on WayChain.**
