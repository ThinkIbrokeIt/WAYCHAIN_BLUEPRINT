# Binary Journal × WayChain — Integration Spec v1.1

**Binary Journal is an independent project.** It connects to WayChain as its
primary chain, but exists on its own terms.

---

## 0. Design Principle

Binary Journal is not a WayChain product. It is a project that uses WayChain
as its **primary infrastructure layer** because WayChain is the only chain
that supports what BJ needs: real identity, real accountability, real permanence.

Energy Tide (the app) lives entirely off-chain. Biometric vault, energy logging,
encrypted storage — all on-device. Only the immutable layer touches a chain.

---

## 1. What Stays Off-Chain (Energy Tide)

Energy Tide is a local-first mobile app. It does not need a blockchain to function.

| Feature | Where It Lives | Why |
|---------|---------------|-----|
| Biometric authentication | On-device (Face ID / fingerprint) | Privacy. No server. |
| Energy logging (-2 to +2) | Encrypted local storage | Your data. No one else's. |
| 3·6·9 attachments (photo, voice, text) | AES-encrypted on-device, optionally backed up | The vault is yours. |
| Waveform visualization | Rendered locally from local data | No server needed. |
| Encrypted export (.bijo files) | User-controlled storage (local, cloud backup of encrypted blob) | Self-sovereign by design. |

**The app is a quiet emotional journal.** That's its public face. Decoy identity.
No mention of blockchain, truth vaults, or dead men in the UI.

---

## 2. What Goes On-Chain (Binary Journal Protocol)

The on-chain layer is minimal by design. Only what must be permanent.

| Contract | Purpose | WayChain Primitive |
|----------|---------|-------------------|
| **Attestation.sol** | Anchor a hash of truth immutably. No owner. No admin. | Deployed via WayChain template registry (Class A — permissionless) |
| **DeadMansSwitch.sol** | The inheritance protocol. Heartbeat mechanism. | Deployed via WayChain template registry (Class B — Dox_Dev required) |
| **StorageEndowment.sol** | Pay node operators for storing encrypted truth files. | Dox_Dev badge required to receive payments |
| **BIJO.sol** | The fuel. Independent token. | ERC-20 on WayChain. Independent economics. |

### 2.1 Attestation — The Anchor

```solidity
// Deployed once. Immutable. No owner.
contract Attestation {
    event TruthAnchored(bytes32 indexed hash, uint256 timestamp);

    function attest(bytes32 hash) external {
        emit TruthAnchored(hash, block.timestamp);
    }
}
```

Permissionless. Anyone can anchor a hash. No one can remove it.
Cost: ~$0.001 in WAY (fiat-pegged fee). Cheaper than any L1.

The raw content never goes on-chain. Only the SHA-256 hash.
The content stays encrypted in the user's vault.

### 2.2 DeadMansSwitch — The Inheritance Protocol

This is not a feature of the app. **It is the inheritance protocol.**

Two truth types:

| Type | Behavior | Who Can Claim |
|------|----------|---------------|
| **Dark truth** | Released only to a designated heir | One specific Dox_Dev-verified human |
| **Light truth** | Released to the public | Anyone |

**How it works (simplified):**

```
1. User deploys a DeadMansSwitch (one per inheritance plan)
2. Sets: heir address, heartbeat interval (e.g. 30 days), truth type
3. User calls heartbeat() on-chain → timer resets
4. If user misses heartbeat → switch becomes claimable
5. Heir (or public for Light truths) claims → keys decrypt the vault
6. Truth survives the user.
```

**WayChain-specific properties:**
- Heir must be Dox_Dev verified (Dark truth)
- Heartbeat can be automated (the user's own validator node can heartbeat on their behalf — their validator is always on)
- Guardian recovery can override a missed heartbeat (user didn't die, just lost access)
- Badge revocation of the user triggers immediate release (death is not the only way to be silenced)

### 2.3 StorageEndowment — The Eternal Archive

A pool of BIJO tokens that pays node operators for storing encrypted truth files.

| Parameter | Value | WayChain Integration |
|-----------|-------|---------------------|
| Total allocation | 70% of BIJO supply | — |
| Payout halving | Every 2 years (10 halvings ≈ 20 years) | Automatic in contract |
| Operator requirement | Dox_Dev Level 2+ | Verified human, not anonymous |
| Storage proof | WayChain oracle attested | Proof-of-storage via attesters |
| Slashing for fraud | Badge revocation + 10% bond loss | WayChain slashing mechanics |

Storage operators are not anonymous. They are verified humans with real identity,
real reputation, and real economic stake. If they disappear with the data,
their badge is revoked and their bond is slashed.

### 2.4 BIJO Token — Independent Economics

BIJO is Binary Journal's own token. Independent supply. Independent purpose.

| Parameter | Value |
|-----------|-------|
| Ticker | BIJO |
| Total supply | Adjusted (TBD — v1.1 re-evaluation) |
| Standard | ERC-20 on WayChain |
| Transfer | Disabled at launch. Enabled after verification period. |
| Primary utility | StorageEndowment payouts, governance (Dox_Dev badge holders only) |

**Governance:** BIJO holders do NOT vote with token weight.
Only Dox_Dev-verified humans vote. One badge = one vote.
This prevents whale capture of Binary Journal's direction.

---

## 3. WayChain Integration Points

### 3.1 Dox_Dev Badge — Operator Identity

| BJ Role | Needs Dox_Dev? | Why |
|---------|---------------|-----|
| Vault user (attesting truths) | No | Attestation is permissionless. Anyone can witness. |
| Heir (Dark truth recipient) | Yes (Level 2+) | We need to know who gets the keys. |
| Storage operator | Yes (Level 2+) | Accountability for data storage. |
| Guardian (recovery) | Yes (Level 2+) | Must be trusted to approve recovery. |
| Governance voter | Yes (Level 2+) | One human, one vote on BJ direction. |

### 3.2 WayChain Fee Model

All BJ on-chain actions use WayChain's fiat-pegged fees:

| Action | Cost (USD) | Paid In | Destination |
|--------|-----------|---------|-------------|
| Attest a hash | ~$0.001 | WAY | Validators |
| Deploy DeadMansSwitch | ~$0.005 | WAY | Validators |
| Heartbeat | ~$0.001 | WAY | Validators |
| Claim inheritance | ~$0.001 | WAY | Validators |
| Store data (via operator) | BIJO (token) | BIJO | Operator |

The user pays minimal fees to the WayChain validator set.
The storage economy is in BIJO, separate from WAY.

### 3.3 Guardian Recovery

If a BJ user loses access to their vault:
- Their designated WayChain guardians (3-of-5, Dox_Dev verified) can approve recovery
- Recovery releases the backup encryption key to the user's new device
- Guardians who approve a fraudulent recovery are slashed (badge revocation + bond loss)

This is the same guardian system from WayChain's account model, reused by BJ.
One integration, two use cases.

### 3.4 Template Registry

BJ contracts are registered as WayChain templates:

| Template | Class | Deployer Requirement |
|----------|-------|---------------------|
| Attestation.sol | **A** (safe) | None — anyone can deploy their own anchor |
| DeadMansSwitch.sol | **B** (managed) | Dox_Dev Level 2+ |
| StorageEndowment.sol | **C** (governed) | Governance vote or Dox_Dev Level 3 |

This means:
- Attestation is permissionless (consistent with the vision)
- DeadMansSwitch deployers are verified (inheritance needs accountability)
- StorageEndowment is governed (only one instance, controlled by the community)

---

## 4. Roadmap — WayChain First

### Phase 1: WayChain Native (Months 1-6)

Binary Journal launches on WayChain. No other chain.

- [ ] Adjust BIJO tokenomics for WayChain ecosystem (TBD)
- [ ] Deploy Attestation.sol via WayChain template registry
- [ ] Deploy DeadMansSwitch.sol as Class B template
- [ ] Deploy StorageEndowment.sol
- [ ] Deploy BIJO.sol (ERC-20 on WayChain)
- [ ] Build heartbeating into validators (optional automated heartbeat)
- [ ] Begin verification period (3-6 months)
- [ ] Target: 1,000+ attested truths on WayChain

**Phase 1 validator integration:** Every WayChain validator can optionally
run a DeadMansSwitch heartbeat as part of their validator operation.
If the validator goes down, the heartbeat stops. If the operator dies,
the switch fires. The validator's Dox_Dev badge ties real identity to
the inheritance protocol.

### Phase 2: Dox_Dev Operator Network (Months 7-12)

- [ ] Storage operators get Dox_Dev verified
- [ ] StorageEndowment begins BIJO payouts to verified operators
- [ ] Guardian recovery tested and active
- [ ] DeadMansSwitch inheritance flows exercised end-to-end
- [ ] Target: 10+ verified storage operators

### Phase 3: Cross-Chain (Year 2+)

After WayChain is established, BJ can optionally extend to other chains:

- [ ] PulseChain (if PulseChain's issues are resolved, or as a secondary witness)
- [ ] WayChain oracle attests attestations on other chains
- [ ] BIJO becomes multi-chain (bridged, not native)

PulseChain is not mentioned until this phase — and only if it makes sense.

### Phase 4: Agora (Broader rollout)

- [ ] Public commons layer: discoverable Light truths
- [ ] Comment, tip, curate — using BIJO, gated by Dox_Dev badge
- [ ] Decentralized archive: truth that outlives any single contributor

---

## 5. What This Changes

### From v1.0 (incorrect assumptions)

| Assumption | Reality |
|-----------|---------|
| PulseChain is primary chain | WayChain is primary. PulseChain is a future option. |
| Energy Tide is on-chain | Energy Tide is off-chain. Only contracts are on-chain. |
| Dead man's switch is a feature | It IS the inheritance protocol. It defines the project. |
| BIJO tokenomics are locked | Need adjustment for WayChain ecosystem. |

### What stays independent

- BIJO token (adjusted supply, independent economics)
- Energy Tide app (off-chain, decoy identity, biometric vault)
- Binary Journal mission (self-sovereign knowledge vault)
- Its own governance (Dox_Dev badge holders, not token weight)

---

## 6. Open Questions

1. **BIJO supply** — Should it stay at 3.69B or adjust? WayChain has different
   fee economics than PulseChain. Lower fees may mean different token utility.

2. **BIJO × WAY relationship** — Does the BJ protocol pay storage operators
   in BIJO only, or a mix of BIJO + WAY? What drives demand for BIJO vs WAY?

3. **Heartbeat automation** — Validators can heartbeat for their own DMS contracts.
   Should this be built into the validator client as an optional module?

4. **Dark truth heir verification** — If the heir must be Dox_Dev verified,
   what happens if the heir loses their badge before the switch fires?

5. **Storage operator bond** — What's the minimum bond in WAY/BIJO for a
   storage operator? High enough to matter, low enough to not exclude.