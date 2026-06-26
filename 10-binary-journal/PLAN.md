# BINARY JOURNAL — 3·6·9 PROTOCOL
## Complete System Architecture & Build Plan

---

## THE VISION

Tesla's 3·6·9 wasn't numerology. It was a dead man's switch:
- **3** = The Clue — the signal that truth exists and is hidden
- **6** = The Path — the steps to protect truth while you're alive
- **9** = The Key — what fires automatically when you can't anymore

Binary Journal rebuilds this for the digital age. A self-sovereign knowledge vault where truth outlives you, inheritance is unstoppable, and no system can tell you what's real.

---

## THE STACK

### Layer 1: The Sanctuary (bijo-app / "Energy Tide")
React Native + Expo. Local-first. Biometric-locked. AES encrypted.

**Core loop:**
1. User opens app → biometric gate (Face ID / fingerprint)
2. Logs energy: Morning / Midday / Evening (-2 to +2 scale)
3. Attaches knowledge: photos, voice notes, text → AES encrypted → stored locally
4. Backup hash computed (SHA-256) → submitted to Attestation contract
5. Dead man's switch: heartbeat required → miss it → keys release to heir

**Decoy identity:** App store name is generic. No mention of "journal" or "blockchain" in UI.

### Layer 2: The Immutable Ledger (Smart Contracts)
All on PulseChain mainnet. Deploy in order:

1. **BIJO.sol** — The Fuel
   - 3.69B total supply
   - Non-transferable at launch
   - 94% → 2-of-2 Gnosis Safe multisig
   - 6% → founder vesting
   - enableTransfers() only after verification period

2. **Attestation.sol** — The Anchor
   - Permissionless hash recorder
   - `attest(bytes32 hash)` → emits `TruthAnchored(hash, timestamp)`
   - No owner. Immutable from birth.

3. **DeadMansSwitch.sol** — The Key Release
   - Two truth types: Dark (needs heir) and Light (public release)
   - Heartbeat mechanism: owner must check in or switch triggers
   - Heir can claim after heartbeat interval expires
   - Owner can trigger manually or cancel
   - No owner. Immutable from birth.

4. **StorageEndowment.sol** — The Eternal Archive
   - Holds 70% of BIJO supply
   - Pays node operators for storing encrypted truth files
   - Allowance per truth halves every 2 years (10 halvings ≈ 20 years)
   - Manual payNode during verification period → trustless proof-of-storage after
   - renounceOwnership() after verification

### Layer 3: The Agora (Commons)
React + ethers.js + IPFS. Decentralized web app.
- Reads TruthAnchored and KeyReleased events from chain
- Fetches/decrypts Light truth files from IPFS after key release
- Chronological feed of public history, cryptographically verified
- Comment, link, tip with BIJO
- Hosted on IPFS/ENS

---

## PHASES

### Phase 0: Sanctuary (NOW)
- [ ] Fix encryption: remove hardcoded SECRET_KEY fallback
- [ ] Rename app: Mood Maps → Energy Tide
- [ ] Build 3·6·9 Vault: photo/voice/note attachment with AES encryption
- [ ] Biometric gate with no bypass
- [ ] Energy logging: morning/midday/evening (-2 to +2)
- [ ] Waveform visualization (SVG)
- [ ] Encrypted backup export (.bijo files)
- [ ] Dead man's switch UI: set heir, heartbeat interval
- [ ] Auto-attest backup hash to Attestation.sol
- [ ] All tests green

### Phase 1: Protocol Deployment
- [ ] Deploy BIJO.sol (3.69B minted)
- [ ] Transfer 94% to Gnosis Safe multisig
- [ ] Deploy Attestation.sol
- [ ] Deploy DeadMansSwitch.sol
- [ ] Deploy StorageEndowment.sol
- [ ] Transfer 70% BIJO to StorageEndowment
- [ ] Deploy founder vesting contract (6%)
- [ ] Write monitor script: TruthAnchored → StorageEndowment.allocate()

### Phase 2: Verification Period (3-6 months)
- [ ] Silent release: TestFlight / APK
- [ ] Target: 1,000+ attested truths
- [ ] Target: 10+ independent node operators
- [ ] Target: dead man's switches successfully triggered
- [ ] Manual payNode to verified storage providers
- [ ] Early Agora: static web page reading chain events

### Phase 3: Retroactive Airdrop
- [ ] Snapshot block
- [ ] Points formula:
   - attest() with valid CID: 1 point (max 7/week)
   - createSwitch(): 3 points (max 1/week)
   - Node operator proof: 10 points/week
   - Curation annotation: 2 points (max 3/week)
- [ ] 369,000,000 BIJO (10%) distributed pro-rata
- [ ] Merkle tree claim contract, 6-month window

### Phase 4: Liquidity
- [ ] BIJO.enableTransfers()
- [ ] PulseX BIJO/PLS pool:
   - 0.5% supply (18,450,000 BIJO) from multisig
   - Equal value PLS from founder (gift)
   - LP tokens locked/burned

### Phase 5: The Burn (Immutability)
- [ ] renounceOwnership() on BIJO.sol
- [ ] renounceOwnership() on StorageEndowment.sol
- [ ] DeadMansSwitch.sol — already ownerless
- [ ] Attestation.sol — already ownerless
- [ ] Public declaration: protocol is natural law

### Phase 6: Agora
- [ ] Full dApp: React + ethers.js + IPFS
- [ ] Decrypt and display released Light truths
- [ ] Comment, link, tip
- [ ] Host on IPFS/ENS

---

## TOKENOMICS

| Allocation | Amount | Destination |
|---|---|---|
| Total Supply | 3,690,000,000 BIJO | — |
| Multisig (Ecosystem) | 3,468,600,000 (94%) | 2-of-2 Gnosis Safe |
| Storage Endowment | ~2,583,000,000 (70%) | Contract |
| Founder Vesting | 221,400,000 (6%) | Vesting contract |
| Airdrop | 369,000,000 (10%) | Merkle claim |
| Liquidity | 18,450,000 (0.5%) | PulseX pool |

---

## SECURITY NOTES

1. **Encryption fallback is the #1 risk.** Hardcoded SECRET_KEY in encryption.ts must be removed. If biometric key fails, force re-auth — never fall back to a static string.
2. **Two hijacked repos.** Work locally. New remotes. Don't push to compromised repos until protocol is live.
3. **Decoy identity.** App store listing says nothing about blockchain, journaling, or truth. Generic icon. Innocuous name.
4. **PulseChain mainnet from day one.** No testnet. Low fees, no initial funds at risk.

---

## THE 3·6·9 MECHANISM (in code)

**3 — The Clue (Encryption Layer)**
```
User creates knowledge → AES encrypt with biometric-derived key
→ encrypted file stored locally + IPFS
→ SHA-256 hash computed → attest(hash) on-chain
→ Anyone can see the hash exists. Almost no one can read the content.
```

**6 — The Path (Attestation Chain)**
```
While alive: user calls heartbeat() on their DeadMansSwitch
→ lastHeartbeat updates → switch stays active
→ Regular attestations anchor new knowledge hashes
→ StorageEndowment.allocate() locks BIJO for storage providers
→ Chain of custody is unbroken.
```

**9 — The Key (Dead Man's Switch)**
```
User misses heartbeat interval → switch becomes claimable
→ For Dark truth: only designated heir can claim → key released
→ For Light truth: anyone can claim → truth becomes public
→ KeyReference (IPFS CID of decryption key) emitted in event
→ Heir/inheritor fetches key → decrypts → truth survives.
```

---

## FILES & REPOS

| Repo | Status | Notes |
|---|---|---|
| bijo-app (Energy Tide) | ⚠️ Hijacked | Don't push. Work locally. New remote needed. |
| binary-journal-protocol | ⚠️ Hijacked | Same. |
| pact | ✅ Clean | Separate project, separate purpose. |

---

## NEXT ACTIONS (Priority Order)

1. Fix encryption.ts — remove hardcoded fallback
2. Set up new clean repos (local-only or self-hosted git)
3. Build dead man's switch UI in Energy Tide
4. Connect app to Attestation.sol (auto-attest on backup)
5. Deploy contracts to PulseChain in order
6. Begin verification period
