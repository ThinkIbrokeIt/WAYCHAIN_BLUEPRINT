# BINARY JOURNAL — Gap Analysis: Repo vs Manual

## Comparison Date: 2026-05-29
Compared: Builder's Manual (BUILDERS_MANUAL.md) vs actual repo files

---

## REPO STATUS

### binary-journal-protocol repo
| File | Status |
|---|---|
| Attestation.sol | ⚠️ Old version (submit/hashValue → should be attest/hash) |
| BIJO.sol | ❌ Missing |
| DeadMansSwitch.sol | ❌ Missing |
| StorageEndowment.sol | ❌ Missing |
| FounderVesting.sol | ❌ Missing |
| monitor-attestations.js | ❌ Missing |
| airdrop-merkle.js | ❌ Missing |
| Agora dApp | ❌ Missing |

### bijo-app repo
| File | Status |
|---|---|
| contracts/BIJO.sol | ✅ Correct |
| contracts/DeadMansSwitch.sol | ✅ Correct |
| contracts/StorageEndowment.sol | ✅ Correct |
| contracts/Attestation.sol | ⚠️ Old version (submit/hashValue → should be attest/hash) |
| scripts/monitor-attestations.js | ✅ Exists |
| scripts/airdrop-merkle.js | ✅ Exists |
| src/utils/encryption.ts | ✅ OK for now (hardcoded key is intentional pre-deployment) |
| src/types/index.ts | ⚠️ Missing DeadMansSwitch + TruthType types |
| src/store/useStore.ts | ⚠️ Missing switch create/heartbeat/hash methods |
| src/hooks/use369Vault.ts | ⚠️ Missing switch UI + auto-attest |
| src/utils/backup.ts | ✅ Correct |
| app/index.tsx | ⚠️ Missing dead man's switch UI, heir management, auto-attest |
| FounderVesting.sol | ❌ Missing entirely |
| Agora dApp | ❌ Missing entirely |

---

## PRIORITIZED FIX LIST

### 🔴 CRITICAL (Blocks basic functionality)
1. **Attestation.sol** — rename `submit()` → `attest()`, `hashValue` → `hash` (in BOTH repos)
2. **src/types/index.ts** — add DeadMansSwitch + TruthType types
3. **src/store/useStore.ts** — add createSwitch(), heartbeat(), computeBackupHash()
4. **FounderVesting.sol** — create new (4yr linear vest, 1yr cliff)

### 🟡 HIGH (Needed before deployment)
5. **binary-journal-protocol repo** — add BIJO.sol, DeadMansSwitch.sol, StorageEndowment.sol
6. **app/index.tsx** — dead man's switch creation UI + heartbeat + heir management + auto-attest
7. **src/hooks/use369Vault.ts** — integrate switch creation flow
8. **Gas planning** — airdrop claims need PLS for gas; plan for relayer or dust drops

### 🟢 MEDIUM (Post-deployment)
9. **Agora dApp** — React + ethers.js + IPFS commons
10. **README.md** — both repos need real documentation
11. **Encryption fallback removal** — before production release only
12. **Test coverage** — switch methods, backup hash attestation

---

## NOTES

- Hardcoded SECRET_KEY in encryption.ts is INTENTIONAL for development.
  Gets replaced with biometric-only key before silent release (Phase 0 checklist).
- Both repos are considered HIJACKED. Work locally. New remotes before protocol is live.
