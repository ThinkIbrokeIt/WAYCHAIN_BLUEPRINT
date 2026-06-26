# Dox_Dev Integration — Protocol Spec v0.1

**Contract:** DeveloperVerificationBadge (existing, 38/38 tests)
**Integration points:** Account model, EVM deploy gate, Onboarding Ledger
**Levels:** 1 (basic), 2 (verified), 3 (trusted)

---

## 1. Architecture

### 1.1 Components

```
                        ┌──────────────────────┐
                        │   Dox_Dev Badge       │
                        │   (ERC-721 soulbound) │
                        │   Existing contract   │
                        └──────────┬───────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │  Account Model  │  │   EVM Deploy   │  │  Onboarding     │
     │  (badge level   │  │   Gate (checks │  │  Ledger (auto-  │
     │   cached per    │  │   stage × dox  │  │  assigns L1 to  │
     │   account)      │  │   level per    │  │  new accounts)  │
     └────────────────┘  │   contract     │  └────────────────┘
                          │   class)       │
                          └────────────────┘
```

### 1.2 Badge Lifecycle

```
                    ┌──────────────┐
                    │  Badge Mint  │
                    │  (admin or   │
                    │  auto-assign)│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │  Level 1   │ │  Level 2   │ │  Level 3   │
     │ (onboarded)│ │ (verified) │ │ (trusted)  │
     └────────────┘ └────────────┘ └────────────┘
           │              │              │
           │              │              │
           ▼              ▼              ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │  Upgrade   │ │  Upgrade   │ │  Expire /  │
     │  to L2     │ │  to L3     │ │  Revoke    │
     └────────────┘ └────────────┘ └────────────┘
                                      │
                                      ▼
                               ┌────────────┐
                               │  Badge End │
                               │(level = 0, │
                               │can't deploy│
                               └────────────┘
```

---

## 2. Account Model Integration

### 2.1 Cached Badge State

Every account has Dox_Dev fields cached for fast protocol-level checks:

```
Account {
    ...
    doxDevBadge: uint256,    // Token ID of Dox_Dev badge (0 = none)
    doxDevLevel: uint8,      // 0-3, cached from badge contract
    doxDevExpiry: uint64,    // Block when badge expires
    doxDevIdentity: address, // The identity wallet (may differ from account)
    doxDevUpdated: uint64,   // Last sync block
    ...
}
```

**Sync mechanism:** When the badge contract emits a state change event,
a system-level callback updates the cached fields:

```
BadgeMinted(identity, tokenId, level) →
    accounts[identity].doxDevBadge = tokenId
    accounts[identity].doxDevLevel = level
    accounts[identity].doxDevUpdated = block.number

BadgeRevoked(identity, tokenId) →
    accounts[identity].doxDevLevel = 0
    accounts[identity].doxDevUpdated = block.number

BadgeLevelUpgraded(identity, tokenId, newLevel) →
    accounts[identity].doxDevLevel = newLevel
    accounts[identity].doxDevUpdated = block.number
```

**Gas savings:** Protocol-level read = 200 (SLOAD cached). External
contract call to read badge level = 10,000+ (CALL opcode).

### 2.2 Identity vs Project Wallets

Dox_Dev has two wallet types:

```
Identity Wallet (holds the badge):
  - The "resume" address
  - Linked to the user's real identity (encrypted)
  - Has doxDevLevel ≥ 1
  - Can upgrade levels, link project wallets, reissue badge

Project Wallet (operational):
  - Linked to an identity wallet via addProjectWallet()
  - Does NOT hold a badge (reuses identity's level for deploy checks)
  - Can be added/removed by identity owner
  - Deploy permissions = identity's Dox_Dev level
```

**Protocol-level check for deploy permissions:**

```
function getDoxLevel(address deployer) → uint8:
    // Check if deployer IS an identity with a badge
    if account[deployer].doxDevLevel > 0:
        return account[deployer].doxDevLevel
    
    // Check if deployer is a PROJECT wallet linked to an identity
    identity = Dox_Dev.getIdentity(deployer)
    if identity != address(0):
        return account[identity].doxDevLevel
    
    // No badge at all
    return 0
```

This means project wallets inherit their identity's deploy permissions
transparently — no extra setup needed once linked.

### 2.3 Account Creation ↔ Badge Assignment

| Creation Path | Dox_Dev Level | How Assigned |
|--------------|--------------|--------------|
| Onboarding (Stage 0) | 1 | Auto-minted by Onboarding Ledger at account creation |
| Wallet connect (Stage 1) | 1 | Auto-assigned, fast-track to Level 2 available |
| Hardware wallet (Stage 2) | 2 | Auto-assigned (hardware wallet = proven responsibility) |
| Contract-only (no key) | 0 | Not applicable (contracts don't need badges) |

**Auto-mint flow (Stage 0):**

```
Onboarding Ledger creates account →
  Ledger calls Dox_Dev.issueVerificationBadge(identity, 1, 0)
  → Badge minted to identity wallet
  → Account model syncs: doxDevLevel = 1
  → Account ready with basic deploy permissions
```

**Fast-track flow (Stage 1 wallet connect):**

```
Wallet connects → existing transaction history detected →
  Account created at Stage 1 →
  Dox_Dev Level 1 auto-assigned →
  User can optionally fast-track to Level 2:
    - Identity verification (GitHub, Farcaster, ENS, or app KYC)
    - Once verified: badge upgraded to Level 2
    - Can now deploy Class C contracts
```

---

## 3. EVM Deploy Gate Integration

### 3.1 CREATE / CREATE2 Override

When CREATE or CREATE2 is called, the EVM performs an additional check
before allowing the deploy:

```
function checkDeployPermission(deployer, bytecode) → bool:
    1. class = classifyContract(bytecode)   // 0=A, 1=B, 2=C, 3=D
    2. stage = account[deployer].stage      // 0, 1, or 2
    3. level = getDoxLevel(deployer)        // 0-3 (checks identity + project wallets)
    
    4. Return checkClassPermission(stage, level, class)

function checkClassPermission(stage, level, class) → bool:
    // Class A (data): Stage ≥ 1, any level (including 0)
    if class == 0: return stage >= 1
    
    // Class B (templates): Stage ≥ 1, level ≥ 1, must use audited template
    if class == 1: return stage >= 1 && level >= 1 && isAuditedTemplate(bytecode)
    
    // Class C (medium risk): Level ≥ 2
    // Stage 2 accounts get a pass (hardware wallet = trusted)
    if class == 2: return (stage >= 2) || (stage >= 1 && level >= 2)
    
    // Class D (high risk): Stage 2 AND Level 3
    if class == 3: return stage >= 2 && level >= 3
    
    // Unclassified: treat as Class C (safe default)
    if class == 4: return (stage >= 2) || (stage >= 1 && level >= 2)
    
    return false
```

If the check fails, the CREATE/CREATE2 reverts with a clear error:

```
revert("Dox_Dev: Insufficient verification level. 
       Required: Level 2 for Class C contracts. 
       Current: Level 1. 
       Upgrade your badge to deploy this contract.")
```

### 3.2 Bytecode Classification at Deploy Time

When CREATE/CREATE2 is called, the EVM classifies the bytecode:

```
function classifyContract(bytes code) → uint8:
    // Deploy-time analysis: scan for patterns
    
    // Class A indicators (data only)
    if hasNoPayableFunctions(code) && hasNoValueTransfers(code):
        return 0  // Class A
    
    // Class B indicators (template match)
    if isKnownTemplate(code):
        return 1  // Class B
    
    // Class D indicators (high risk)
    if containsCreateOpcode(code):       // Factory contracts
        return 3  // Class D
    if containsSelfdestruct(code):       // Can destroy itself
        return 3  // Class D (but SELFDESTRUCT disabled, so this is legacy)
    if hasBridgePatterns(code):          // Cross-chain bridge logic
        return 3  // Class D
    
    // Class C default (medium risk)
    return 2  // Class C
```

Classification is deterministic (same bytecode = same class every time).
The analysis runs in ~500 gas (fast bytecode scan, no execution).

**Governance override:** Contracts can be manually reclassified:

```
reclassifyContract(address, newClass):
    // Emergency: 1 block delay (security council)
    // Standard: 7 day delay (governance vote)
    Emit ContractReclassified(address, oldClass, newClass)
```

### 3.3 Template Verification

For Class B contracts, the deploy gate checks the bytecode against
the template registry:

```
function isAuditedTemplate(bytes code) → bool:
    codeHash = keccak256(code)
    template = TemplateRegistry.getByCodeHash(codeHash)
    return template != address(0) && template.audited
```

Parameters are verified separately:

```
function verifyTemplateParameters(code, params) → bool:
    template = TemplateRegistry.getByCodeHash(keccak256(code))
    
    // Check each parameter against the template's constraints
    for param in template.params:
        if param.name == "totalSupply":
            require(params.totalSupply <= template.maxSupply)
        if param.name == "lockPeriod":
            require(params.lockPeriod >= template.minLockPeriod)
        // ... etc
    
    return true
```

---

## 4. Badge Verification Flows

### 4.1 Standard Flow (Contract Calls `isVerified`)

Any contract can call Dox_Dev's `isVerified(address)` to check:

```solidity
// In a DEX: "only verified deployers can create pools"
function createPool(address tokenA, address tokenB) external {
    require(
        Dox_Dev.isVerified(msg.sender),
        "Only verified developers can create pools"
    );
    // ... create pool
}
```

This works for both identity wallets and project wallets (the badge
contract handles the delegation transparently).

### 4.2 Protocol-Level Deployment Check (CREATE)

For contract deployment, the EVM uses the cached account state (200 gas)
instead of an external call (10,000+ gas). This is a system-level
check — contracts cannot bypass it.

### 4.3 Wallet-Level Display

Wallets and explorers show the Dox_Dev level:

```
Address: 0x... (Dox_Dev Level 2 ✓)
├── Identity: 0x... (Level 2, verified developer)
├── Project wallets: [0x..., 0x...]
└── Badge expires: Block 10,000,000 (never, Level 2)
```

Non-verified addresses show:

```
Address: 0x...
└── Not verified (no Dox_Dev badge)
```

---

## 5. Badge Levels Detail

### 5.1 Level 1 — Basic

```
Requirements:
  - Auto-assigned to all new accounts
  - Stage 0 or Stage 1 account
  - No identity verification needed

Privileges:
  - Deploy Class A contracts (data)
  - Deploy Class B contracts (via templates w/ Trustless Lock)
  - Basic wallet-to-wallet transfers within limits

Duration: No expiration (active until revocation or upgrade)
Cost: Free (included in account creation)
```

### 5.2 Level 2 — Verified

```
Requirements:
  - Stage 1 account (or Stage 2 hardware wallet)
  - Identity verification (one of):
    a) GitHub account with 90+ days history
    b) Farcaster/ENS with verified socials
    c) App-based KYC (government ID, privacy-preserving)
    d) Existing on-chain reputation (>100 txns, >1yr)
  - Application reviewed by Dox_Dev curators

Privileges:
  - Deploy Class C contracts (DEXs, lending, staking)
  - Unrestricted template deployment
  - Can become a validator (if otherwise qualified)

Duration: 1 year renewable
Cost: 100 native tokens (one-time fee)
```

### 5.3 Level 3 — Trusted

```
Requirements:
  - Stage 2 account (hardware wallet or equivalent)
  - Level 2 held for at least 3 months
  - No slashing or revocation history
  - Multi-sig or hardware wallet signing
  - Community endorsement (5 existing Level 3 holders vouch)
  - Final approval by Dox_Dev governance

Privileges:
  - Deploy Class D contracts (bridges, factories, protocols)
  - Become a Dox_Dev curator (approve new Level 2 applications)
  - Serve on security council
  - Unrestricted deployment across all classes

Duration: 2 years renewable
Cost: 500 native tokens (one-time fee)
```

### 5.4 Level Progression

```
L1 → L2:
  User submits identity verification
  → Curators review (automated + manual for edge cases)
  → If approved: upgradeBadge(identity, 2)
  → Account model syncs: doxDevLevel = 2
  → Typical: 1-24 hours for approval

L2 → L3:
  User has held L2 for 3+ months
  → Gathers 5 endorsements from existing L3 holders
  → Submits governance review
  → If approved: upgradeBadge(identity, 3)
  → Account model syncs: doxDevLevel = 3
  → Typical: 1-7 days for approval
```

---

## 6. Integration Points Summary

| Integration Point | What It Does | Gas Impact |
|-----------------|-------------|------------|
| Account model cache | Stores doxDevLevel for fast access | +512 bytes per account |
| EVM CREATE/CREATE2 | Checks stage × level × class | +2,000 gas per deploy |
| Badge event sync | Updates cache on mint/upgrade/revoke | System callback (free) |
| Project wallet check | Delegates identity's level to project wallet | +200 gas (SLOAD) |
| Template registry | Verifies bytecode hash against audited templates | +500 gas per template check |
| Bytecode classifier | Scans bytecode at deploy time for class | ~500 gas |
| Wallet connect flow | Auto-assigns L1 + fast-track prompt | One-time cost |
| Onboarding ledger flow | Auto-mints L1 at account creation | Integration via system call |
| Governance override | Reclassifies contracts (emergency + standard) | Event-only |

All contract deploy checks are enforced at the EVM level — they cannot
be bypassed at the smart contract level, only overridden by governance.