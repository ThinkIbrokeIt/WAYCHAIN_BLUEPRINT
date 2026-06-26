# WAYCHAIN BLUEPRINT — Unified Architecture & Design

**Version:** 2.0
**Last Updated:** 2026-06-24
**Total Spec Lines:** 9,974 (25 documents)

---

## Purpose

This document is the single entry point for ALL WayChain architecture.
Every spec, every design decision, every principle is referenced from here.
No developer should have to search for specs — they start here.

---

## Reading Order (Dependency Chain)

Read these in order. Each builds on the previous.

### Layer 0: Vision & Philosophy

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 1 | [NEW_CHAIN_VISION.md](./01-vision/NEW_CHAIN_VISION.md) | 655 | Use case audit, 10 design principles, gap analysis |
| 2 | [NEW_CHAIN_VISION_GAP_ANALYSIS.md](./01-vision/NEW_CHAIN_VISION_GAP_ANALYSIS.md) | 204 | Spec vs reality comparison |

### Layer 1: Protocol Core

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 3 | [NEW_CHAIN_CONSENSUS_SPEC.md](./02-protocol-core/NEW_CHAIN_CONSENSUS_SPEC.md) | 644 | BFT PoS, 1s blocks, sqrt-weighted lottery |
| 4 | [NEW_CHAIN_EVM_SPEC.md](./02-protocol-core/NEW_CHAIN_EVM_SPEC.md) | 666 | EVM opcodes, precompile architecture, gas model |
| 5 | [NEW_CHAIN_ACCOUNT_SPEC.md](./02-protocol-core/NEW_CHAIN_ACCOUNT_SPEC.md) | 698 | 3-stage accounts, guardian recovery, account abstraction |

### Layer 2: Safety & Identity

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 6 | [NEW_CHAIN_DOXDEV_SPEC.md](./03-safety-identity/NEW_CHAIN_DOXDEV_SPEC.md) | 434 | Dox_Dev badges, deploy gate, progressive hardening |
| 7 | [NEW_CHAIN_TEMPLATES_SPEC.md](./03-safety-identity/NEW_CHAIN_TEMPLATES_SPEC.md) | 650 | Template registry, Trustless Lock, atomic deploy |
| 8 | [NEW_CHAIN_TOKENOMICS.md](./03-safety-identity/NEW_CHAIN_TOKENOMICS.md) | 475 | Supply curve, fee flows, burning mechanisms, staking |

### Layer 3: Data & Truth

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 9 | [NEW_CHAIN_ORACLE_SPEC.md](./04-data-truth/NEW_CHAIN_ORACLE_SPEC.md) | 729 | Staked attesters, TLS proofs, VRF, time execution |
| 10 | [NEW_CHAIN_BINARY_JOURNAL_INTEGRATION.md](./04-data-truth/NEW_CHAIN_BINARY_JOURNAL_INTEGRATION.md) | 263 | Truth anchoring, inheritance, 3·6·9 protocol |
| 11 | [NEW_CHAIN_BITCOIN_INTEGRATION.md](./04-data-truth/NEW_CHAIN_BITCOIN_INTEGRATION.md) | 299 | Bitcoin SPV verification, native BTC, no wrapping |
| 12 | [NEW_CHAIN_CROSS_CHAIN_ATTESTATIONS.md](./04-data-truth/NEW_CHAIN_CROSS_CHAIN_ATTESTATIONS.md) | 288 | Cross-chain data bridging, graduated trust |

### Layer 4: User Experience

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 13 | [NEW_CHAIN_UX_SPEC.md](./05-user-experience/NEW_CHAIN_UX_SPEC.md) | 804 | 3-stage onboarding, session keys, gas abstraction |
| 14 | [NEW_CHAIN_USER_FLOW.md](./05-user-experience/NEW_CHAIN_USER_FLOW.md) | 308 | Alice's memecoin walkthrough, user journey |
| 15 | [NEW_CHAIN_SUPPLY_ROADMAP.md](./05-user-experience/NEW_CHAIN_SUPPLY_ROADMAP.md) | 311 | 100M WAY distribution, inflation schedule |

### Layer 5: Governance

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 16 | [NEW_CHAIN_GOVERNANCE_SPEC.md](./06-governance/NEW_CHAIN_GOVERNANCE_SPEC.md) | 321 | Quadratic voting, futarchy, 1 badge = 1 vote |

### Layer 6: Stablecoins

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 17 | [1WAY_STABLECOIN_SPEC.md](./06-stablecoins/1WAY_STABLECOIN_SPEC.md) | 192 | BTC-backed stablecoin, 3-of-5 oracle multi-sig |
| 18 | [2WAY_SPECIFICATION.md](./06-stablecoins/2WAY_SPECIFICATION.md) | 459 | Multi-collateral stablecoin, cross-chain assets |

### Layer 7: Special Topics

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 19 | [NEW_CHAIN_MINERAL_RIGHTS_TOKENIZATION.md](./07-special-topics/NEW_CHAIN_MINERAL_RIGHTS_TOKENIZATION.md) | 292 | Mineral rights on-chain, environmental preservation |

### Layer 8: Execution

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 20 | [NEW_CHAIN_BUILD_ORDER.md](./08-execution/NEW_CHAIN_BUILD_ORDER.md) | 239 | Phased implementation plan, dependency order |
| 21 | [LAUNCH_PLAN.md](./08-execution/LAUNCH_PLAN.md) | 222 | All 5 phases, verification checklist |
| 22 | [DEVELOPER_GUIDE.md](./08-execution/DEVELOPER_GUIDE.md) | 366 | Foundry/viem/ethers.js integration |

### Layer 9: Whitepaper

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 23 | [WHITEPAPER.md](./09-whitepaper/WHITEPAPER.md) | 455 | Complete public-facing paper |

### Layer 10: Binary Journal (Sanctuary + Inheritance)

| # | File | Lines | What It Defines |
|---|------|-------|----------------|
| 24 | [BUILDERS_MANUAL.md](./10-binary-journal/BUILDERS_MANUAL.md) | 1,188 | Complete build guide: sanctuary, ledger, verification, airdrop, liquidity, burn |
| 25 | [BUILDERS_SEQUENCE.md](./10-binary-journal/BUILDERS_SEQUENCE.md) | 204 | Step-by-step build order for all 6 phases |
| 26 | [GAP_ANALYSIS.md](./10-binary-journal/GAP_ANALYSIS.md) | 68 | Binary Journal gap analysis |
| 27 | [INHERITANCE_GAPS.md](./10-binary-journal/INHERITANCE_GAPS.md) | 175 | Inheritance protocol gap analysis |
| 28 | [PLAN.md](./10-binary-journal/PLAN.md) | 203 | Binary Journal full plan |

---

## Cross-Reference: Design Principles vs Implementation

| Principle | Spec | Status |
|-----------|------|--------|
| 0. Dox_Dev verified deployment | DOXSPEC | ✅ Live (precompile 0x13) |
| 1. Privacy as default | UX Spec (ZK section) | ❌ Spec'd only |
| 2. Fixed low fees | TOKENOMICS | ✅ Live |
| 3. Native oracle layer | ORACLE_SPEC | ✅ Live (7 precompiles) |
| 4. Identity as primitive | DOXSPEC + ACCOUNT_SPEC | ✅ Partial (badges live, recovery not) |
| 5. Role-based access control | DOXSPEC | ✅ Partial (deploy gate live, fine-grained not) |
| 6. Multi-dimensional storage pricing | TOKENOMICS | ❌ Spec'd only |
| 7. Non-plutocratic governance | GOVERNANCE_SPEC | ❌ Spec'd only |
| 8. Cross-chain native | CROSS_CHAIN_SPEC | ⚠️ Bitcoin only |
| 9. Sovereign recovery | ACCOUNT_SPEC | ⚠️ DeadMansSwitch only |
| 10. Anti-fraud consensus | GOVERNANCE_SPEC | ❌ Spec'd only |
| 11. State rent / expiry | TOKENOMICS | ❌ Spec'd only |
| 12. Verifiable off-chain compute | UX Spec | ❌ Spec'd only |

---

## Critical Dependencies (Read Before Building Anything)

1. **ACCOUNT_SPEC** defines how keys work — everything depends on this
2. **ORACLE_SPEC** defines how the chain sees reality — all data apps depend on this
3. **GOVERNANCE_SPEC** defines how changes happen — all parameter changes depend on this
4. **TOKENOMICS** defines the economic model — all financial apps depend on this

---

## File Integrity

All files are in `/home/wink/projects/WAYCHAIN_BLUEPRINT/` organized by layer.
Original locations are legacy — this directory is the single source of truth.

To verify nothing is missing:
```bash
find /home/wink/projects/WAYCHAIN_BLUEPRINT -name "*.md" | wc -l
```
Expected: 29 files (24 architecture + 5 binary journal)
