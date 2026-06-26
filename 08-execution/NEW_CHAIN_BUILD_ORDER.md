# Build Order — Dependency Map

What blocks what, and what order we should spec+ship.

---

## Dependency Graph

```
Layer 0: Consensus + EVM
  │
  ├──► Account Model (how accounts work, keys, signing)
  │     │
  │     ├──► Dox_Dev (badges on accounts, deploy gates)
  │     │     │
  │     │     ├──► Trustless Lock templates (Class B safety)
  │     │     └──► Contract classification (Class A/B/C/D checks)
  │     │
  │     ├──► UX Onboarding Ledger (Stage 0 accounts, recovery)
  │     │
  │     └──► Session keys, gas abstraction (UX primitives)
  │
  ├──► Native Oracle (attesters, TLS proofs, dispute)
  │     │
  │     ├──► Cross-chain attestations
  │     └──► Binary Journal truth anchoring (3·6·9 accum.)
  │
  └──► State rent / state expiry
```

---

## Must-Have at Genesis (Layer 0)

These are non-negotiable. Without them, the chain doesn't exist.

| Component | Depends On | Spec Status | File |
|-----------|-----------|-------------|------|
| **Consensus mechanism** | Nothing (foundation) | ✅ Done — BFT PoS, 200 validators, sqrt-weighted random lottery, 1s blocks, instant finality | `NEW_CHAIN_CONSENSUS_SPEC.md` |
| **EVM execution layer** | Consensus | ✅ Done — Fully EVM-compatible, 8 new opcodes (0xF0-0xF7), 7 precompiles (0x0C-0x12), parallel lanes, contract classification | `NEW_CHAIN_EVM_SPEC.md` |
| **Account model + abstraction** | EVM | ✅ Done — Unified accounts, 3 stages (0/basic/full), guardian recovery, key rotation, session keys | `NEW_CHAIN_ACCOUNT_SPEC.md` |
| **Native token economics** | Consensus, EVM | ✅ Done — Dynamic supply curve (no fixed cap), 7% target APY, 60% state rent burn, 50% slash burn, validator + oracle + template + Dox_Dev fee flows | `NEW_CHAIN_TOKENOMICS.md` |

All four foundation specs are written and ready for implementation.

---

## Must-Have Before Public Deployment

These protect users from the scam problem we're trying to solve.
The chain should not be usable by the public until these are active.

| Component | Depends On | Why Critical | Status | File |
|-----------|-----------|-------------|--------|------|
| **Dox_Dev** | Account model | Stops anonymous deploys at genesis | ✅ Done — Badge integration, 3-level verification, Class C/D deploy gates | `NEW_CHAIN_DOXDEV_SPEC.md` |
| **Contract classification** | Dox_Dev, EVM | Prevents unverified risk levels; Class A/B/C/D enforced at protocol level | ✅ Done (embedded in EVM spec) | `NEW_CHAIN_EVM_SPEC.md` |
| **Trustless Lock templates** | Dox_Dev, EVM | Memecoin safety enforced at protocol level; template contract orchestrates atomic deploy (user never touches LP tokens) | ✅ Done — 3 lock types (time/vesting/multi-sig), template registry, atomic deploy flow | `NEW_CHAIN_TEMPLATES_SPEC.md` |

If the chain launches without Dox_Dev, the same scammers from the
PulseChain investigation (1,152 backdoored pairs, 5,208 contracts)
will deploy here on day one.

---

## Important But Not Blocking Launch

These are differentiators that make the chain truly better, but the
chain functions without them at launch.

| Component | Depends On | Priority | Status | File |
|-----------|-----------|----------|--------|------|
| **Native oracle** | Consensus, EVM | High — enables real-world use cases | ✅ Done — Separate attester set, TLS proofs, graduated trust oracle census, dispute model | `NEW_CHAIN_ORACLE_SPEC.md` |
| **UX onboarding ledger** | Account model, EVM, Oracle (for $ limits) | High — enables normie adoption | ✅ Done — 3-stage account progression, session keys, gas abstraction, guardian recovery | `NEW_CHAIN_UX_SPEC.md` |
| **Gas abstraction** | Account model, EVM | High — UX requirement | ✅ Done (embedded in UX spec + EVM spec — pay with any token) | `NEW_CHAIN_UX_SPEC.md` |
| **Session keys** | Account model | High — UX requirement | ✅ Done (embedded in account spec — ephemeral permissions, device-bound) | `NEW_CHAIN_ACCOUNT_SPEC.md` |

---

## Long-Term Vision (Phase 2+)

These are what make the chain revolutionary, but they can ship after
launch. The chain is still valuable without them.

| Component | Depends On | Status |
|-----------|-----------|--------|
| **Binary Journal** | EVM, Oracle | 🔲 Not specced — 3·6·9 accumulation protocol |
| **Cross-chain attestations** | Oracle, Consensus | 🔲 Not specced — extension of oracle |
| **State rent / expiry** | EVM | ✅ Done (spec'd in tokenomics — 60% burn on state rent) |
| **Governance 2.0** | Dox_Dev, Account model | 🔲 Not specced — quadratic/liquid/futarchy |

---

## Recommended Spec & Build Order

### Phase 0 — Foundation (✅ written)

```
1. ✅ Consensus mechanism
   └── PoS design: BFT, 200 validators, sqrt-weighted lottery, 1s blocks
   └── File: NEW_CHAIN_CONSENSUS_SPEC.md
2. ✅ EVM spec
   └── Opcodes 0xF0-0xF7, precompiles 0x0C-0x12, parallel lanes, contract class
   └── File: NEW_CHAIN_EVM_SPEC.md
3. ✅ Account model
   └── Unified accounts, 3 stages, guardians, session keys
   └── File: NEW_CHAIN_ACCOUNT_SPEC.md
4. ✅ Tokenomics
   └── Dynamic supply, fee flows, state rent, slashing, treasury
   └── File: NEW_CHAIN_TOKENOMICS.md
```

### Phase 1 — Safety (✅ written)

```
5. ✅ Dox_Dev integration
   └── Badge → deploy gate mapping, 3-level verification
   └── File: NEW_CHAIN_DOXDEV_SPEC.md
6. ✅ Trustless Lock templates
   └── Template registry, atomic deploy, 3 lock types
   └── File: NEW_CHAIN_TEMPLATES_SPEC.md
7. ✅ User flow walkthrough
   └── Alice creates memecoin: template contract orchestrates everything
   └── File: NEW_CHAIN_USER_FLOW.md
```

### Phase 2 — UX & Data (✅ written)

```
8. ✅ UX onboarding ledger
   └── Stage 0/1/2 accounts, recovery, gas abstraction
   └── File: NEW_CHAIN_UX_SPEC.md
9. ✅ Oracle consensus model
   └── Separate attester set, TLS proofs, graduated trust, disputes
   └── File: NEW_CHAIN_ORACLE_SPEC.md
```

### Phase 3 — The Vision (🔲 not specced yet)

```
10. 🔲 Binary Journal integration  — 3·6·9 accumulation protocol
11. 🔲 Cross-chain attestation layer — Oracle extension
12. 🔲 Governance 2.0 — Quadratic/liquid/futarchy
```

---

## The Blocker We'd Hit If Wrong

The biggest risk is launching without Dox_Dev + contract classification.
That recreates the exact problem we're trying to solve — anonymous
deployers running scams on our chain.

Second biggest: launching the onboarding ledger before the account
model is stable. Stage 0 recovery depends on how accounts work. If
the account model changes, the onboarding ledger has to be rewritten.

## L2 / Scaling — When and Why We'd Need It

**Short answer:** Our design eliminates the artificial bottlenecks (gas
auctions, state bloat, oracle congestion, MEV, whale dominance) but does
NOT solve the fundamental EVM single-threaded throughput limit. We won't
need an L2 for years at moderate usage, but certain use cases will
eventually outgrow the L1.

### What We Fixed At The Protocol Level

| Bottleneck | How We Fixed It | Confidence |
|------------|----------------|-----------|
| Gas fee spikes | Fixed base fee (not auction-based), priority tip optional | ✅ High |
| State bloat | State rent + 60% burn, pruning after 30 days frozen | ✅ High |
| Oracle congestion | Separate execution lane, 5M gas dedicated, 1-block delay | ✅ High |
| MEV / proposer manipulation | Deterministic block building (tx ordered by hash) | ✅ High |
| Whale validator dominance | sqrt-weighted random lottery, equal voting power per validator | ✅ High |

### What Remains As A Real Limit

| Limit | Our Cap | When It Bites |
|-------|---------|---------------|
| EVM single-threaded execution | 30M gas/block (~600 simple txs) | During NFT mints, mass adoption spikes |
| 1s block propagation | Theoretical (NY-Tokyo latency is ~150ms alone) | At global validator scale |
| Active state growth | ~100K contracts ≈1GB | At 1M+ contracts |
| Oracle lane capacity | ~500 attestations/block | At 500+ data feeds |

### Will We Need An L2?

**Yes — for specific use cases.** But not because the L1 is broken.
Because some things shouldn't live on an L1:

- **High-frequency DEX / gaming** → Needs sub-second finality, not 1s
- **Massive data storage** → Needs Arweave/Filecoin integration, not chain state
- **Real-time social** → Needs a consensus optimized for social graphs

**When we do need an L2, ours inherits all our primitives.** Dox_Dev,
the oracle model, Trustless Lock, the account model — they all work at
L2 level too. The L2 doesn't rebuild the safety layer.

### Honest Assessment

| Scenario | L2 Needed? | Timeline |
|----------|-----------|----------|
| Safe memecoin deploys + DeFi + governance | No | Years |
| 10,000 tps global payment settlement | Yes | Depends on growth |
| Gaming with sub-second finality | Yes | Parallel lane could help |
| Enterprise data anchoring | No (use state rent) | Never for L1 |

**Ethereum ran for 7 years before L2s were necessary.** Our gas model
(govened base fee, not auction) and parallel lanes give us more runway,
not less. The L2 question is a feature priority decision, not a design
flaw.

### L2 Design Principles (when the time comes)

```
1. Inherit L1 primitives (Dox_Dev, account model, oracle) — don't rebuild
2. Use L1 for settlement + data availability (not a separate consensus)
3. L2 operators are Dox_Dev-verified (same deploy gates apply)
4. Trustless Lock works across L1 ↔ L2 bridges
5. Native token flows between L1 and L2 without wrapping
```

---

## Summary Status — All Specs

| Phase | Component | Status | File |
|-------|-----------|--------|------|
| **0** | Consensus mechanism | ✅ Written | `NEW_CHAIN_CONSENSUS_SPEC.md` |
| **0** | EVM execution layer | ✅ Written | `NEW_CHAIN_EVM_SPEC.md` |
| **0** | Account model | ✅ Written | `NEW_CHAIN_ACCOUNT_SPEC.md` |
| **0** | Token economics | ✅ Written | `NEW_CHAIN_TOKENOMICS.md` |
| **1** | Dox_Dev integration | ✅ Written | `NEW_CHAIN_DOXDEV_SPEC.md` |
| **1** | Trustless Lock templates | ✅ Written | `NEW_CHAIN_TEMPLATES_SPEC.md` |
| **1** | User flow walkthrough | ✅ Written | `NEW_CHAIN_USER_FLOW.md` |
| **2** | UX onboarding ledger | ✅ Written | `NEW_CHAIN_UX_SPEC.md` |
| **2** | Oracle consensus model | ✅ Written | `NEW_CHAIN_ORACLE_SPEC.md` |
| **3** | Binary Journal | 🔲 Not specced | — |
| **3** | Cross-chain attestations | 🔲 Not specced | — |
| **3** | Governance 2.0 | 🔲 Not specced | — |

**9 of 12 components specced and ready for implementation.**