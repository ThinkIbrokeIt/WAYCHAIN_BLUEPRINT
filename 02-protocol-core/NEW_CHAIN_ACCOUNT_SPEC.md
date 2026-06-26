# Account Model — Protocol Spec v0.1

**Design:** Unified account model (no EOA vs contract distinction)
**Stages:** 0 (onboarding), 1 (standard), 2 (self-custody)
**Keys:** secp256k1, rotated via key rotation protocol
**Recovery:** Guardian-based multi-sig at protocol level

---

## 1. Account Structure

### 1.1 Unified Account

There is no distinction between EOAs (externally owned accounts) and
contract accounts at the protocol level. Every account is the same
structure:

```
Account {
    // ── Core fields (always present) ──
    nonce: uint64,                // Transaction count (anti-replay)
    balance: uint256,             // Native token balance
    storageRoot: bytes32,         // Merkle root of account storage
    codeHash: bytes32,            // hash of deployed code (empty = no code)
    code: bytes,                  // Deployed bytecode (empty for user accounts)

    // ── Key field (unique to this chain) ──
    // Instead of EOA being "address = key hash", every account has
    // a key field that can be rotated. Empty for contract-only accounts.
    key: PublicKey,               // secp256k1 public key (33 or 65 bytes)
    keyType: uint8,               // 0=no key, 1=secp256k1, 2=BLS, 3=multisig

    // ── Security stage ──
    stage: uint8,                 // 0=onboarding, 1=standard, 2=self-custody
    stageSince: uint64,           // Block when current stage began

    // ── Dox_Dev integration ──
    doxDevBadge: uint256,         // Token ID of Dox_Dev badge (0 = none)
    doxDevLevel: uint8,           // 0-3, cached for fast reads
    doxDevExpiry: uint64,         // Block when badge expires

    // ── Recovery / guardians ──
    guardianRoot: bytes32,        // Merkle root of guardian addresses + weights
    guardianCount: uint8,         // How many guardians registered
    guardianThreshold: uint8,     // Approvals needed for recovery
    recoveryTimeout: uint64,      // Blocks before auto-recovery
    recoverySequence: uint64,     // Increments on each recovery (prevents replay)

    // ── Session keys ──
    sessionKeyRoot: bytes32,      // Merkle root of active session keys
    sessionKeyCount: uint16,      // How many session keys active

    // ── Spending limits (Stage 0) ──
    dailySendLimit: uint256,      // Max native token per day
    dailySendUsed: uint256,       // Spent today
    dailySendResetBlock: uint64,  // Block of last reset
    maxAccountValue: uint256,     // Max total value allowed

    // ── Dead man's switch ──
    deadmanBlock: uint64,         // Block after which switch triggers (0=off)
    deadmanHeir: address,         // Who receives assets on trigger
    deadmanInterval: uint64,      // How often to check-in (blocks)

    // ── State rent ──
    rentPaid: uint256,            // Total rent paid
    lastRentBlock: uint64,        // Last block rent was paid
    frozen: bool,                 // True if rent in arrears
}
```

**Total size per active account:** ~512 bytes (before storage/code)

### 1.2 Address Derivation

Addresses are derived the same way as Ethereum:

```
address = keccak256(publicKey)[12..32]
```

For contract creation:
```
address = keccak256(deployerAddress, nonce)[12..32]
// or for CREATE2:
address = keccak256(0xFF, deployerAddress, salt, keccak256(bytecode))[12..32]
```

The difference: a single address can have BOTH a key (acts as a user
account) AND code (acts as a contract account). The execution context
determines which applies:

- If `tx.to` matches address with a key → process as user transaction
- If `tx.to` matches address with code → process as contract call
- If both → contract call (user operations go through the key path)

This enables:
- User accounts that also act as smart wallets
- Contract accounts that can send transactions (autonomous agents)
- Multi-sig accounts that are both a contract AND have a key for
  gas sponsorship

---

## 2. Key Management

### 2.1 Supported Key Types

| Type | Value | Curve | Key Size | Signature Size | Use Case |
|------|-------|-------|----------|---------------|----------|
| None | 0 | — | 0 | 0 | Contract-only accounts |
| secp256k1 | 1 | secp256k1 | 33 or 65 bytes | 65 bytes | Standard ECDSA (MetaMask, Ledger) |
| BLS | 2 | BLS12-381 | 48 bytes | 96 bytes | Aggregate signatures, consensus |
| Multisig | 3 | — | Variable | Variable | N-of-M threshold signatures |

Default for user accounts: secp256k1 (compatible with all existing wallets).

### 2.2 Signature Verification

```
verify(tx, signature, publicKey) → bool:
    1. hash = keccak256(encode(tx))
    2. Recover public key from (hash, signature)
    3. Compare recovered key to account.key
    4. If match → valid
    5. If no match → check session keys
```

Session keys use a shorter path:

```
verifySession(tx, signature, sessionPublicKey) → bool:
    1. Check sessionPublicKey is in account's sessionKey merkle tree
    2. Check session hasn't expired
    3. Check session has permission for this action
    4. Verify signature against sessionPublicKey
```

### 2.3 Key Rotation

```
rotateKey(newKey: bytes, signature: bytes) → bool:
    1. Verify signature from CURRENT key
    2. Set account.key = newKey
    3. Emit KeyRotated(account, oldKey, newKey, block)
    4. Return true
```

Key rotation is always possible with the current key. If the current
key is lost, use recovery (Section 5).

**Stage-specific behavior:**
- Stage 0: rotation also updates the Onboarding Ledger's encrypted backup
- Stage 1: rotation standalone (user manages their own seed phrase)
- Stage 2: rotation + confirmation via hardware wallet

### 2.4 Key Compromise Protocol

If a user believes their key is compromised:

```
1. Emergency freeze (via deadman or guardian):
   freeze() → Account locked for 48 hours
   → No transfers, no contract calls
   → User can unfreeze via guardian approval or original key

2. Key rotation (during freeze):
   rotateWhileFrozen(newKey, guardianSignatures):
   → Requires guardianThreshold approvals
   → New key set, freeze lifted
   → Old key invalidated

3. Report compromised key:
   reportCompromised(key, evidence):
   → Key added to global compromised-key list
   → All accounts using that key are frozen
   → Governance reviews and takes action
```

---

## 3. Account Stages

### 3.1 Stage 0 — Onboarding

**Purpose:** First experience for new users. Safety net is active.

```
Stage 0 constraints (protocol-enforced):
    dailySendLimit:    300 USD equivalent (via oracle feed)
    maxAccountValue:   2,500 USD equivalent
    guardianThreshold: 2 (minimum)
    deadmanBlock:      optional (recommended)
    
    Cannot:
    - Deploy Class C or D contracts
    - Remove all guardians (min 2)
    - Disable recovery
```

Stage 0 accounts have encrypted key backup in the Onboarding Ledger
and rely on guardian-based recovery.

### 3.2 Stage 1 — Standard

**Purpose:** Regular user with key responsibility. Safety net optional.

```
Stage 1 constraints:
    dailySendLimit:    10,000 USD equivalent
    maxAccountValue:   Unlimited
    guardianThreshold: 0 (optional)
    
    Cannot:
    - Deploy Class D contracts (bridges, factories)
    - Class C requires Dox_Dev Level 2
```

Stage 1 accounts manage their own seed phrase. Guardian recovery
resets to Stage 0 if used (incentivizes seed phrase management).

### 3.3 Stage 2 — Self-Custody

**Purpose:** Full sovereignty. No safety net.

```
Stage 2 constraints:
    dailySendLimit:    Unlimited
    maxAccountValue:   Unlimited
    guardianThreshold: 0 (cannot set guardians)
    deadmanBlock:      optional (self-managed)
    
    Can deploy:
    - All contract classes (A, B, C, D with appropriate Dox_Dev level)
```

Stage 2 accounts have no recovery mechanism. Seed phrase loss = total
loss. No Onboarding Ledger involvement.

### 3.4 Stage Transitions

```
Stage 0 → Stage 1: Graduation
  - Pass education module (seed phrase, key management quiz)
  - Save seed phrase (confirmed via 3-word test)
  - Encrypted backup deleted from Onboarding Ledger
  - Account.key becomes the user's sole responsibility
  - Sets stageSince = current block
  - IRREVERSIBLE

Stage 1 → Stage 2: Full Self-Custody
  - Must have been Stage 1 for at least 30 days
  - Must have never used guardian recovery
  - Must pass advanced security quiz
  - Must connect hardware wallet or confirm via existing key
  - IRREVERSIBLE

Stage 2 → Stage 1: Not possible (protocol-enforced)
Stage 1 → Stage 0: Not possible (protocol-enforced)
```

**Stage downgrade via recovery:**
If a Stage 1 account uses guardian recovery (lost seed phrase), the
recovery mechanism resets the account to Stage 0:

```
1. Recovery executed via guardian approval
2. Stage set to 0
3. Limits re-imposed ($300/day, $2,500 cap)
4. New encrypted backup created in Onboarding Ledger
5. User must re-graduate to restore Stage 1
```

---

## 4. Session Keys

### 4.1 Session Key Structure

```
SessionKey {
    id: bytes32,                  // Unique identifier
    publicKey: bytes,             // 33 or 65 bytes (secp256k1)
    
    // Permissions (encoded as bitmask + values)
    permissions: uint256,         // Bitmask of allowed actions
    allowedContracts: address[],  // Only these contracts (empty = any)
    allowedSelectors: bytes4[],   // Only these function selectors
    
    // Limits
    maxSpend: uint256,            // Max per transaction
    cumulativeDaily: uint256,     // Max per day across all txs
    cumulativeSpent: uint256,     // How much spent today
    
    // Expiry
    expiryBlock: uint32,          // Auto-expires at this block
    issuedAt: uint64,             // Block when created
    revoked: bool,                // Manually revoked
}
```

Stored in account state as a merkle tree with the root at
`account.sessionKeyRoot`. Gas for session key verification: 2,000.

### 4.2 Granting a Session Key

```
grantSession(sessionPublicKey, permissions, expiryBlock):
    1. Owner signs the session key parameters
    2. Session key is added to account's session key merkle tree
    3. sessionKeyRoot updated
    4. Emit SessionKeyGranted(account, sessionId, expiryBlock)
    
    Gas: 10,000
    One-time operation (session key lasts until revoked or expired)
```

### 4.3 Using a Session Key

```
// App wants to execute a transaction on behalf of user
tx = {
    to: 0x...,
    data: 0x...,
    value: 100,
    sessionKeyId: 0x...,          // Which session key is signing
    signature: 0x...,             // Signed by the session key, not the owner key
}

// Protocol verification:
1. Look up session key from account.sessionKeyRoot
2. Verify session hasn't expired (expiryBlock > block.number)
3. Verify session isn't revoked
4. Verify permissions allow this action
5. Verify cumulativeDaily not exceeded
6. Verify maxSpend not exceeded
7. Verify signature against session public key
8. Execute transaction
9. Update cumulativeSpent

Gas: 2,000 verification + standard tx gas
```

### 4.4 Revoking a Session Key

```
revokeSession(sessionId):
    1. Owner signs revocation
    2. Session key removed from merkle tree
    3. sessionKeyRoot updated
    4. Emit SessionKeyRevoked(account, sessionId)
    
    Gas: 5,000
```

Any session key can be revoked at any time by the account owner. A
compromised session key cannot be used after revocation.

### 4.5 Stage-Specific Session Management

| Stage | Default Behavior | User Can Change |
|-------|-----------------|-----------------|
| 0 | Auto-grant per-app with 24h expiry | Cannot disable (UX requirement) |
| 1 | Manual grant only | Full control: grant, revoke, expire |
| 2 | Manual grant + multi-sig requirement | Full control + hardware wallet signing |

---

## 5. Guardian Recovery

### 5.1 Guardian Registration

Guardians are registered at account creation or added later:

```
addGuardian(guardianAddress, weight):
    1. Guardian weight: 1 (default), higher for trusted guardians
    2. Guardian added to account's guardian merkle tree
    3. guardianRoot updated
    4. Emit GuardianAdded(account, guardian, weight)
    
removeGuardian(guardianAddress):
    1. Guardian removed from merkle tree
    2. guardiantRoot updated
    3. Must still have ≥ guardianThreshold total weight
    4. Emit GuardianRemoved(account, guardian)
```

Guardian weight thresholds:

| Stage | Min Guardians | Min Total Weight | Recovery Approval Weight Needed |
|-------|--------------|------------------|-------------------------------|
| 0 | 2 | 2 | 2 |
| 1 | 0 | 0 | 2 (if guardians exist) |
| 2 | 0 | 0 | N/A (no recovery) |

### 5.2 Recovery Flow

```
requestRecovery():
    1. Requestor provides:
       - target: address (account to recover)
       - newDeviceKey: bytes (new public key for the recovered account)
       - signedStatement: bytes (signed by the recovery requestor)
    2. Recovery request created:
       - recoveryId = hash(target, block, sequence)
       - Status: PENDING
    3. All guardians notified (on-chain event emitted)

approveRecovery(recoveryId, guardianSignature):
    1. Verify caller is a registered guardian for the target
    2. Verify guardian hasn't already approved
    3. Add guardian's weight to approval total
    4. If total ≥ threshold:
       - Execute recovery immediately

executeRecovery(recoveryId):
    1. Called after enough guardian approvals
    2. Or called after recoveryTimeout blocks (auto-recovery)
    3. Recovery executed:
       a. Account.key = newDeviceKey
       b. recoverySequence++
       c. If account was Stage 1: stage = 0 (reset to onboarding)
       d. If account was Stage 0: stage stays 0
       e. Onboarding Ledger releases encrypted backup
       f. Emit RecoveryExecuted(account, newKey, block)

revokeRecovery(recoveryId):
    1. Guardian changes their mind
    2. Guardian's weight removed from approval total
    3. If total drops below threshold:
       - Recovery is pending again
    4. Can only revoke within timeout window
```

### 5.3 Guardian Stake

Guardians stake 100 native tokens when they accept. Slashing:
- Frivolous flip-flop (approve then revoke repeatedly): 25% of stake
- Wrongful approval (proven fraudulent recovery): 100% of stake
- Successful recovery: guardian earns 10 tokens + stake returned

### 5.4 Timeout Recovery

If guardians are unresponsive after `recoveryTimeout` blocks
(default: 7 days worth of blocks):

```
1. Anyone can call timeoutRecovery(recoveryId)
2. Recovery executes with current guardian approvals (even if < threshold)
3. Guardian stake is slashed 10% for non-responsiveness
4. Emit RecoveryTimeout(account)
```

This prevents dead guardian scenarios.

---

## 6. Dead Man's Switch

### 6.1 Protocol-Level Primitive

The dead man's switch is built into the account model, not a
smart contract:

```
setDeadman(heir, timeout, interval):
    deadmanBlock = block.number + timeout
    deadmanHeir = heir
    deadmanInterval = interval    // How often to check in (blocks)
    
checkin():
    // Extends the timer: the account must call this periodically
    deadmanBlock = block.number + currentTimeout
    
trigger():
    // Called by anyone after deadmanBlock has passed
    require(block.number > deadmanBlock)
    require(account.balance > 0)
    account.balance = 0
    deadmanHeir.balance += transferredAmount
    Emit DeadmanTriggered(account, deadmanHeir, amount)
```

### 6.2 How It Works

```
User sets: "If I don't check in for 30 days, transfer everything to my wife"
  → deadmanBlock = now + 30 days
  → deadmanHeir = wife's address

Normal operation:
  User makes a transaction → checkin() called automatically
  → deadmanBlock reset to now + 30 days

User disappears:
  After 30 days with no transaction:
  → Anyone can call trigger()
  → All funds transferred to heir
  → Account frozen
```

### 6.3 Stage-Specific Behavior

| Stage | Can Set Deadman | Can Clear Deadman | Default |
|-------|----------------|-------------------|---------|
| 0 | Yes | Yes (with guardian approval) | Recommended at creation |
| 1 | Yes | Yes | Optional |
| 2 | Yes | Yes (with key) | Optional |

---

## 7. Account Creation Flows

### 7.1 Onboarding (Stage 0)

```
1. User opens app → generates keypair on-device
2. User picks recovery contacts → contacts invited via app
3. Contacts accept → their addresses registered as guardians
4. Account created with:
   - Key: device-generated secp256k1
   - Stage: 0
   - Guardians: 2+ (user's contacts)
   - Key backup: encrypted and stored in Onboarding Ledger
   - No seed phrase shown to user
   - Dox_Dev badge: Level 1 auto-assigned
   - Account ready in ~30 seconds
```

### 7.2 Wallet Connect (Stage 1)

```
1. User connects MetaMask/Rabby/Keplr → wallet address imported
2. User signs a message proving ownership ("I am claiming this account")
3. Account created with:
   - Key: the wallet's public key (secp256k1, from the signature)
   - Stage: 1 (detected experienced user)
   - Guardians: none (optional, user can add later)
   - No Onboarding Ledger involvement
   - Dox_Dev badge: Level 1 (fast-track to Level 2 available)
   - User manages their own seed phrase
   - Account ready in ~10 seconds
```

### 7.3 Hardware Wallet (Stage 2)

```
1. User connects Ledger/Trezor → hardware pubkey detected
2. User signs a message proving ownership
3. Account created with:
   - Key: the hardware wallet's public key
   - Stage: 2 (detected hardware wallet)
   - Guardians: none (cannot add)
   - No Onboarding Ledger involvement
   - Dox_Dev badge: Level 2 auto-assigned (fast-track to Level 3)
   - User responsible for hardware wallet seed
   - Account ready in ~10 seconds
```

### 7.4 Contract-Only (No Key)

```
1. CREATE/CREATE2 deploys bytecode to an address
2. Account created with:
   - Key: none (keyType = 0)
   - Stage: 1 (default for contracts)
   - No guardians
   - code = deployed bytecode
   - Controlled by the contract's logic (governance, multi-sig, etc.)
```

---

## 8. Dox_Dev Integration

### 8.1 Badge → Account Link

Dox_Dev badges are soulbound ERC-721 tokens. The account model
caches the badge level for fast access:

```
// Protocol-level check (no external call needed)
function getDeployPermission(address deployer, uint8 contractClass) → bool:
    1. stage = account[deployer].stage
    2. level = account[deployer].doxDevLevel
    3. class = contractClass
    4. Return checkPermission(stage, level, class)

// Badge level is cached from the Dox_Dev contract
// Updated on: badge mint, upgrade, revocation, expiry
// Gas savings: 200 (cached) vs 10,000+ (external call)
```

### 8.2 Deploy Permission Matrix

| Stage | Dox_Dev Level | Class A | Class B | Class C | Class D |
|-------|--------------|---------|---------|---------|---------|
| 0 | 1 | ✓ | ✗ | ✗ | ✗ |
| 1 | 0 | ✓ | ✗ | ✗ | ✗ |
| 1 | 1 | ✓ | ✓ | ✗ | ✗ |
| 1 | 2 | ✓ | ✓ | ✓ | ✗ |
| 2 | 0 | ✓ | ✓ | ✗ | ✗ |
| 2 | 1 | ✓ | ✓ | ✗ | ✗ |
| 2 | 2 | ✓ | ✓ | ✓ | ✗ |
| 2 | 3 | ✓ | ✓ | ✓ | ✓ |

### 8.3 Badge Sync

The account model syncs with the Dox_Dev contract at these events:

| Event | Action |
|-------|--------|
| Badge minted | account.doxDevLevel = new level |
| Badge upgraded | account.doxDevLevel = new level |
| Badge revoked | account.doxDevLevel = 0 |
| Badge expired | account.doxDevLevel = 0 |
| Badge reissued | account.doxDevLevel = new level |
| Account created via Dox_Dev | account.doxDevBadge set at creation |

Sync happens via system-level callback (not polling).

---

## 9. Account Lifetime

### 9.1 Account States

```
                    ┌──────────────┐
                    │    Active     │
                    │ (has balance  │
                    │  or storage)  │
                    └──────┬───────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
            ▼              ▼              ▼
    ┌────────────┐ ┌────────────┐ ┌────────────┐
    │   Frozen    │ │   Pruned   │ │   Deadman   │
    │ (rent due)  │ │(30d frozen)│ │(triggered)  │
    └────────────┘ └────────────┘ └────────────┘
            │              │              │
            │              │              │
            ▼              ▼              ▼
    ┌────────────┐ ┌────────────┐ ┌────────────┐
    │  Unfrozen  │ │ Recovered  │ │  Inherited  │
    │ (rent paid)│ │(pay + rein-│ │(funds sent  │
    │            │ │  statement)│ │ to heir)    │
    └────────────┘ └────────────┘ └────────────┘
```

### 9.2 Rent Lifecycle

```
Block 1: Account created
Block 5000: Rent due (0.001/KB/block × 10KB × 5000 = 50 tokens)
Block 5000: Account has 100 tokens → rent deducted → 50 remaining
Block 10000: Rent due again → 50 tokens → 0 remaining
Block 10001: Account is FROZEN (reads ok, writes blocked)
Block 30001: Still frozen → PRUNED (state removed, hash remains)
Block 30002: User pays 50 tokens back rent + 10 reinstatement
Block 30002: State restored from last checkpoint
Block 30002: Account ACTIVE again
```

### 9.3 Account Deletion

An account is fully removed (deleted from state trie) only when:
- Balance = 0
- Storage = empty (pruned after 30 days frozen)
- Code = empty
- No guardian entries
- No session keys
- No Dox_Dev badge

At this point, the account can be recreated (recycled addresses are
safe due to nonce checks).

---

## 10. Summary

| Feature | Value |
|---------|-------|
| Account model | Unified (no EOA/contract split) |
| Key types | secp256k1, BLS, multisig, none |
| Key rotation | Always possible with current key |
| Stages | 0 (onboarding), 1 (standard), 2 (self-custody) |
| Stage transitions | One-way (0→1→2, never backwards) |
| Guardian recovery | Threshold-based, time-out fallback |
| Guardian stake | 100 tokens (slashed on misconduct) |
| Session keys | Merkle tree, 2,000 gas to verify |
| Dead man's switch | Built-in, check-in timer + heir |
| Dox_Dev sync | Badge level cached on account |
| State rent | 0.001 native / KB / block |
| Account size | ~512 bytes (before storage) |
| Account creation | 3 paths: onboarding, wallet connect, hardware |
| Deploy permissions | Dox_Dev level × stage × contract class matrix |
| Emergency freeze | 48-hour lock, guardian-unfreeze |