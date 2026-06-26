# Trustless Lock Templates — Protocol Spec v0.1

**Purpose:** Safe, audited contract templates for Class B deployments
**Enforcement:** Trustless Lock baked into every template that handles value
**Access:** Stage 1+ accounts with Dox_Dev Level 1+

---

## 1. Template Registry

### 1.1 System Contract

The Template Registry is a system-level contract (deployed at genesis)
that stores all audited deployment templates:

```
TemplateRegistry {
    // Template storage
    templates: mapping(bytes32 => Template),    // codeHash → Template
    templateList: bytes32[],                     // All registered template hashes
    
    // Permissions
    deployerParamDefaults: mapping(bytes32 => bytes),  // Default params per deployer
    templateUsage: mapping(bytes32 => uint256),        // How many times each template used
    
    // Admin
    curator: address,                     // Multi-sig curation council
    curatorThreshold: uint8,              // N-of-M approvals needed
    auditAuthority: address,              // External audit firm's on-chain key
}
```

### 1.2 Template Structure

```
Template {
    // Identity
    id: bytes32,                          // keccak256(name, version)
    name: string,                         // e.g., "Memecoin v1"
    version: uint8,                       // Incrementing version
    bytecodeHash: bytes32,                // keccak256 of the audited bytecode
    bytecode: bytes,                      // The full bytecode (with parameter placeholders)
    
    // Classification
    riskClass: uint8,                     // Always CLASS_B (1) for templates
    trustlessLockRequired: bool,          // Must use Trustless Lock? (true for value-moving templates)
    
    // Constraints
    parameters: Parameter[],              // User-configurable parameters with constraints
    constraints: Constraint[],            // Constraints on those parameters
    
    // Audit
    auditReport: bytes32,                 // IPFS hash of audit report
    auditDate: uint64,                    // Block when audit was completed
    auditor: address,                     // Audit firm's address
    
    // Lifecycle
    active: bool,                         // Can this template still be used?
    deprecated: bool,                     // Replaced by newer version?
    deprecationBlock: uint64,             // Block when deprecated
    replacementId: bytes32,              // Newer template to use instead
    
    // Usage
    totalDeployments: uint256,            // How many times deployed
    totalValueLocked: uint256,            // Total value locked across all deployments
    lastDeployment: uint64,               // Block of last deployment
}
```

### 1.3 Parameter System

Each template defines its user-configurable parameters with constraints:

```
Parameter {
    name: string,                         // e.g., "name", "symbol", "totalSupply"
    paramType: uint8,                     // 0=string, 1=uint256, 2=address, 3=bool
    defaultValue: bytes,                  // ABI-encoded default value
    isRequired: bool,                     // Must the user provide a value?
    
    // Constraints
    minValue: uint256,                    // For uint params
    maxValue: uint256,                    // For uint params
    allowedValues: bytes[],               // For enum-like params (empty = any)
    minLength: uint16,                    // For string params
    maxLength: uint16,                    // For string params
    mustMatchPattern: string,             // Regex for string params
}
```

---

## 2. The Full Flow: DEX + Trustless Lock

Trustless Lock is NOT a DEX. It's a safety layer that sits on top
of our native DEX (SwapRoute). The DEX handles all trading; Trustless
Lock ensures liquidity can't be pulled before conditions are met.

```
User creates memecoin
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Template: Memecoin v1                               │
│                                                     │
│  1. Deploy token contract                            │
│  2. Create Pair on SwapRoute (native DEX)            │
│     → Token + PLS → SwapRoute Pair                   │
│     → This is where the community trades             │
│  3. LP tokens from SwapRoute → Trustless Lock        │
│     → Locked atomically (never in deployer's wallet) │
│  4. Renounce ownership                               │
└─────────────────────────────────────────────────────┘
         │
         ▼
Community trades on SwapRoute ← LP liquidity locked via Trustless Lock
```

**Two separate roles:**
- **SwapRoute (native DEX):** The exchange. Handles swaps, adds
  liquidity, provides prices. Standard AMM model (constant product).
- **Trustless Lock:** The vault. Holds LP tokens safely until
  conditions are met. Cannot be bypassed.

**Why separate:**
- DEX and lock have different upgrade cycles
- DEX needs frequent optimization (routing, gas, MEV)
- Lock needs extreme simplicity (fewer functions = fewer bugs)
- Other DEXs can integrate with Trustless Lock too (not locked to
  SwapRoute)

### 2.1 What Trustless Lock Does (The Vault)

Trustless Lock ensures that liquidity deposited into a pool CANNOT be
withdrawn before the lock conditions are met — by anyone, including
the contract owner. The lock is enforced at the bytecode level, not
by trust. Three lock types are supported.

```
Lock {
    poolAddress: address,                 // The liquidity pool address
    token0: address,                      // First token (usually the native token)
    token1: address,                      // Second token (usually the paired token)
    liquidityAmount: uint256,             // Amount of LP tokens locked
    locked: bool,                         // Is the lock active?
    
    // Lock type
    lockType: uint8,                      // 0=Time, 1=Vesting, 2=Multi-sig
    
    // Type 0 — Time Lock
    lockPeriod: uint256,                  // Total lock period in blocks
    unlockBlock: uint64,                  // Block when lock expires (time only)
    
    // Type 1 — Vesting
    vestingStart: uint64,                 // Block vesting starts
    vestingEnd: uint64,                   // Block vesting ends
    vestingCliff: uint64,                 // Block cliff ends (0 = no cliff)
    totalReleased: uint256,               // How much released so far
    
    // Type 2 — Multi-sig
    guardianRoot: bytes32,                // Merkle root of guardian addresses
    guardianThreshold: uint8,             // Approvals needed
    guardianApprovals: uint8,             // Current approval count
    releaseSignatures: mapping(uint8 => bytes),  // Collected signatures
    
    // Common
    unlockRecipient: address,             // Who can claim after conditions met
}
```

**Key properties:**
- Lock is created at deploy time (before any trading can happen)
- Lock period is set in the template parameters (governance-enforced minimum)
- No function in the contract can bypass the lock (audited at bytecode level)
- After unlock, liquidity can be claimed by the unlock recipient
- The lock contract is a precompile — runs at system level for safety

### 2.2 Protocol-Enforced Requirements

For any Class B template with `trustlessLockRequired = true`:

```
1. The template MUST include a Trustless Lock at deploy time
2. The lock MUST be created before any token transfer occurs
3. The lock period MUST be ≥ governance minimum (default: 30 days)
4. The lock MUST cover ≥ governance minimum of total liquidity
   (default: 60% of raised liquidity must be locked)
5. The lock recipient MUST be a time-locked address (cannot be the deployer
   directly — must be a time-lock contract that releases to deployer after
   lock period + 7 days delay)
```

These requirements are enforced by the template bytecode itself and
verified during the audit process. They are not governance settable
per-deployment — only per-template.

### 2.3 Trustless Lock Precompile

The Trustless Lock is a precompile (`0x13`) that enforces lock rules:

```
function trustlessLock_createTimeLock(
    poolAddress: address,
    token0: address,
    token1: address,
    amount: uint256,
    lockPeriod: uint256,      // In blocks
    unlockRecipient: address
) → lockId: bytes32

function trustlessLock_createVestingLock(
    poolAddress: address,
    token0: address,
    token1: address,
    amount: uint256,
    vestingStart: uint64,
    vestingEnd: uint64,
    vestingCliff: uint64,      // 0 = no cliff
    unlockRecipient: address
) → lockId: bytes32

function trustlessLock_createMultiSigLock(
    poolAddress: address,
    token0: address,
    token1: address,
    amount: uint256,
    guardians: address[],
    threshold: uint8,
    unlockRecipient: address
) → lockId: bytes32

function trustlessLock_getLock(
    lockId: bytes32
) → Lock memory

function trustlessLock_releasableAmount(
    lockId: bytes32
) → uint256  // How much CAN be released right now

function trustlessLock_release(
    lockId: bytes32
) → uint256  // Release releasable tokens to recipient

function trustlessLock_extend(
    lockId: bytes32,
    additionalBlocks: uint256
) → bool    // Lock can only be extended, never shortened
```

Gas: 5,000 to create, 500 to query, 3,000 to release.

---

## 3. Memecoin Template

### 3.1 Template Details

| Field | Value |
|-------|-------|
| Name | "Memecoin v1" |
| Class | B |
| Trustless Lock | Required (enforced) |
| Audit | Pre-audited (bytecode hash verified) |

### 3.2 Parameters

```
Parameters the user sets:
  - name: string (2-32 chars, letters/numbers/spaces)
  - symbol: string (2-8 chars, uppercase letters)
  - totalSupply: uint256 (1,000,000 - 1,000,000,000,000)
  - liquidityPercent: uint8 (10-100% of totalSupply to pair)
  
  - pairedNativeAmount: uint256 (amount of native token to add as liquidity)
    → The user must send this amount with the deploy transaction
    → It's paired with liquidityPercent of the token supply
    → Example: 75% of supply + 10,000 native = initial LP pool

  - lockType: uint8 (0=Time, 1=Vesting, 2=Multi-sig)
  - lockPeriod: uint256 (minimum 30 days, max 730 days) [time-based]
    OR
  - vestingStartBlock: uint64 [vesting]
  - vestingEndBlock: uint64 [vesting]
  - vestingCliff: uint64 (blocks before any release) [vesting]
    OR
  - lockGuardians: address[] (2-5 addresses) [multi-sig]
  - lockThreshold: uint8 (approvals needed to release) [multi-sig]
  
  - pairedToken: address (native token by default, or governance-approved)
  - logoURI: string (optional, IPFS hash of logo)

Fixed (not user-configurable):
  - No taxes/buy/sell fees
  - No reflection/redistribution
  - No mint function
  - No blacklist
  - No pausing
  - Trustless Lock unbypassable
  - Ownership renounced at deploy
```

### 3.3 Deploy Flow — Atomic

ALL of this happens in a SINGLE transaction. The user signs once.
There is NEVER a moment where the liquidity exists unlocked.

```
1. User connects wallet (Stage 1+, Dox_Dev Level 1+)
2. User selects "Memecoin v1" template
3. User fills in parameters:
   - Name: "My Dog Coin"
   - Symbol: "WOOF"
   - Supply: 1,000,000,000
   - Liquidity: 75% of supply to pair
   - Native to pair: 10,000 PLS
   - Lock type: Vesting
   - Vesting period: 180 days
   - Cliff: 30 days (no release in first 30 days)
4. User sends ONE transaction:

   ┌─────────────────────────────────────────────────────┐
   │  ATOMIC TRANSACTION                                  │
   │  User sends: 10,000 PLS (pairedNativeAmount)         │
   │  Included in the same tx as the deploy call          │
   │                                                     │
   │  1. Deploy token contract                            │
   │     → Token created, 1B supply minted to deployer    │
   │                                                     │
   │  2. Create liquidity pair on SwapRoute (native DEX)       │
   │     → 750M WOOF tokens transferred to pair                 │
   │     → User's 10,000 PLS transferred to pair                │
   │     → SwapRoute LP tokens minted                           │
   │     → LP tokens forwarded immediately to Trustless Lock    │
   │                                                     │
   │  3. Create Trustless Lock (type: vesting)            │
   │     → LP tokens locked immediately (precompile 0x13) │
   │     → Lock type: Vesting                             │
   │     → Period: 180 days (0.55%/day linear release)    │
   │     → Cliff: 30 days (nothing released first 30 days)│
   │     → Lock recipient: deployer's address             │
   │     → LP tokens NEVER sit in deployer's wallet       │
   │       (forwarded at mint → locked atomically)        │
   │                                                     │
   │  4. Renounce ownership                               │
   │     → Owner = address(0)                             │
   │     → No one can change the contract ever            │
   │                                                     │
   │  5. Verify state                                     │
   │     → Pair created at expected address: check        │
   │     → Lock active with correct params: check         │
   │     → Ownership renounced: check                     │
   │     → Only then: transaction succeeds                │
   └─────────────────────────────────────────────────────┘

5. Transaction confirmed →
   User receives: "Your coin is live!
     750M WOOF / 10,000 PLS locked in vesting
     0.55% releases per day starting day 31
     No early withdrawal possible. Enforced by protocol."
```

**Critical invariant:** Step 2 and Step 3 happen in the same atomic
execution. The LP tokens go from the DEX pair contract → into the
Trustless Lock — they never pass through the deployer's wallet.
There is zero blocks of time where the deployer could withdraw.

**If any step fails:**
- Token deploy fails → everything reverts
- Pair creation fails → token deploy reverts
- Lock creation fails → pair creation reverts
- Ownership renounce fails → lock creation reverts
- Verification check fails → everything reverts

The user pays gas ONCE, regardless of which step fails.

### 3.4 What the User CANNOT Do (Bytecode Enforced)

```
✗ Renounce ownership?        Already renounced at deploy
✗ Add taxes?                  No tax functions exist
✗ Mint more tokens?           No mint function exists
✗ Blacklist addresses?        No blacklist exists
✗ Pause trading?              No pause function exists
✗ Withdraw liquidity early?   Locked by Trustless Lock (protocol-enforced)
✗ Upgrade contract?           No upgrade mechanism
✗ Drain via backdoor?         Bytecode has been audited, no backdoors
```

### 3.5 Trustless Lock Types

Users choose their lock type at creation. All locks are enforced at
the bytecode level — no functions exist to bypass them.

**Type 0 — Time Lock (default)**
LP tokens are completely frozen for the full period. After the period
expires, the recipient can withdraw everything at once.

```
Lock type: Time
Params:   period = 180 days
Behavior: Blocks 0-180: locked. Block 181+: fully released.
Best for: Simple memecoins, fair launches
Risk:     If the project fails, deployer can't exit early
```

**Type 1 — Vesting Lock**
LP tokens release gradually on a linear schedule. A cliff prevents
any release in the first N days.

```
Lock type: Vesting
Params:   totalPeriod = 365 days, cliff = 30 days
Behavior: Blocks 0-30:  0% released (cliff)
          Blocks 31-365: ~0.3% per day releases linearly
          Block 366+:    100% released
Best for: Projects with ongoing development
Safety:   Even after release starts, selling 0.3% daily can't crash
          the pool — prevents "unlock and dump"
```

**Type 2 — Multi-Sig Lock**
Lock releases only when N-of-M guardian addresses sign a release
transaction. No automatic release.

```
Lock type: Multi-sig
Params:   guardians = [0xA, 0xB, 0xC, 0xD, 0xE], threshold = 3/5
Behavior: No automatic release at any block.
          Requires 3 of 5 guardians to sign a release tx.
          Guardians can be: team members, community reps, advisors.
Best for: DAO treasuries, team tokens, project funds
Safety:   No single person can rug. Team collusion requires 3/5+
```

**Comparison:**

| Property | Time Lock | Vesting | Multi-sig |
|----------|-----------|---------|-----------|
| Automatic release? | Yes (at expiry) | Yes (gradual) | No (requires signatures) |
| Can accelerate? | No | No | Yes (if guardians agree) |
| Can extend? | Yes (by deployer) | Yes (by deployer) | Yes (by guardians) |
| Can revoke? | No | No | No |
| Best for | Fair launches | Ongoing projects | Team/DAO funds |
| Governance min | 30 days | 30 days total, 7 day cliff | N/A (guardian-set) |
| Unique risk | Dump at expiry | Gradual sell pressure | Guardian collusion |

```
✓ Change logo/description (via IPFS + setter)
✓ Airdrop tokens (if supply remaining after liquidity pool)
✓ Burn their own tokens (standard ERC-20 burn)
✓ Transfer their unlocked tokens (liquidity is locked, their personal bag isn't)
✓ Promote the coin (marketing)
✓ Add more liquidity (can always add, just can't remove before unlock)
```

---

## 4. NFT Collection Template

### 4.1 Template Details

| Field | Value |
|-------|-------|
| Name | "NFT Collection v1" |
| Class | B |
| Trustless Lock | Not required (NFTs don't hold liquidity pools) |
| Audit | Pre-audited (bytecode hash verified) |

### 4.2 Parameters

```
Parameters the user sets:
  - name: string (2-32 chars)
  - symbol: string (2-8 chars)
  - maxSupply: uint256 (1 - 10,000)
  - baseURI: string (IPFS or HTTPS URI)
  - mintPrice: uint256 (in native tokens)
  - mintStart: uint64 (block number, can be "immediate")
  - royalties: uint16 (0-1000 basis points, 0-10%)
  - royaltyRecipient: address

Fixed (not user-configurable):
  - No staking mechanics
  - No breeding/evolution
  - Simple mint → transfer flow
  - Royalties via ERC-2981 standard
  - Metadata update restricted to owner
  - Contract ownership transferable
```

### 4.3 Deploy Flow

```
1. User connects wallet (Stage 1+, Dox_Dev Level 1+)
2. User selects "NFT Collection v1" template
3. User fills in parameters:
   - Name: "Pixel Punks"
   - Symbol: "PP"
   - Max supply: 10,000
   - Base URI: ipfs://...
   - Mint price: 10 native
   - Royalties: 5%
4. Contract deployed with parameters
5. User uploads metadata + images to IPFS
6. Minting is live
```

---

## 5. Crowdfunding Template

### 5.1 Template Details

| Field | Value |
|-------|-------|
| Name | "Crowdfunding v1" |
| Class | B |
| Trustless Lock | Conditional (if liquidity pool is created) |
| Audit | Pre-audited |

### 5.2 Parameters

```
Parameters the user sets:
  - projectName: string
  - fundingGoal: uint256 (minimum to raise)
  - maxContribution: uint256 (per wallet)
  - deadline: uint64 (block)
  - benefitTiers: Tier[] (contribution levels → rewards)

Fixed:
  - Funds held in escrow until goal met
  - If goal not met by deadline: all contributors can withdraw
  - If goal met: funds released to project creator after 7-day dispute window
  - Project creator must be Dox_Dev verified (Level 1+)
  - No partial releases (all or nothing)
```

---

## 6. Template Lifecycle

### 6.1 Adding a New Template

```
1. Developer submits template for audit:
   - Source code + compiled bytecode
   - Parameter definitions with constraints
   - Trustless Lock integration verification
   
2. Audit firm (governance-approved) reviews:
   - Bytecode matches source code (verification)
   - No backdoors or hidden functions
   - Trustless Lock correctly enforced
   - Parameters are properly constrained
   - No way to bypass the lock
   
3. On successful audit:
   - Audit report published (IPFS)
   - Template registered in TemplateRegistry
   - bytecode hash + constraints stored
   - Template is active

4. Users can now deploy from the template
```

### 6.2 Deprecating a Template

```
1. Newer version available or vulnerability found
2. Governance votes to deprecate
3. Template.active = false
4. Template.deprecated = true
5. If replacement exists: template.replacementId set
6. Existing deployments are NOT affected (immutable contracts)
7. Users cannot deploy new instances of deprecated template
8. Users are directed to the replacement template
```

### 6.3 Emergency Pause

If a vulnerability is found in a template:

```
1. Security council votes (5/7 multi-sig)
2. Template paused (cannot deploy new instances)
3. Governance notified
4. Existing deployments evaluated:
   - If funds at risk: security council can freeze + migrate
   - If no funds at risk: template deprecated, replacement created
5. Full governance review within 7 days
```

---

## 7. Template Constraints Governance

### 7.1 Governable Parameters

| Parameter | Default | Range | Change Delay |
|-----------|---------|-------|-------------|
| Min lock period | 30 days | 7-365 days | 7 days |
| Min liquidity locked | 60% | 25-100% | 7 days |
| Max total supply (memecoin) | 1T | 100M-1Q | 14 days |
| Min lock period (memecoin) | 30 days | 7-365 days | 7 days |
| Min lock period (crowdfunding) | 7 days | 0-30 days | 3 days |
| Max royalty % (NFT) | 10% | 5-25% | 14 days |
| Audit firms | Whitelist | Add/remove | 14 days |

### 7.2 Per-Template Constraints

Individual templates can have stricter constraints than the defaults:
- Memecoin v1 might require 60% liquidity locked (the model enforces this)
- A future "Premium Memecoin v2" might allow 80% with 90-day min lock
- Templates compete on safety features, not just features

---

## 8. Deployment Cost Breakdown

| Template | User Gas Cost | Protocol Fee | Estimated Total |
|----------|-------------|--------------|-----------------|
| Memecoin v1 | 150,000 | 10 native | ~$0.50-2.00 |
| NFT Collection v1 | 120,000 | 10 native | ~$0.40-1.50 |
| Crowdfunding v1 | 100,000 | 10 native | ~$0.30-1.00 |
| Custom (Class C+) | 200,000+ | 50-500 native | Variable |

**Protocol fee distribution:**
- 50% to the template author (creator of the template)
- 30% to the audit firm
- 20% to protocol treasury

---

## 9. Summary

| Feature | Value |
|---------|-------|
| Template registry | System contract at genesis |
| Initial templates | Memecoin, NFT Collection, Crowdfunding |
| Trustless Lock | Precompile 0x13, enforced at bytecode level |
| Min lock period | 30 days (governance-set, adjustable) |
| Min liquidity locked | 60% of raised liquidity |
| Ownership | Renounced at deploy (cannot be changed) |
| No taxes/fees | Enforced by bytecode (no tax functions exist) |
| No mint | Enforced by bytecode (no mint function exists) |
| Template audit | Required before registration |
| Governance override | Security council can pause templates |
| Protocol fee | 10 native tokens per deployment |
| Template author reward | 50% of protocol fee |
| User requirement | Stage 1+, Dox_Dev Level 1+ |