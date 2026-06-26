# User Flow Scenario — "PepePulse" Memecoin Creation

**Alice** has 10,000 PLS in her wallet. She wants to create a memecoin
called "PepePulse" (PEPE) and have a liquid market for people to trade
it — with total confidence she can't rug pull.

---

## Step 0: Alice's Starting State

```
Alice's Wallet:
  ┌──────────────────┐
  │ 10,000 PLS        │  ← Her native tokens to pair with
  │ Stage 1 Account   │
  │ Dox_Dev Level 1   │
  └──────────────────┘

No PEPE exists yet. No pair exists yet.
```

---

## Step 1: Alice Opens the Memecoin Template

She navigates to the template page:
- Selects "Memecoin v1"
- Fills in:
  - Name: "PepePulse"
  - Symbol: "PEPE"
  - Supply: 1,000,000,000 PEPE
  - Liquidity to pair: 75% → 750,000,000 PEPE
  - Native to pair: 10,000 PLS
  - Lock type: Time Lock
  - Lock period: 180 days

She clicks **"Deploy + Lock"**.

---

## Step 2: The Atomic Transaction (Alice Signs Once)

Her wallet asks: "Send 10,000 PLS + deploy contract?"

She confirms. One transaction fires. Inside that transaction, the
**template contract** acts as the orchestrator — it does every step.

---

### Step 2a — Template Contract Deploys the PEPE Token

The template contract (NOT Alice) creates the PEPE token using CREATE:

```
Template Contract calls CREATE → PEPE contract at 0x...
  1,000,000,000 PEPE minted to Alice
  Owner = Alice (for now — will be renounced later)

Chain state after:
  PEPE contract at 0x... exists
  Alice has 1,000,000,000 PEPE
  Template contract has 0 PEPE
```

---

### Step 2b — Template Contract Calls SwapRoute

Alice approved the template contract to move her PLS (she sent it
as msg.value). Now the template contract:

1. **Transfers 750,000,000 PEPE from Alice to itself**
   (The template was pre-approved or this is done atomically at deploy)

2. **Calls SwapRoute's addLiquidity():**
```
Template Contract → SwapRoute.addLiquidity(
    tokenA:  PEPE (address 0x...),
    tokenB:  PLS  (native token),
    amountA: 750,000,000,
    amountB: 10,000,
    recipient: Template Contract ← KEY: LP tokens go HERE, not Alice
)
```

SwapRoute creates pair 0xPEPE_PLS, mints LP tokens, and sends them to
the TEMPLATE CONTRACT (the `recipient` parameter):

```
After addLiquidity:
  SwapRoute pair 0xPEPE_PLS:
    Reserves: 750M PEPE + 10,000 PLS  ← Tradeable by community

  Template Contract now holds:
    1,000 LP tokens  ← NOT Alice. NOT in Alice's wallet.
```

---

### Step 2c — Template Contract Locks LP Tokens in Trustless Lock

The template contract still has the LP tokens. Immediately — same
execution — it locks them:

```
Template Contract → Trustless Lock.createTimeLock(
    poolAddress:  0xPEPE_PLS,
    token0:       PEPE,
    token1:       PLS,
    amount:       1,000 LP tokens,
    lockPeriod:   180 days,
    unlockRecipient: Alice  ← Alice can claim AFTER lock expires
)
```

Trustless Lock creates Lock #1 holding the 1,000 LP tokens.

```
Template Contract now holds: 0 LP tokens (all locked)
Trustless Lock now holds:    1,000 LP tokens (locked 180 days)
```

---

### Step 2d — Template Contract Renounces Token Ownership

The template contract still controls the PEPE token (or Alice does,
depending on deploy setup). To fully renounce:

```
PEPE Contract:
  Owner before: Alice (set at deploy)
  Owner after:  address(0) — renounced by Alice's signature
                (included in the deploy tx authorization)
```

Or more elegantly: the template contract calls `renounceOwnership()`
on the PEPE token as part of the atomic flow.

**Now no one controls the token. Ever.**

---

### Step 2e — Verification Check (Protocol Level)

```
[✓] PEPE bytecode matches audited template
[✓] SwapRoute pair created, reserves match
[✓] Trustless Lock #1 holds 1,000 LP tokens
[✓] Lock params: 180 days, recipient Alice
[✓] PEPE ownership: address(0)
```

All pass → transaction succeeds.

---

## The Critical Difference

**Before (wrong flow):**

```
Alice creates token → Alice adds liquidity on DEX
                     → LP goes to Alice's wallet
                     → Alice must separately lock LP
                     → Window exists where Alice could rug
```

**After (correct flow):**

```
Template contract creates token
Template contract adds liquidity on SwapRoute (user chooses DEX)
LP goes to template contract (never touches Alice)
Template contract locks LP in Trustless Lock
Template contract renounces token ownership
All in ONE atomic execution. No window.
```

**The DEX is a parameter the user chooses.** The template contract
accepts a DEX address. At deploy time, the user can select:
- SwapRoute (our native DEX, default)
- Any compatible DEX on the chain (must implement standard `addLiquidity`)
- Future: cross-chain DEXs

The template contract is pre-audited to work with specific DEX
interfaces. Unsupported DEXs are rejected at the bytecode level.

---

## Step 3: Alice Sees Result

```
Transaction confirmed in 1 second.

Your coin is live!
  PEPE/PPLS pair at: 0xPEPE_PLS
  Initial price: ~75,000 PEPE per 1 PLS
  Liquidity locked: 180 days (no early withdrawal)
  Pool reserves: 750M PEPE + 10,000 PLS

Your wallet:
  250,000,000 PEPE (25% — your personal bag)
  0 PLS (all 10,000 used for liquidity)
```

---

## Step 4: Community Trades on SwapRoute

Bob sends 100 PLS to buy PEPE:

```
SwapRoute Pair: 0xPEPE_PLS

Before trade:              After trade:
  Reserves:                   Reserves:
    750,000,000 PEPE            746,268,657 PEPE (Bob received 3,731,343)
    10,000 PLS                  10,100 PLS       (Bob sent 100)

Bob's Wallet:
  -100 PLS
  +3,731,343 PEPE
```

Bob trades on SwapRoute just like any DEX. The pair has real liquidity.
The fact that the LP tokens are locked doesn't affect trading — the
reserves stay in the pair, available for swaps.

**Trustless Lock has NO impact on trading.** It only prevents
removing liquidity from the pair.

---

## Step 5: 180 Days Later — Lock Expires

Time lock expires at block 4,665,600.

Alice can now call `trustlessLock_release(1)` to get her LP tokens
back.

With LP tokens in hand, she can withdraw her share of the pool
reserves from SwapRoute.

**What the community sees:**
- 180 days of uninterrupted trading
- No rug risk during that period
- If Alice built a real project, she earns from her LP position
- If Alice was a scammer, she can't touch the liquidity for 180 days

---

## Where Each Token Lives

```
                     ┌──────────────────┐
                     │   SwapRoute Pair  │
                     │   0xPEPE_PLS     │
                     │                  │
                     │  Reserves:       │
                     │   750M PEPE      │ ← Traded by community
                     │   10,000 PLS     │ ← Traded by community
                     └──────┬───────────┘
                            │
                            │ LP tokens represent ownership
                            │ of these reserves
                            ▼
                     ┌──────────────────┐
                     │  Trustless Lock  │
                     │   Lock #1        │
                     │                  │
                     │  Holds:          │
                     │   1,000 LP       │ ← No one can touch
                     │   tokens         │   for 180 days
                     └──────────────────┘
                            │
                            │ After 180 days:
                            ▼
                     ┌──────────────────┐
                     │  Alice's Wallet  │
                     │  1,000 LP tokens │ ← Can now withdraw
                     │                  │   from SwapRoute
                     └──────────────────┘
```

---

## What Was Missing in Our Thinking

The gap was: **where does the actual tradeable liquidity live?**

Trustless Lock holds LP tokens (ownership proof), but the actual
tradable reserves (PEPE + PLS) live in the **SwapRoute pair
contract**. The community trades against those reserves on SwapRoute.
Locking the LP tokens just means no one can pull the reserves out.

**Template Contract orchestrates everything.** Three separate contracts,
three jobs. The user never touches LP tokens:

| Contract | Holds | Does |
|----------|-------|------|
| **Template Contract** | Nothing (temporary intermediary) | Deploys token, calls DEX, locks LP, renounces — all atomic |
| **SwapRoute Pair** | PEPE + PLS reserves | Handles swaps, maintains price |
| **Trustless Lock** | LP tokens (ownership proof) | Prevents liquidity withdrawal for 180 days |
| **PEPE Token** | Balance of every holder | ERC-20 transfers, approvals |

The community **never interacts with Trustless Lock**. They buy/sell
on SwapRoute. Trustless Lock is invisible to traders — it's a
safety mechanism for the deployer's side only.