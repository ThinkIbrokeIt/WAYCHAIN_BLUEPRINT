# Governance 2.0 — Dox_Dev-Weighted Decision Protocol v1.0

WayChain's governance is not token-weighted. It is identity-weighted.
One verified human = one equal voice. This spec defines how decisions
are made, from parameter tweaks to existential changes.

---

## 0. Design Principles

| Principle | What It Means |
|-----------|---------------|
| **One human, one vote** | Dox_Dev Level 2+ badge = governance seat. Token weight is zero. |
| **Prevent capture** | No single entity or coalition can dominate every issue. |
| **Protect minorities** | Passionate minorities can win on their priority issues. |
| **Slow and careful** | Changes require time, deliberation, and high thresholds. |
| **Immutable core** | Some things cannot be voted on (one badge = one validator, progressive staking, fiat-pegged fees, no pre-mine). |
| **Self-correcting** | Bad decisions can be reversed, with friction proportional to severity. |

---

## 1. The Voting Mechanisms

WayChain uses three voting mechanisms, each suited to different decisions.

### 1.1 Direct Vote (Simple Issues)

Every badge holder votes directly. No delegation. No quadratic math.

Used for: parameter adjustments within established ranges, routine treasury
allocations, emergency freezes.

**How it works:**

```
1. Proposal created (100 WAY bond — returned if not abusive)
2. Discussion period: 3 days
3. Vote period: 7 days
4. Quorum: 20% of active badge holders
5. Pass threshold: simple majority (>50%)
6. Timelock: 7 days (simple), 30 days (treasury)

If quorum not met: proposal fails. Bond returned.
If proposal is spam/abusive: curators can flag → bond burned → proposer temp-suspended.
```

### 1.2 Quadratic Vote (Important Issues)

Each badge holder has a budget of **credits** per voting period (replenished
every 90 days). The cost to vote on an issue is the square of the number of
issues you vote on:

```
If there are 5 proposals this period:
- Vote on 1 proposal: costs 1 credit (1²)
- Vote on 2 proposals: costs 4 credits (2²)
- Vote on 3 proposals: costs 9 credits (3²)
- Vote on 4 proposals: costs 16 credits (4²)
- Vote on 5 proposals: costs 25 credits (5²)

Each badge holder gets 9 credits per 90-day period.
This allows voting on 3 issues per quarter at full strength,
or more issues with reduced weight per issue.
```

**Your vote weight on each issue you choose to vote on:**
```
weight = credits_spent / total_credits_spent_across_all_voters_on_this_issue
```

**Effect:** A passionate minority that cares deeply about one issue can
concentrate all their credits on it and win against a diffuse majority
that spreads across many issues. The majority still wins if they coordinate,
but they must prioritize.

Used for: inflation rate changes, fee target adjustments, validator set
size changes, treasury allocation adjustments.

**Mechanics:**
```
1. Proposal created (500 WAY bond)
2. Discussion period: 7 days
3. Vote period: 14 days
4. Quorum: 30% of active badge holders
5. Pass threshold: supermajority (>60%)
6. Timelock: 30 days (standard), 90 days (inflation/treasury)
```

### 1.3 Futarchy-Informed Vote (Critical Issues)

Before voting on high-impact changes, a **prediction market** runs to inform
the decision. The market outcome does not dictate the vote — but it provides
data that voters should consider.

**How it works:**

```
1. Proposal created (1,000 WAY bond)
2. Prediction market opens: "If this passes, will the WAY token price
   be higher in 90 days than at the time of proposal?"
3. Market duration: 7 days
4. Anyone with a Dox_Dev badge can trade (buy YES/NO shares)
5. Maximum position: 1,000 WAY per badge holder (anti-whale)
6. Market resolves 90 days after the vote (winning shares pay out)
7. Vote proceeds after market resolves
```

**The market is information, not governance.**
- If the market strongly predicts "NO" (price will drop), voters can still
  pass the proposal — but they must explain why they know better than the market.
- If the market strongly predicts "YES", voters have data supporting the change.
- The market outcome is recorded on-chain alongside the vote result for audit.

Used for: irreversible changes, existential parameters, new fee lanes, changes
to the immutable core via the Amendment Process (Section 4).

**Mechanics:**
```
1. Proposal created (1,000 WAY bond)
2. Prediction market: 7 days
3. Discussion period: 7 days (overlaps with market)
4. Vote period: 14 days
5. Quorum: 40% of active badge holders
6. Pass threshold: 2/3 supermajority (>66%)
7. Timelock: 90 days
```

---

## 2. Proposal Lifecycle

### 2.1 Submission

| Vote Type | Bond | Who Can Submit |
|-----------|------|---------------|
| Direct | 100 WAY | Any Dox_Dev Level 2+ |
| Quadratic | 500 WAY | Any Dox_Dev Level 3 |
| Futarchy | 1,000 WAY | Any Dox_Dev Level 3 + 10 endorsers (Level 2+) |

The bond prevents spam. If the proposal passes, the bond is returned.
If the proposal fails but was made in good faith (determined by curators),
the bond is returned. If abusive, the bond is burned.

### 2.2 Discussion Period

Each proposal has a discussion period before voting opens:
- Direct: 3 days
- Quadratic: 7 days
- Futarchy: 7 days (overlapping with prediction market)

During discussion:
- Proposal text is visible on-chain
- Badge holders can comment (stored on-chain or via Binary Journal anchor)
- Amendments can be proposed (new proposal with changes, old one deprecated)
- Curators can flag abusive proposals

### 2.3 Voting

| Parameter | Direct | Quadratic | Futarchy |
|-----------|--------|-----------|----------|
| Vote duration | 7 days | 14 days | 14 days |
| Quorum | 20% | 30% | 40% |
| Pass threshold | >50% | >60% | >66% |
| Vote types | For / Against / Abstain | Credits allocation | Yes / No / Abstain |
| Results | Raw count | Quadratic-weighted | Raw count |

### 2.4 Execution

If a proposal passes:
1. **Timelock** begins (7-90 days depending on type)
2. During timelock, badge holders can cancel if new information emerges
   (requires 3/4 supermajority to cancel a passed proposal)
3. After timelock, the parameter change is executed atomically
4. The change is recorded on-chain with the proposal ID

### 2.5 Emergency Override

If a parameter change would cause obvious harm (e.g., inflation set to 0%
causing the chain to stall):

| Mechanism | Threshold | Effect |
|-----------|-----------|--------|
| Emergency freeze | Simple majority, immediate | Freezes the parameter at current value, requires new vote to change |
| Curator veto | 3/5 curator council | Blocks execution, triggers automatic futarchy review |
| Community veto | 1/3 of badge holders sign within 7 days | Forces a new vote (the proposal must now pass at futarchy level) |

---

## 3. What Governance Controls

### 3.1 Adjustable Parameters

| Parameter | Current | Range | Vote Type | Timelock |
|-----------|---------|-------|-----------|----------|
| Inflation rate | 7% | 3-10% | Quadratic | 90 days |
| Fee USD targets | $0.001/tx | $0.0005-$0.01 | Quadratic | 30 days |
| Progressive bracket boundaries | 1K/10K/100K/1M | Adjustable | Quadratic | 90 days |
| Validator set size | 200 | 100-300 | Quadratic | 90 days |
| Treasury allocation % | 10% | 5-15% | Quadratic | 90 days |
| State rent burn % | 60% | 40-80% | Quadratic | 30 days |
| Slash percentages | 5%/0.5%/10% | 1-20% range | Quadratic | 90 days |

### 3.2 Future-Proofing (New Parameters)

New parameter types can be added via:
1. Direct vote to propose the new parameter
2. Futarchy vote to define its boundaries
3. Quadratic vote to set initial value

### 3.3 What Is Immutable

These were enforced at genesis. Governance cannot change them:

- One Dox_Dev badge = one validator (cannot be changed)
- One badge = one vote (token weight is always zero)
- Progressive staking exists (brackets adjustable, curve cannot be removed)
- Fiat-pegged fee model (peg mechanism cannot be removed)
- No pre-mine / no pre-sale (cannot be added after genesis)
- Burn mechanisms exist (percentages adjustable, mechanisms cannot be removed)
- Genesis distribution = equal per verified human (cannot be redistributed)

---

## 4. The Amendment Process — Changing the Immutable

If there is ever a need to change something in the immutable core (Section 3.3),
it requires the **Amendment Process:**

```
1. Futarchy proposal (1,000 WAY bond, 10 endorsers)
2. Prediction market: 14 days
3. Discussion: 14 days
4. Vote: 21 days
5. Quorum: 60% of active badge holders
6. Pass threshold: 3/4 supermajority (>75%)
7. Timelock: 180 days
8. During timelock: any badge holder can call for a review vote
   → Review vote: 21 days, 3/4 supermajority to confirm or cancel
```

**This process exists so the protocol can evolve if need arises,**
but the friction is intentionally high. An amendment should feel
like changing a country's constitution, not updating software.

---

## 5. Curator Council

A small council of trusted Dox_Dev Level 3 badge holders handles
administrative functions:

| Function | Description | Term |
|----------|-------------|------|
| Proposal filtering | Flag spam/abusive proposals | 90 days |
| Emergency freeze | Pause a parameter change | At will |
| Discussion moderation | On-chain comment curation | 90 days |
| Prediction market oversight | Ensure market integrity | 90 days |

**Council size:** 5 members
**Election:** Quadratic vote every 90 days
**Removal:** Simple majority vote, 7-day timelock
**Compensation:** 500 WAY per member per term (from treasury)

The council cannot change parameters. They can only flag, freeze,
or moderate. All council actions are public and reversible by vote.

---

## 6. Governance UI

### 6.1 On-Chain Minimal

The base layer is on-chain proposals and votes. Nothing fancy.
Badge holders interact via:
- WayChain dashboard (web UI)
- CLI commands (hermes governance)
- Direct contract calls

### 6.2 Off-Chain Deliberation

Discussion happens outside the chain:
- Binary Journal anchored proposals (immutable proposal records)
- Agora layer (Light truth discussion threads)
- Badge-holder-only channels (verified identity gated)

### 6.3 Quorum Notifications

If quorum isn't met in the first 5 days of a 7-day vote, a notification
is sent to all badge holders via their registered contact method.
If quorum isn't met by day 7, the proposal fails.

---

## 7. Comparison: WayChain vs. All Other L1s

| Feature | All Other L1s | WayChain |
|---------|--------------|----------|
| Voting weight | 1 token = 1 vote | 1 badge = 1 vote |
| Quadratic mechanism | None | Yes (credit-budgeted) |
| Prediction markets | None on-chain | Futarchy for critical decisions |
| Liquidity delegation | None | Yes (per-issue, revocable) |
| Immutable core | Usually not defined | Defined at genesis. Amendment possible but hard. |
| Curator council | None or opaque | 5 elected, public, limited scope |
| Proposal bonds | Often none | Scaled by impact (100-1,000 WAY) |
| Timelocks | Often none or bypassable | 7-180 days by impact |
| Emergency override | Foundation / multisig | Community-veto + curator freeze |

---

## 8. Summary

| Vote Type | For | Bond | Quorum | Threshold | Timelock |
|-----------|-----|------|--------|-----------|----------|
| Direct | Routine params | 100 WAY | 20% | >50% | 7-30 days |
| Quadratic | Important params | 500 WAY | 30% | >60% | 30-90 days |
| Futarchy | Critical/irreversible | 1,000 WAY | 40% | >66% | 90 days |
| Amendment | Change immutable core | 1,000 WAY | 60% | >75% | 180 days |

Governance 2.0 is the last piece. Everything before it — consensus, EVM,
accounts, Dox_Dev, oracle, UX, tokenomics, supply roadmap, Binary Journal,
cross-chain attestations — all of it is a chain waiting for its people
to govern it. This is how they do it.