# INHERITANCE DECRYPTION — Missing Pieces Analysis

## The Problem

When the dead man's switch triggers, the heir gets a `keyReference` (IPFS CID) from the `KeyReleased` event.
But the heir **cannot decrypt anything** because:

1. The encryption key is derived from the **biometric of the deceased** — gone forever
2. The encrypted files are on the deceased's device or IPFS — the heir doesn't have them
3. There's no mechanism to wrap the encryption key so someone else can unlock it

**Current state:** The `KeyReleased` event emits `keyReference` but there's no way for the heir to actually use it.

---

## The Solution: Key Wrapping + IPFS Storage

The mechanism has 3 parts:

### Part 1: Encrypt-then-Wrap (in Energy Tide app)

When a user creates a Dead Man's Switch, the app must:

```
1. Generate a random "truth key" (AES-256) — encrypts all files for this switch
2. Encrypt each file with the truth key
3. Upload encrypted files to IPFS → get fileCid
4. Wrap (encrypt) the truth key with the user's biometric-derived public key
5. Upload the wrapped key to IPFS → get keyReference (CID)
6. Store on-chain: createSwitch(heir, truthType, keyReference, fileCid, interval)
```

The wrapped key can ONLY be decrypted by the biometric key of the owner while alive.

### Part 2: Key Release on Switch Trigger

When the switch triggers (heartbeat expires or owner releases):

```
Option A — Owner alive (manual trigger):
  Owner calls keyRef = contract.triggerRelease(id)
  The keyReference CID is emitted in KeyReleased event
  Owner (or app) fetches the wrapped key from IPFS
  Owner decrypts it with their biometric key
  Owner re-encrypts (wraps) the truth key with the HEIR's public key
  Uploads the re-wrapped key to IPFS
  Sends the new CID to the heir (off-chain: email, messenger, etc.)
  Heir downloads, decrypts with their biometric key → gets truth key → decrypts files

Option B — Owner deceased (auto-trigger):
  The keyReference is the IPFS CID of the truth key wrapped with owner's biometric key
  BUT the owner is dead → biometric key is gone → unwrappable
  
  THIS IS THE FUNDAMENTAL PROBLEM.
```

### Part 3: How to Solve the Dead Biometric Problem

There are several approaches:

#### Approach A: Shamir's Secret Sharing (Recommended)
```
Owner sets up their switch:
1. Truth key is generated
2. Truth key is split into N shards using Shamir's Secret Sharing
3. K of N shards required to reconstruct (e.g., 3 of 5)
4. Each shard is encrypted with a different recipient's public key
5. Shard CIDs stored on IPFS
6. Heir locations stored in the contract's keyReference field (or on-chain mapping)

When switch triggers:
- Heirs collect K shards
- Each decrypts their shard with their biometric key
- K shards reconstruct the truth key
- Truth key decrypts files from IPFS
```

#### Approach B: Timelock Encryption
```
Owner's truth key is encrypted with a timelock puzzle
The puzzle takes longer to solve than the owner's heartbeat interval
If owner doesn't reset, the timelock puzzle is publicly solvable
Heir (or anyone for Light truth) solves the timelock → gets truth key
```

#### Approach C: Re-wrapping Before Death (Pragmatic)
```
Periodically (or at switch creation), the owner:
1. Wraps the truth key with the heir's public key
2. Stores the wrapped shard on IPFS
3. The CID is the keyReference

When switch triggers:
Heir fetches the pre-wrapped key from IPFS
Decrypts with their own biometric key
Uses truth key to decrypt files

Problem: If the owner dies unexpectedly without pre-wrapping, the key is lost.
Mitigation: App automatically pre-wraps on switch creation AND periodically re-wraps with updated keys.
```

---

## Recommended Architecture: Hybrid A + C

Combine Shamir's Secret Sharing with periodic re-wrapping:

### In the Energy Tide app (Dead Man's Switch creation screen):

```
User creates a switch:
1. Inputs: heir addresses, heartbeat interval, truth type (Dark/Light)
2. App generates random truth key (32 bytes)
3. For each date/attachment:
   - Encrypt file with truth key → upload to IPFS
   - Store encrypted file CID
4. For EACH heir:
   - Generate heir-specific wrapping key
   - Wrap truth key with heir's public key (or passphrase-derived key)
   - Upload wrapped key to IPFS → get shard CID
   - Store: heir address → shard CID mapping on IPFS (or in contract)
5. Call createSwitch() on-chain with:
   - heir: heir address
   - truthType: Dark or Light
   - keyReference: IPFS CID of the shard registry (maps heir → shard CID)
   - fileCid: IPFS CID of the encrypted file manifest
   - heartbeatInterval: e.g., 30 days
```

### Inheritance flow:

```
Owner dies → heartbeat expires → heir calls claim(switchId)
→ Contract emits KeyReleased(switchId, heir, keyReference)
→ Heir app listens for KeyReleased events where heir == their address
→ Fetches keyReference from IPFS (shard registry)
→ Finds their shard CID in the registry
→ Downloads their wrapped shard from IPFS
→ Decrypts shard with their biometric key → gets truth key portion
→ If K-of-N: collects other shards from other heirs/custodians
→ Reconstructs full truth key
→ Downloads encrypted files from IPFS (from fileCid manifest)
→ Decrypts files with truth key
→ Truth is inherited.
```

---

## Files That Need to Be Created/Modified

| File | What to Add |
|---|---|
| `src/utils/keyWrapping.ts` | Shamir's split/reconstruct, key wrapping/unwrapping with public keys |
| `src/hooks/useInheritance.ts` | Hook for heirs: detect KeyReleased events, fetch shards, reconstruct key, decrypt |
| `src/types/index.ts` | Add: InheritanceShard, SwitchConfig, KeyReleaseEvent types |
| `src/store/useStore.ts` | Add: createDeadMansSwitch(), heartbeat(), getKeyReleaseEvents(), claimInheritance() |
| `app/index.tsx` | New UI: switch creation screen, heir management, heartbeat button, inheritance claiming |
| `src/utils/encryption.ts` | Add: wrapKey(), unwrapKey(), generateTruthKey() |
| `contracts/DeadMansSwitch.sol` | Add: shardRegistry mapping, multi-heir support |

---

## Critical Gap Summary

The ENTIRE inheritance decryption pipeline is missing. Specifically:

1. **No truth key generation** — app encrypts with biometric-derived key directly. Need a SEPARATE truth key for each switch.
2. **No key wrapping** — no mechanism to wrap the truth key so heirs can unwrap it.
3. **No IPFS upload** — encrypted files stay on device. Heir can't access them.
4. **No IPFS decryption** — even if heir gets the files, they need the truth key.
5. **No shard management** — no Shamir's Secret Sharing, no multi-heir support.
6. **No inheritance UI** — no screen to create switches, manage heirs, claim inheritance.
7. **No event listener** — app doesn't watch for KeyReleased events.

This is the single biggest missing piece. Without it, the dead man's switch is just an event emitter — it tells you something happened but gives you nothing useful.
