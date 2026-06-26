# 1WAY — Trustless Bitcoin-Backed Stablecoin

## How It Works, How It Stays Stable, How It's Enforced

---

## 1. The Core Mechanism (Trustless)

1WAY is backed 1:1 by Bitcoin locked in an oracle multi-sig. No single party controls the BTC. Every key holder has a Dox_Dev badge — permanent identity, real-world consequences.

### 1.1 Mint (BTC → 1WAY)

```
User sends 1 BTC to 3-of-5 Dox_Dev oracle multi-sig address
  → BTC is locked. No one can move it without 3 of 5 oracle keys.
  → WayChain oracles witness the transaction
  → BitcoinSPV verifies (6+ confirmations)
  → 1WAY contract mints ~70,000 1WAY (at 70% ratio, 143% collateral)
  → User's WayChain wallet receives 1WAY
```

**One Bitcoin transaction. One mint. BTC is in the lockbox.**

### 1.2 Burn (1WAY → BTC)

```
User has 70,000 1WAY and wants their BTC back
  → User burns 70,000 1WAY in the 1WAY contract
  → Contract emits Burned event
  → Oracles witness the burn
  → 3 of 5 oracles sign a Bitcoin transaction
  → BTC released from multi-sig to user's Bitcoin address
  → 1WAY supply reduced. Peg intact.
```

**No permission. No delay. No one can stop the withdrawal.**

### 1.3 The Multi-Sig Trustlessness

The lockbox is a 3-of-5 multi-sig. Each key is held by a different Dox_Dev-verified human:

| Key Holder | Dox_Dev Level | What They Can Do Alone |
|------------|--------------|----------------------|
| Oracle A (US) | Level 3 | Nothing — needs 2 more keys |
| Oracle B (EU) | Level 3 | Nothing — needs 2 more keys |
| Oracle C (Asia) | Level 3 | Nothing — needs 2 more keys |
| Oracle D (Brazil) | Level 3 | Nothing — needs 2 more keys |
| Oracle E (Australia) | Level 3 | Nothing — needs 2 more keys |

**No single human can move the BTC. No two can. Three can — but all three would need to collude, each losing their Dox_Dev badge permanently.**

---

## 2. How the Peg Stays Stable

### 2.1 Overcollateralization

| BTC Price | 1 BTC Commits | 1WAY Minted | Collateral Ratio |
|-----------|---------------|-------------|-----------------|
| $68,000 | 1 BTC | ~47,600 1WAY | **143%** |
| $50,000 | 1 BTC | ~35,000 1WAY | **143%** |
| $30,000 | 1 BTC | ~21,000 1WAY | **143%** |

The ratio is always 143%. For every $100 of BTC committed, $70 of 1WAY is minted. The 30% buffer absorbs BTC price drops.

### 2.2 Liquidation (Automatic)

If BTC price drops such that the collateral ratio falls below 110%:

```
BTC price drops 30%+
  → Oracle detects: collateral at 108% — below 110% threshold
  → Anyone can liquidate: burn the 1WAY, claim the BTC at a discount
  → Liquidator burns 47,600 1WAY
  → 3 of 5 oracles sign to release 1 BTC to liquidator
  → Liquidator receives 1 BTC worth more than the 1WAY they burned
  → User receives nothing (they were undercollateralized)
  → Peg holds. 1WAY supply reduced. No one else is affected.
```

**The liquidation is not punitive — it's protective.** If a user doesn't maintain their 1WAY/BTC ratio, the system protects every other 1WAY holder.

### 2.3 Liquidation Warning

Before liquidation, the user gets warnings:

```
Warning 1: Collateral at 120% — "Add more BTC or burn 1WAY"
Warning 2: Collateral at 115% — "Liquidation in 24 hours"
Warning 3: Collateral at 112% — "Liquidation in 6 hours"
At 110%: Anyone can liquidate. No warning needed.
```

The 24-hour grace period at 115% gives the user time to act — either add more BTC to the lockbox or burn 1WAY to restore the ratio.

---

## 3. Enforcement (Beyond Code)

Code is speech. But real-world assets require real-world enforcement. 1WAY has two additional release mechanisms for edge cases that code cannot handle.

### 3.1 Court Order (Legal Enforcement)

If a court of competent jurisdiction issues a legitimate order:

```
Court order presented to the oracle council
  → Dox_Dev lawyer verifies the order is authentic
  → Order is published on-chain via Binary Journal anchor
  → 3 of 5 oracles sign to comply with the order
  → BTC released according to the court's instruction
```

**This is not a loophole. It's the rule of law.** Every oracle is a known human in a known jurisdiction. They can and will obey lawful court orders. This is no different from how banks, exchanges, and custodians operate. The difference is: the court order requires **3 independent oracles** to verify it first.

**What a court order cannot do:**
- Cannot order the release of BTC without a valid legal basis
- Cannot order oracles to violate their own jurisdiction's laws
- Cannot override a liquidation that already occurred

### 3.2 Local Vote (Community Enforcement)

If a clear case of injustice occurs (user lost keys, oracle collusion suspected, catastrophic error):

```
Any Dox_Dev Level 2+ holder submits a governance proposal
  → 7-day discussion period
  → 7-day vote period
  → Requires: 40% quorum, 2/3 supermajority
  → If passed: oracles execute the vote's instruction
  → BTC released according to the vote
```

**This is the emergency brake.** In the worst case — 5 oracles collude, a user's funds are stuck, an edge case the code didn't anticipate — the community can vote to intervene. It's a last resort, not a daily mechanism.

### 3.3 Full Accountability (Criminal)

Everyone involved in the 1WAY system is Dox_Dev-verified. Every action is on-chain. Every signature is recorded.

```
If oracles collude to steal BTC:
  → It's on-chain. Traceable. Permanent.
  → All 5 oracles are known humans with known identities.
  → They will be prosecuted to the fullest extent of the law.
  → Theft of 1,000+ BTC is a federal crime in every jurisdiction.
  → Dox_Dev isn't just a badge — it's a legal identity.

If a user fraudulently claims lost keys:
  → They are identified. Their badge is revoked.
  → Their identity is known. Prosecution follows.
```

**There is no anonymity in the 1WAY system.** Not for oracles. Not for users who commit meaningful BTC. Every action has a real person behind it. This is not surveillance — it's accountability. And it's what makes a trustless system actually work.

---

## 4. The Three Release Mechanisms

| Mechanism | Trigger | Speed | Trust Required |
|-----------|---------|-------|---------------|
| **Normal burn** | User burns 1WAY | Minutes (once tx confirms) | None — cryptographic |
| **Liquidation** | Collateral < 110% | Automatic | Oracles sign — identity-backed |
| **Court order** | Valid legal order | Days-weeks | Rules of law |
| **Local vote** | Community governance | 2 weeks | 2/3 supermajority of Dox_Dev holders |

**The first two are automatic and trustless. The last two are emergency valves for edge cases no code can predict.**

---

## 5. Summary

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. Send BTC → 3-of-5 oracle multi-sig (locked by code)     │
│  2. Receive 1WAY (70% ratio, 143% collateral)               │
│  3. Burn 1WAY → get BTC back (no permission)                │
│  4. Price drops → liquidator burns 1WAY → peg holds         │
│  5. Edge case → court order or community vote               │
│  6. Everyone is known → everyone is accountable             │
│                                                              │
│  Trust the math. Trust the identity. But the law backs it.  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

| Party | Can steal? | Can break peg? | Consequence |
|-------|-----------|----------------|-------------|
| User | No (BTC locked) | No | Badge revoked + legal prosecution |
| Single oracle | No (needs 3 keys) | No | — |
| 3+ oracles colluding | Theoretically yes | Theoretically yes | **Permanent badge revocation + federal prosecution** |
| Court | Yes (lawfully) | Yes (temporarily) | Legal authority |
| Community vote | Yes (emergency) | Yes (emergency) | Last resort only |