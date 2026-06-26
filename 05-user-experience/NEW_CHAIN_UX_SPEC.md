# UX Wall — Account & Recovery Architecture v0.1

**Design philosophy:** Not "remove key responsibility forever" but
"provide a safety net that graduates away." The chain meets users
where they are — zero crypto experience — and walks them to full
self-sovereignty at their own pace.

---

## 1. Core Architecture — Three Account Stages

Every account exists on a spectrum of self-sovereignty. The user
chooses when to advance. The chain never forces graduation.

```
                    ┌──────────────────────┐
     Stage 0 ──────►│  Onboarding Account   │ ◄── Default for new users
                    │ (Chain-assisted)      │
                    └──────────┬───────────┘
                               │ User graduates
                               ▼
                    ┌──────────────────────┐
     Stage 1 ──────►│   Standard Account    │ ◄── Has own key + safety net
                    │ (Shared custody)      │
                    └──────────┬───────────┘
                               │ User graduates
                               ▼
                    ┌──────────────────────┐
     Stage 2 ──────►│ Self-Custody Account  │ ◄── Full sovereignty
                    │ (No safety net)       │
                    └──────────────────────┘
```

| Dimension | Stage 0: Onboarding | Stage 1: Standard | Stage 2: Self-Custody |
|-----------|-------------------|-------------------|---------------------|
| Who holds the key | Protocol-managed + user | User + guardians | User only |
| Seed phrase shown? | No | Yes (encouraged to save) | Yes (required) |
| Value limit | $300 equivalent | $10,000 equivalent | Unlimited |
| Recovery | Built-in social recovery | Social recovery (opt-in) | None (unless user sets it) |
| Gas | Sponsored by dApps/protocol | User pays (abstracted) | User pays (abstracted) |
| Dox_Dev badge | Auto-assigned Level 1 | Can upgrade to Level 2 | Can upgrade to Level 3 |
| Session keys | Auto-granted per app | User-managed | User-managed |
| Transaction signing | Auto-signed (no popup) | User confirms | User confirms |
| Best for | First-time users | Regular users | Power users / developers |

---

## 2. The Onboarding Ledger (Stage 0)

### 2.1 What It Is

The Onboarding Ledger is a system-level contract (deployed at genesis)
that manages the chain's assisted accounts. It is NOT a custodial
service — the chain cannot move user funds without the user's key.
Rather, it provides recovery infrastructure and safety limits for users
who aren't ready for full self-custody.

### 2.2 Account Creation Flow

```
User downloads app
     │
     ▼
"Welcome. Choose who should help you if you lose access:"
  [X] My spouse (send them a link)
  [X] My best friend (send them a link)
  [ ] My hardware wallet (advanced)
     │
     ▼
User selects recovery contacts → app sends invite links
     │
     ▼
Contact accepts → their wallet address is registered as guardian
     │
     ▼
Onboarding account created. Account ID: @username
Seed phrase NEVER shown. User told:
  "If you lose your phone, ask your recovery contacts to help."
     │
     ▼
User can now:
  - Receive crypto (share @username or QR code)
  - Use any dApp (auto-connected, no wallet popups)
  - Send up to $300/day
  - No seed phrase, no gas fees, no signing prompts
```

**What the user experiences:**
- Opens app, enters username, picks recovery contacts
- Gets a QR code / account handle
- That's it. No seed phrase. No gas. No signing.

**What happens under the hood:**
1. A keypair is generated on-device and encrypted with the device's
   secure enclave (FaceID/TouchID/PIN)
2. The encrypted key is stored on-device AND backed up to the
   Onboarding Ledger (encrypted, protocol-managed escrow)
3. Recovery guardians are registered in the ledger
4. The account is created with Stage 0 limits
5. The user's Dox_Dev identity is initialized (Level 1)

### 2.3 The Onboarding Ledger's State

The ledger maintains:

```
onboardingAccounts[accountId] = {
    // Identity
    identityWallet: address,
    doxDevLevel: 1,

    // Custody
    stage: 0,
    encryptedKeyBackup: bytes,    // AES-256 encrypted with user's recovery passphrase
    keyBackupHash: bytes32,       // Hash of the backup (for verification)

    // Recovery
    guardians: address[],          // Recovery contacts
    guardianApprovalsRequired: 2,  // How many guardians needed to recover
    recoveryTimeout: 0,            // Blocks before recovery auto-approves

    // Limits
    dailySendLimit: 300 * 1e18,   // $300 in native token
    totalValueLimit: 2500 * 1e18, // $2500 max in account
    dailyUsed: uint256,            // Today's spend (resets per block day)

    // Graduation
    canGraduate: false,            // True after user passes graduation quiz
    graduatedAt: 0,               // Block when they graduated
}
```

### 2.4 Recovery Flow (Stage 0)

1. User loses phone / gets new device
2. Downloads app, selects "Lost access? Recover your account"
3. App sends recovery request to their chosen guardians:
   - Guardian A: "Your friend @user wants to recover their account. Approve?"
   - Guardian B: same
4. Once 2/3 guardians approve, the Onboarding Ledger:
   - Releases the encrypted key backup to the new device
   - User authenticates with their recovery passphrase
   - Key is decrypted on the new device
5. Account restored. User never sees a seed phrase.

**Guardian timeout:** If guardians don't respond within 7 days,
recovery auto-approves (preventing dead guardian scenarios). User
can set a longer or shorter timeout.

**Guardian recovery cost:** Guardians stake a small amount (100 tokens)
to approve a recovery. If they approve a fraudulent recovery (proven
via on-chain dispute), their stake is slashed. If they correctly
approve a legitimate recovery, they earn a small fee.

### 2.5 What Stage 0 Cannot Do

To prevent abuse of the assisted account model:

| Cannot Do | Why |
|-----------|-----|
| Send > $300/day | Limits blast radius of device theft |
| Hold > $2,500 total | Prevents large-target account accumulation |
| Deploy contracts | Class A data contracts allowed (no value at risk) |
| Remove all guardians | Must have at least 2 guardians |
| Disable recovery | Recovery is mandatory in Stage 0 |
| Set session keys | Not needed — they auto-expire per app |
| Sign arbitrary messages | Prevents phishing (signed messages limited to protocol ops) |

These limits are designed to be outgrown, not escaped.

---

## 3. Graduation — Stage 1 (Standard Account)

### 3.1 When to Graduate

The user chooses to graduate when:
- They understand what a seed phrase is
- They want to hold more than $1,000
- They want to deploy contracts (Dox_Dev Level 2)
- They want full control

The app periodically offers: "You're ready for more control. Want to
take full ownership of your account?"

### 3.2 Graduation Flow

```
User taps: "Take full control"
     │
     ▼
Protocol presents:
  "Your account is currently protected by your recovery contacts.
   To take full control, you'll need to:
   1. Create and save your recovery phrase
   2. Understand what happens if you lose it
   3. Confirm you're ready"
     │
     ▼
Educational module (required):
  - What is a seed phrase?
  - How to store it safely (paper, metal, never digital photo)
  - What happens if you lose it (no one can help you)
  - Quiz: 5 questions, must pass to graduate
     │
     ▼
Seed phrase generated and displayed:
  "Write this down. Do not save it digitally. This is your
   responsibility now."
     │
     ▼
User confirms seed phrase (re-enter 3 random words)
     │
     ▼
Account upgraded to Stage 1
Limits removed: $10,000 per day
Recovery guardians still available (opt-out)
Dox_Dev can upgrade to Level 2
Session keys: user-managed
Transaction signing: user confirms
```

**After graduation:**
- The encrypted key backup in the Onboarding Ledger is deleted
- The user's key is now ONLY on their device and their seed phrase
- Recovery guardians remain as a safety net (user can remove them)
- The user can now deploy contracts, hold more value, sign messages

### 3.3 Recovery (Stage 1)

If the user loses their device but HAS their seed phrase:
1. Enter seed phrase into new device
2. Account restored immediately
3. Recovery guardians not needed

If the user loses their device AND their seed phrase:
1. Same recovery flow as Stage 0 (guardian approval)
2. BUT: recovery resets account to Stage 0 (limits re-imposed)
3. User must re-graduate
4. This prevents "I lost my seed phrase but kept my limits" abuse

### 3.4 Standard Account Features

| Feature | Available? |
|---|
| Seed phrase recovery | Yes (primary) |
| Guardian recovery | Yes (backup, resets to Stage 0) |

### 3.5 Contract Deployment by Class — Stage 1 Access

Data contracts are not financial contracts. A contract that stores
a record, anchors a truth, or tracks chain of custody can't rug
anyone. Blocking them from Stage 1 would kill utility for no reason.

The protocol classifies contracts by risk:

**Class A — Data / Non-Financial (Stage 1, no Dox_Dev required)**
- Binary Journal entries (truth anchoring)
- Timestamping / notarization
- Content publishing (publishing records, copyright)
- Attestation / credential issuance
- Supply chain tracking
- Public registries (ENS-like naming)
- Voting records (non-financial ballots)
- Reputation / ratings
- Membership lists
- Proof of existence (document hashing)

These contracts cannot move value. They store data and emit events.
Deploying them is as safe as posting a message on a forum.

**Class B — Low Financial Risk (Stage 1, Dox_Dev Level 1, Trustless Lock enforced)**
- Memecoins / simple tokens (liquidity auto-locked via Trustless Lock)
- NFT collections (no royalties, no staking)
- Raffles / lotteries (prize pool locked at deploy time)
- Crowdfunding (funds held in escrow, released only on conditions)

These contracts CAN move value, but the risk is contained:
Trustless Lock prevents liquidity pulls. Escrow prevents fund grabs.
Audited templates prevent exploits.

**Class C — Medium Financial Risk (Dox_Dev Level 2)**
- DEXs / AMMs
- Lending pools
- Staking contracts (custom reward mechanics)
- Tokens with tax / reflection / rebase mechanics
- DAO treasuries
- NFT marketplaces (with escrow)

**Class D — High Financial Risk (Dox_Dev Level 3)**
- Bridges (cross-chain, any)
- Factory contracts (deploy other contracts)
- LP pairs (direct, not via template)
- Governance frameworks (protocol-level)
- Protocol upgrade contracts

**Summary:**

```
Stage 0: Cannot deploy anything
Stage 1: Class A (data) + Class B (templates w/ Trustless Lock)
Stage 2 + L2: Class C (custom financial contracts)
Stage 2 + L3: Class D (protocol-level infrastructure)
```

This ensures data-oriented contracts are the easiest to deploy,
while financial contracts require progressively more trust
verification. The barrier is proportional to the risk, not the
complexity.
| Deploy contracts | Class A (data, no Dox required) + Class B (templates w/ Trustless Lock, Dox L1) |
| Class C-D contracts | Requires Dox_Dev Level 2+ (Stage 2) |
| Session keys | Yes (user-managed) |
| Transaction signing | Yes (user confirms each) |
| Gas abstraction | Yes (dApps can sponsor) |
| Human-readable txs | Yes (protocol-level decoding) |

---

## 4. Full Self-Custody — Stage 2

### 4.1 When to Graduate

The user graduates to Stage 2 when:
- They have been in Stage 1 for at least 30 days
- They have never used guardian recovery (proven key responsibility)
- They pass an advanced security quiz
- They explicitly confirm: "If I lose my keys, my funds are gone"

### 4.2 What Changes

| Before (Stage 1) | After (Stage 2) |
|-----------------|-----------------|
| Recovery guardians available | No recovery mechanism |
| $10,000 daily limit | No limits |
| Dox_Dev Level 2 | Dox_Dev Level 3 |
| Session keys user-managed | Session keys + multi-sig capable |
| User confirms each tx | User confirms each tx |

**The user is told:**
"There is no recovery. If you lose your seed phrase, your account is
gone forever. Are you sure you want full self-custody?"

### 4.3 What Stage 2 Enables

- Deploy protocol-level contracts (factories, LPs, bridges — Dox_Dev L3)
- Multi-sig account creation for team/vault accounts
- Unrestricted value transfer
- Participation in governance as a delegate
- No daily limits

---

## 5. Veteran / Developer Path — Skip Stage 0 Entirely

Not everyone needs the training wheels. Devs, existing crypto users,
and power users should be able to connect their existing wallets and
start at Stage 1 or 2 directly.

### 5.1 Direct Entry Points

```
                  ┌──────────────────────────────────────────┐
                  │      First Interaction with Chain        │
                  └────────────────┬─────────────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │  Stage 0        │  │  Stage 1        │  │  Stage 2        │
     │  Onboarding     │  │  Standard       │  │  Self-Custody   │
     │  (newbies)      │  │  (devs/regulars) │  │  (veterans)     │
     │  Default path   │  │  Wallet connect  │  │  Hardware       │
     │  No wallet yet  │  │  MetaMask/etc   │  │  wallet / Ledger│
     └────────────────┘  └────────────────┘  └────────────────┘
```

### 5.2 Stage 1 — Direct Entry (Wallet Connect)

A dev connects their existing wallet (MetaMask, Rabby, Keplr, etc.):

```
Dev opens app → "Connect Wallet" → Chooses MetaMask
     │
     ▼
App detects: this is an EXISTING wallet with transaction history
     │
     ▼
Proposed: "You look like you know what you're doing.
  Want to skip the onboarding? You'll enter at Stage 1:
  - No daily limits up to $10,000
  - Your keys, your responsibility
  - Full Dox_Dev Level 2 after identity verification
  - Recovery guardians optional (set up later if you want)"
     │
     ▼
Dev accepts → Account created at Stage 1
  - No Onboarding Ledger involvement
  - Key is the user's existing wallet key
  - Seed phrase is THEIR seed phrase (already managed)
  - Dox_Dev Level 1 auto-assigned (upgradeable to Level 2)
  - No guardians required
  - No recovery via ledger (unless they opt in)
  - Can deploy Class A (data) contracts immediately
  - Can deploy Class B (templates: memecoins w/ Trustless Lock, NFTs) 
  - Class C-D contracts require Dox_Dev Level 2+
```

**What changes from Stage 0:**
- The Onboarding Ledger does NOT store any key backup
- The user's key is THEIR key, not protocol-managed
- No $300 daily limit (starts at $10,000)
- Can deploy contracts immediately (once Dox_Dev Level 2 verified)
- Can use their existing wallet's seed phrase for recovery
- Guardian recovery available as opt-in, not mandatory

**Dox_Dev for existing wallets:**
- Identity wallet = the connected wallet address
- Dev can link project wallets underneath via Dox_Dev's addProjectWallet()
- Verification: dev uses existing identity (GitHub, Farcaster, ENS, or
  the app's own KYC flow)
- Once verified at Level 2 or 3, deployment is authorized

### 5.3 Stage 2 — Direct Entry (Hardware Wallet / Veteran)

A veteran with a hardware wallet or deep experience enters directly:

```
Veteran connects Ledger/Trezor → "You're connecting a hardware wallet"
     │
     ▼
"Your security level is already at Stage 2. You get:
  - Unlimited transfers
  - Dox_Dev Level 3 after identity verification
  - No recovery mechanism (you manage your own keys)
  - Multi-sig capable for team accounts"
     │
     ▼
Veteran accepts → Account created at Stage 2
  - Key is the hardware wallet public key
  - No limits
  - No Onboarding Ledger interaction at all
  - Dox_Dev Level 2 auto-assigned (upgradeable to Level 3)
  - Full contract deployment access
```

**Hardware wallet detection:**
- The app detects the wallet type from the connection method
- Ledger/Trezor/GridPlus → automatically Stage 2
- Hot wallet with >50 previous transactions → Stage 1 suggestion
- Fresh new wallet → Stage 0 suggestion (with option to skip)

### 5.4 Comparing Entry Paths

| | Onboarding (Stage 0) | Wallet Connect (Stage 1) | Hardware (Stage 2) |
|---|---|---|---|
| Who | First-time users | Devs, regular crypto users | Veterans, power users |
| Entry requirement | None | Existing wallet | Hardware wallet |
| Key managed by | Device + ledger backup | User (seed phrase) | User (hardware) |
| Recovery | Mandatory (guardians) | Seed phrase (guardians opt-in) | Seed phrase only |
| Limits | $300/day | $10,000/day | Unlimited |
| Dox_Dev entry | Level 1 | Level 1 (fast-track to Level 2) | Level 2 (fast-track to Level 3) |
| Onboarding Ledger | Yes (stores encrypted backup) | No | No |
| Contract deploy | Class A data only (no value at risk) | Yes (Class A + B templates) | Yes |
| Best for | "I've never used crypto" | "I have a MetaMask" | "I use a Ledger" |

### 5.5 Dox_Dev + Existing Wallets

Dox_Dev works with ANY wallet type:

```
Wallet Type          →  Dox_Dev Identity          →  Can Deploy?
────────────────────────────────────────────────────────────────
Onboarding (Stage 0) →  @username (Level 1)       →  No
MetaMask             →  0x... (Level 1 → L2 fast) →  Yes (after L2)
Ledger               →  0x... (Level 2 fast)       →  Yes (after L3)
Multi-sig (Gnosis)   →  Multi-sig address (L3)     →  Yes
```

The Dox_Dev contract doesn't care where the wallet came from. It
checks: does this address hold a valid, active badge at the required
level? That's it. The onboarding ledger is just one way to get there.

### 5.6 Migration Between Paths

Users can migrate between paths at any time:

| From → To | How |
|-----------|-----|
| Stage 0 → Stage 1 | Export key, graduate (save seed phrase) |
| Stage 0 → Stage 2 | Export key to hardware wallet, then connect |
| Stage 1 → Stage 2 | Connect hardware wallet, transfer key |
| Stage 1 → Stage 0 | Cannot downgrade (irreversible) |
| Stage 2 → Stage 1/0 | Cannot downgrade (irreversible) |

Graduation is one-way. Once a user has proven they can handle more
responsibility, they can't go back.

This means the onboarding ledger is a true safety net, not a
dependency. Devs never touch it. Veterans never touch it. Only
newbies use it, and they graduate away.

---

## 6. The Onboarding Ledger's Technical Design

### 6.1 Account Abstraction Integration

All accounts (Stage 0, 1, 2) use a unified account abstraction model
similar to ERC-4337 but at the protocol level:

```
Account = {
    // The execution logic
    implementation: address,       // Account contract (upgradeable)
    
    // The user's key (can be rotated)
    key: PublicKey,
    
    // Recovery configuration
    recoveryModule: address,       // Points to Onboarding Ledger or custom
    
    // Session keys
    sessions: SessionKey[],         // Limited-time, limited-scope keys
    
    // Gas
    gasSponsor: address,           // Optional: who pays gas for this account
    
    // Verification (Dox_Dev)
    doxDevBadge: uint256,           // Token ID or 0
}
```

### 6.2 Key Rotation

```
rotateKey(newPublicKey, proofOfOldKey):
    - Validates signature from old key
    - Updates account's key to newPublicKey
    - In Stage 0: also updates encrypted backup in ledger
    - In Stage 1-2: seed phrase is the source of truth
```

This means a user can change devices without a seed phrase in Stage 0.

### 6.3 Session Keys

Each app the user connects to gets a session key:

```
sessionKey = {
    appId: bytes32,
    publicKey: PublicKey,
    permissions: {                  // Granular permissions
        maxSpend: uint256,          // Per-transaction limit
        cumulativeDaily: uint256,   // Daily total limit
        allowedContracts: address[], // Only these contracts
        allowedFunctions: bytes4[],  // Only these function selectors
        expiryBlock: uint32,        // Auto-expires
    },
    revoked: bool,
}
```

Session keys are granted by the user signing once. After that, the
app can execute permitted transactions without prompting the user
until the session expires or is revoked.

### 6.4 Recovery Smart Contract

The Onboarding Ledger's recovery module:

```solidity
contract OnboardingLedger {
    // Recovery requests
    struct RecoveryRequest {
        address target;          // Account being recovered
        uint256 guardiansApproved;
        mapping(address => bool) hasApproved;
        uint256 expiryBlock;
        bytes encryptedKeyBackup;
        bool executed;
    }

    // Submit recovery request
    function requestRecovery(address target) external {
        require(accounts[target].stage < 2, "Self-custody accounts don't use ledger recovery");
        RecoveryRequest storage req = recoveryRequests[target];
        req.target = target;
        req.expiryBlock = block.number + accounts[target].recoveryTimeout;
        emit RecoveryRequested(target);
    }

    // Guardian approves
    function approveRecovery(address target) external {
        require(isGuardian(target, msg.sender), "Not a guardian");
        RecoveryRequest storage req = recoveryRequests[target];
        require(!req.executed, "Already executed");
        require(!req.hasApproved[msg.sender], "Already approved");
        req.hasApproved[msg.sender] = true;
        req.guardiansApproved++;

        if (req.guardiansApproved >= accounts[target].guardianApprovalsRequired) {
            executeRecovery(target);
        }
    }

    // Timeout recovery (guardians didn't respond)
    function timeoutRecovery(address target) external {
        RecoveryRequest storage req = recoveryRequests[target];
        require(block.number >= req.expiryBlock, "Timeout not reached");
        require(!req.executed, "Already executed");
        executeRecovery(target);
    }

    // Execute the recovery — releases encrypted backup
    function executeRecovery(address target) internal {
        RecoveryRequest storage req = recoveryRequests[target];
        req.executed = true;
        emit RecoveryApproved(target, req.encryptedKeyBackup);
    }
}
```

Guardians can also revoke their approval within the timeout window
if they suspect fraud:

```solidity
function revokeRecovery(address target) external {
    RecoveryRequest storage req = recoveryRequests[target];
    require(req.hasApproved[msg.sender], "Haven't approved");
    require(block.number < req.expiryBlock, "Past timeout");
    req.hasApproved[msg.sender] = false;
    req.guardiansApproved--;
}
```

### 6.5 Guardian Incentives

Guardians stake a small amount (100 tokens) when they accept being a
guardian. This prevents:

- **Frivolous recovery flips:** Guardians who approve and then revoke
  repeatedly lose their stake
- **Malicious approvals:** If a guardian approves a recovery that was
  fraudulent (proven via on-chain dispute), they're slashed
- **Guardian collusion:** Multiple guardians from the same IP/cluster
  flagged by protocol

**Guardian rewards:**
- Each successful recovery: guardian earns 10 native tokens
- Guardian reputation tracked in Binary Journal
- High-reputation guardians can serve as protocol-managed guardians
  for users who don't have trusted contacts

---

## 7. Progressive Security Model

### 7.1 Limits by Stage

| | Stage 0 | Stage 1 | Stage 2 |
|---|---------|---------|---------|
| Max daily send | $300 | $10,000 | Unlimited |
| Max account value | $2,500 | No limit | No limit |
| Recovery available | Mandatory | Optional | None |
| Seed phrase shown | No | Yes | Required |
| Guardians required | 2 minimum | 0 minimum | 0 |
| Tx signing | Auto (session keys) | Confirm each | Confirm each |
| Contract deploy | No | Class A (data) + Class B (templates) | Class C-D (custom), Dox L3 |
| Dox_Dev max level | Level 1 | Level 2 | Level 3 |

### 7.2 Escalation on Suspicious Activity

If the protocol detects suspicious activity on an account:

| Signal | Action |
|--------|--------|
| Login from new device + location | Stage 0: notify guardians. Stage 1-2: notify via secondary channel |
| Multiple failed tx signatures | Temp freeze for 1 hour |
| Recovery request from guardian + suspicious IP | Delay recovery by 24 hours, notify all guardians |
| Large withdrawal from normally low-activity account | Freeze + guardian notification |
| Known phishing contract interaction | Block tx, warn user |

### 7.3 Emergency Freeze

A user can freeze their own account at any stage:

```
Freeze triggered → All transfers blocked for 48 hours
                  → Guardians notified
                  → User can unfreeze with key + guardian approval
                  → After 48h, account auto-unfreezes
```

This gives a user time to recover if their device is stolen, without
permanently locking their funds.

---

## 8. Integration with Other Design Principles

### 8.1 Dox_Dev

Every onboarding account automatically gets Dox_Dev Level 1:
- Identity wallet is the onboarding account
- Project wallets can be linked later
- Badge is soulbound to the identity

Graduation to Level 2 requires Stage 1+ and passing the education
module. Graduation to Level 3 requires Stage 2.

### 8.2 Gas Abstraction

In Stage 0, all gas is sponsored:
- dApps pay for their users' transactions (protocol-enforced)
- If a dApp doesn't sponsor, protocol covers up to $0.10/day per user
- User never sees a "gas fee" prompt

In Stage 1-2, gas is still abstracted — deducted from the asset being
moved (pay swap fees in USDC, not native token). User can choose to
hold native tokens for gas if they want.

### 8.3 Oracle Integration

The Onboarding Ledger uses the oracle network for:
- **Exchange rates** — Convert limits between native token and USD
  (limits are in USD, enforced via oracle price feed)
- **Location verification** — Optional: check IP geolocation for
  suspicious activity detection
- **Identity verification** — Optional: verify government ID via
  privacy-preserving oracle attestation (for Dox_Dev Level 3+
  without trusting a centralized KYC provider)

### 8.4 Binary Journal

Each account's journey is recorded in Binary Journal:
- Account created at block X
- Recovery guardians set
- Recovery events (each one recorded, time-stamped)
- Graduation events (Stage 0 → 1 → 2)
- Key rotation events

This provides an immutable account history that the user owns.
Provenance of every security decision is verifiable.

---

## 9. What the User Never Sees (The Promise)

The entire design is judged by this standard:

**A user who has never heard of blockchain should be able to:**
1. Download the app
2. Create an account with a username and 2 recovery contacts
3. Buy crypto with a credit card (built-in on-ramp)
4. Use any dApp without prompts
5. Switch phones without losing access
6. Send money to a friend by scanning a QR code
7. Hold $1,000+ by graduating (after learning about seed phrases)
8. Eventually hold unlimited value with full self-custody

**They never see:**
- A seed phrase (until they graduate)
- A gas fee popup (ever — it's always abstracted)
- A network selector (ever — it's one chain)
- A hex address (can share @username or QR code)
- A transaction hash (ever — it's shown as "confirmed" or "pending")
- A browser extension (can use a mobile app or web login)

---

## 10. Key Tradeoffs & Risks

| Risk | Mitigation |
|------|-----------|
| **Guardian collusion to steal funds** | Guardians must stake; slashing for fraudulent recovery; user chooses their own guardians |
| **Onboarding Ledger hack** | Ledger doesn't hold unencrypted keys (device-encrypted, user passphrase-protected). Hack leaks encrypted blobs only |
| **Users never graduating** | Value limits force graduation for any meaningful use. Once they want to hold >$1,000, they must graduate |
| **Guardian unavailability** | Timeout recovery after 7 days; protocol-managed guardians for users who can't find trusted contacts |
| **Social engineering of guardians** | Recovery request notifies ALL guardians; revocation window; identity verification for high-value accounts |
| **Centralization of onboarding** | Ledger is a system contract — immutable and permissionless. Anyone can build alternative onboarding UIs |
| **Key loss in Stage 2** | That's the point of Stage 2 — user has accepted this responsibility. Education module ensures informed decision |

---

## 11. Spec Summary

| Component | Stage 0 | Stage 1 | Stage 2 |
|-----------|---------|---------|---------|
| Account model | Protocol-assisted | Self-custody with safety net | Full self-custody |
| Key custody | Device + encrypted ledger backup | Seed phrase | Seed phrase |
| Recovery | Guardians (2+ approval) | Seed phrase OR guardians | Seed phrase only |
| Limits | $300/day, $2,500 total | $10,000/day | Unlimited |
| Seed phrase | Not shown | Shown + quiz | Required |
| Dox_Dev | Level 1 | Level 2 | Level 3 |
| Deploy contracts | No | Yes (templates) | Yes (protocol-level) |
| Gas | Fully sponsored | Abstracted | Abstracted |
| Session keys | Auto (per app) | User-managed | User-managed |
| Tx signing | Auto (within limits) | User confirms | User confirms |
| Guardian requirement | 2+ | 0+ | 0 |

The Onboarding Ledger is the key innovation — a system contract that
makes self-custody approachable by providing a safety net that
gradually withdraws. It's not a custodial service (doesn't hold keys),
it's a recovery infrastructure that users outgrow.