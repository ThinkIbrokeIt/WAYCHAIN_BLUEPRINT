# NEW_CHAIN_VISION — Gap Analysis: Spec vs Reality

**Date:** June 24, 2026
**Purpose:** Compare what's LIVE on WayChain vs what's been fully SPECCED but not yet implemented.

**Key insight:** Every item below has a complete specification document. The gap is
implementation, not design. We know exactly how to build each piece.

---

## Part 4: Design Principles — Spec Status

### Principle 0: Dox_Dev Verified Deployment

| Component | Spec | Status |
|-----------|------|--------|
| Soulbound badge system (3-level) | `NEW_CHAIN_DOXDEV_SPEC.md` | ✅ **DEPLOYED** — Precompile 0x13 |
| Deploy gate (3 layers) | `NEW_CHAIN_DOXDEV_SPEC.md` | ✅ **DEPLOYED** — RPC, block, EVM |
| Identity wallet ↔ project wallet linking | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |
| Encrypted identity + court-ordered disclosure | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |
| Progressive hardening | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |
| Badge reissue with wallet migration | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |
| Cross-chain deployer blacklist | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |

### Principle 1: Privacy as Default

| Component | Spec | Status |
|-----------|------|--------|
| ZK-selective disclosure precompile | Session `20260620` (delegated research) | ❌ Not implemented |
| Private-by-default execution | Session `20260620` | ❌ Not implemented |
| Healthcare/identity/corporate data privacy | Session `20260620` | ❌ Not implemented |

### Principle 2: Fixed Low Fees

| Component | Spec | Status |
|-----------|------|--------|
| Fixed fee model (not auctioned) | `NEW_CHAIN_TOKENOMICS.md` | ✅ **DEPLOYED** |
| State rent / state expiry | `NEW_CHAIN_TOKENOMICS.md` | ❌ Not implemented |

### Principle 3: Native Oracle Layer

| Component | Spec | Status |
|-----------|------|--------|
| Staked attestors with slashing | `NEW_CHAIN_ORACLE_SPEC.md` | ✅ **DEPLOYED** — 7 precompiles |
| Separate attester/validator sets | `NEW_CHAIN_ORACLE_SPEC.md` | ✅ **DEPLOYED** |
| Binary Journal truth anchoring | `NEW_CHAIN_BINARY_JOURNAL_INTEGRATION.md` | ✅ **DEPLOYED** — Precompile 0x14 |
| Verifiable randomness at protocol level | `NEW_CHAIN_ORACLE_SPEC.md` | ❌ Not implemented |
| Time-based execution at protocol level | `NEW_CHAIN_ORACLE_SPEC.md` | ❌ Not implemented |
| Cross-chain data bridging native | `NEW_CHAIN_CROSS_CHAIN_ATTESTATIONS.md` | ❌ Not implemented |
| Graduated trust model | `NEW_CHAIN_ORACLE_SPEC.md` | ❌ Not implemented |

### Principle 4: Identity as First-Class Primitive

| Component | Spec | Status |
|-----------|------|--------|
| Soulbound badges | `NEW_CHAIN_DOXDEV_SPEC.md` | ✅ **DEPLOYED** |
| Verifiable credentials | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |
| Reputation system | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |
| Graduated disclosure | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |

### Principle 5: Role-Based Access Control

| Component | Spec | Status |
|-----------|------|--------|
| "Only for verified humans" | `NEW_CHAIN_DOXDEV_SPEC.md` | ✅ **DEPLOYED** (deploy gate) |
| "Only for accredited investors" | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |
| Privacy-preserving compliance | `NEW_CHAIN_DOXDEV_SPEC.md` | ❌ Not implemented |

### Principle 6: Multi-Dimensional Storage Pricing

| Component | Spec | Status |
|-----------|------|--------|
| Tx vs storage vs compute pricing | `NEW_CHAIN_TOKENOMICS.md` | ❌ Not implemented (flat gas) |

### Principle 7: Governance Without Plutocracy

| Component | Spec | Status |
|-----------|------|--------|
| Identity-weighted voting (1 human = 1 vote) | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |
| Direct vote mechanism | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |
| Conviction voting | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |
| Quadratic voting | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |
| Futarchy (prediction-market governance) | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |
| Treasury system | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |
| Emergency freeze mechanism | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |

### Principle 8: Cross-Chain as Native

| Component | Spec | Status |
|-----------|------|--------|
| Bitcoin SPV verification | `NEW_CHAIN_BITCOIN_INTEGRATION.md` | ✅ **DEPLOYED** — Precompile 0x16 |
| Cross-chain attestations | `NEW_CHAIN_CROSS_CHAIN_ATTESTATIONS.md` | ❌ Not implemented |
| LayerZero OFT for 2WAY | `2WAY_SPECIFICATION.md` | ❌ Not implemented |

### Principle 9: Sovereign Recovery

| Component | Spec | Status |
|-----------|------|--------|
| Three-stage account model (onboarding → standard → self-custody) | `NEW_CHAIN_ACCOUNT_SPEC.md` | ❌ Not implemented |
| Guardian-based recovery | `NEW_CHAIN_ACCOUNT_SPEC.md` | ❌ Not implemented |
| Dead man's switch (inheritance) | `NEW_CHAIN_DOXDEV_SPEC.md` | ✅ **DEPLOYED** — Precompile 0x15 |
| Account abstraction (unified model) | `NEW_CHAIN_ACCOUNT_SPEC.md` | ❌ Not implemented |
| No seed phrases at protocol level | `NEW_CHAIN_ACCOUNT_SPEC.md` | ❌ Not implemented |

### Principle 10: Anti-Fraud at Consensus Level

| Component | Spec | Status |
|-----------|------|--------|
| Contract freezing by community vote | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |
| On-chain dispute resolution | `NEW_CHAIN_GOVERNANCE_SPEC.md` | ❌ Not implemented |

### Principle 11: State Rent / State Expiry

| Component | Spec | Status |
|-----------|------|--------|
| State rent mechanism | `NEW_CHAIN_TOKENOMICS.md` | ❌ Not implemented |
| Pruning after 30 days frozen | `NEW_CHAIN_TOKENOMICS.md` | ❌ Not implemented |

### Principle 12: Verifiable Off-Chain Compute

| Component | Spec | Status |
|-----------|------|--------|
| ZK proof verification for external compute | Session `20260620` | ❌ Not implemented |

---

## Part 5: UX Wall — Spec Status

| Component | Spec | Status |
|-----------|------|--------|
| Three-stage onboarding (no seed phrase initially) | `NEW_CHAIN_UX_SPEC.md` | ❌ Not implemented |
| Social recovery (guardian-based) | `NEW_CHAIN_UX_SPEC.md` | ❌ Not implemented |
| Gas abstraction (pay in asset, not native token) | `NEW_CHAIN_UX_SPEC.md` | ❌ Not implemented |
| dApp gas sponsorship | `NEW_CHAIN_UX_SPEC.md` | ❌ Not implemented |
| Session keys (limited permissions, auto-expiry) | `NEW_CHAIN_UX_SPEC.md` | ❌ Not implemented |
| Batched transactions (atomic multi-action) | `NEW_CHAIN_UX_SPEC.md` | ❌ Not implemented |
| Human-readable transaction decoding | `NEW_CHAIN_UX_SPEC.md` | ❌ Not implemented |
| Progressive security (value-based tiers) | `NEW_CHAIN_UX_SPEC.md` | ❌ Not implemented |

---

## Part 6: Oracle Problem — Spec Status

| Component | Spec | Status |
|-----------|------|--------|
| Staked attestors = separate from validators | `NEW_CHAIN_ORACLE_SPEC.md` | ✅ **DEPLOYED** |
| Binary Journal as truth anchoring | `NEW_CHAIN_BINARY_JOURNAL_INTEGRATION.md` | ✅ **DEPLOYED** |
| Consensus on real-world data | `NEW_CHAIN_ORACLE_SPEC.md` | ⚠️ Demo only |
| Data source diversity enforcement | `NEW_CHAIN_ORACLE_SPEC.md` | ❌ Not implemented |
| Verifiable randomness (VRF) | `NEW_CHAIN_ORACLE_SPEC.md` | ❌ Not implemented |
| Time-based execution | `NEW_CHAIN_ORACLE_SPEC.md` | ❌ Not implemented |
| Cross-chain data bridging | `NEW_CHAIN_CROSS_CHAIN_ATTESTATIONS.md` | ❌ Not implemented |
| Graduated trust model | `NEW_CHAIN_ORACLE_SPEC.md` | ❌ Not implemented |

---

## Summary: What's Spec'd But Not Built

| Category | Spec Documents | Components | Priority |
|----------|---------------|------------|----------|
| **Privacy** | Session research | ZK-selective disclosure, private execution | HIGHEST |
| **Governance** | `NEW_CHAIN_GOVERNANCE_SPEC.md` | Voting, treasury, emergency freeze, dispute resolution | HIGH |
| **Account Model** | `NEW_CHAIN_ACCOUNT_SPEC.md` | 3-stage accounts, guardian recovery, account abstraction | HIGH |
| **UX** | `NEW_CHAIN_UX_SPEC.md` | Session keys, gas abstraction, batching, human-readable tx | HIGH |
| **Oracle Extensions** | `NEW_CHAIN_ORACLE_SPEC.md` | VRF, time execution, graduated trust | MEDIUM |
| **Cross-Chain** | `NEW_CHAIN_CROSS_CHAIN_ATTESTATIONS.md` | Cross-chain data bridging, LayerZero OFT | MEDIUM |
| **State Rent** | `NEW_CHAIN_TOKENOMICS.md` | State rent, pruning | MEDIUM |
| **Anti-Fraud** | `NEW_CHAIN_GOVERNANCE_SPEC.md` | Contract freezing, dispute resolution | MEDIUM |
| **Dox_Dev Extensions** | `NEW_CHAIN_DOXDEV_SPEC.md` | Progressive hardening, project wallets, cross-chain blacklist | LOW |

---

## What This Means

**WayChain has deployed the foundation that no other chain has:**
- Dox_Dev + deploy gate (identity-gated deployment)
- Fixed fees (predictable costs)
- Native Bitcoin SPV verification
- Oracle infrastructure (7 precompiles, separate attester sets)
- 1-second finality with instant settlement
- Dead man's switch inheritance
- Truth anchoring (Binary Journal)

**What's spec'd but not built is what every other chain also lacks:**
- Privacy (every chain struggles — we have a ZK plan)
- Governance (most chains have token-weighted plutocracy — we have identity-weighted voting)
- Account abstraction (ERC-4337 exists but <1% adoption — we have a 3-stage model)
- UX abstraction (seed phrases remain the norm — we have progressive security)

**The honest assessment:** We have 19 spec documents covering every gap. The designs are complete. The gap is implementation, not design knowledge. We can build any of these pieces with confidence because the research and architecture are done.

---

## Recommended Implementation Order

| Phase | Items | Spec | Est. Effort |
|-------|-------|------|-------------|
| 6a | State rent + pruning | `NEW_CHAIN_TOKENOMICS.md` | 1 week |
| 6b | Account model (3-stage + guardian recovery) | `NEW_CHAIN_ACCOUNT_SPEC.md` | 2 weeks |
| 6c | Governance (identity-weighted voting + treasury) | `NEW_CHAIN_GOVERNANCE_SPEC.md` | 2 weeks |
| 6d | UX (session keys, gas abstraction, batching) | `NEW_CHAIN_UX_SPEC.md` | 2 weeks |
| 6e | Privacy (ZK-selective disclosure) | Session research | 3 weeks |
| 6f | Cross-chain (LayerZero OFT for 2WAY) | `2WAY_SPECIFICATION.md` | 2 weeks |
| 6g | Oracle extensions (VRF, time execution) | `NEW_CHAIN_ORACLE_SPEC.md` | 1 week |
