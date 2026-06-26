# WayChain Launch Plan — v2.0 (COMPLETE)

**Status:** All 5 phases complete. Chain is live at waychain.org.
**Chain:** 80,000+ blocks at 1/sec, 3 validators, BoltDB persistence
**Stack:** Go daemon + Cloudflare Tunnel + Vercel frontend
**Tagline:** "Actually Decentralized"

---

## Phase 0: Foundation ✅ COMPLETE

### P0.1 — Persistent State ✅
- [x] BoltDB integration — store/store.go with accounts, blocks, meta, node, txs buckets
- [x] Block storage + height metadata — sync on every block commit
- [x] Tx storage with hash index — O(1) tx lookup by hash
- [x] Node restart recovery — OpenStore loads accounts + height from disk
- [x] chain.db at ~/.waychain/ — 33MB+, restart-proven

### P0.2 — Transaction Lifecycle ✅
- [x] Transaction struct + TxPool (pending FIFO queue)
- [x] Ed25519 signature verification in block production
- [x] Nonce ordering enforced (skip wrong nonce)
- [x] Gas metering — actual gas used deducted from sender
- [x] Block gas limit (30M configured in EVM)
- [x] Sender balance check before execution
- [x] Binary wire format for tx serialization (serialize.go)
- [x] `eth_sendRawTransaction` — parses, validates sig/nonce/balance/deploy-gate, adds to pool, returns real hash
- [x] `eth_getTransactionByHash` — scans pool → in-memory blocks → persistent store index
- [x] `eth_getTransactionReceipt` — status, block hash, block number, gas used, from/to, contractAddress

### P0.3 — Multi-Process Node ✅
- [x] P2P networking — TCP-based, gob serialization, 6 message types
- [x] Multi-node discovery + gossip (tested 3-node mesh)
- [x] Block/tx/vote propagation between peers
- [x] P2P callbacks: OnTx adds to pool, OnBlock logs
- [x] BroadcastBlock() + BroadcastTransaction() wired into daemon loop
- [x] 3-validator set in production (proposers 01/02/03 rotating)

### P0.4 — CLI & Operations ✅
- [x] `waychain start` — long-running daemon with block production loop
- [x] `waychain init` — creates genesis.json with config
- [x] `waychain version` — shows version
- [x] systemd service — waychain.service with auto-restart
- [x] Graceful shutdown — SIGINT/SIGTERM → sync state → exit
- [x] Health endpoint at /health (blocks, status)
- [x] Structured logging — slog JSON format via `log/slog`
- [x] Rate limiting — 100 req/sec/IP, token bucket with auto-refill

---

## Phase 1: Safety ✅ COMPLETE

### P1.1 — Deploy Gate Enforcement ✅
- [x] EVM CREATE opcode checks deployer DoxDevLevel (requires L2+, ClassB)
- [x] EVM CREATE2 opcode added with same enforcement
- [x] Chain.ProduceBlock checks deployer level for deploy txs
- [x] RPC eth_sendRawTransaction validates deployer level
- [x] CanDeployContract helper in evm package
- [x] Tests verify L0 rejected, L2+ allowed, ClassB enforced

### P1.2 — Dox_Dev Infrastructure ✅
- [x] Badge application page at /badge on waychain.org
- [x] Badge level checking via way_getDoxLevel RPC
- [x] Curator info and verification workflow documentation
- [x] Genesis curators: alice (L3), bob (L2), treasury (L3)
- [x] Dev keypair auto-generated on daemon start with 10M WAY

### P1.3 — Template Registry (Solidity) ✅
- [x] TemplateRegistry contract compiles and tests pass
- [x] Class B deploy gating implemented
- [x] Ready for on-chain deployment when network launches

---

## Phase 2: Infrastructure ✅ COMPLETE

### P2.1 — WayChain.org ✅
- [x] Landing page — hero, features, live block feed
- [x] Dashboard — real-time WS push, stats, address lookup
- [x] Block Explorer — latest blocks, search, block/tx/account detail
- [x] Badge UI — check Dox_Dev level, level descriptions
- [x] SSL via Vercel + Cloudflare (HTTPS)
- [x] DNS: waychain.org → Vercel, api.waychain.org → Cloudflare Tunnel

### P2.2 — Public RPC ✅
- [x] Public JSON-RPC endpoint (api.waychain.org:8545 via tunnel)
- [x] Rate limiting per IP (100/sec)
- [x] WebSocket support for subscriptions (eth_subscribe)
- [x] Chain ID 10008 registered
- [x] CORS configured for web clients

### P2.3 — Token Infrastructure ✅
- [x] WAY token on WayChain itself (native — no contract needed)
- [x] 1WAY stablecoin spec (bitcoin-backed, oracle multi-sig)
- [x] 2WAY stablecoin spec (multi-collateral, cross-chain)
- [x] DEX pair contracts (WayChainFactory + WayChainPair) — 11 Solidity contracts ready

---

## Phase 3: User Interfaces ✅ COMPLETE

### P3.1 — Block Explorer ✅
- [x] Visual explorer page at /explorer
- [x] Latest 20 blocks with real data (height, hash, proposer, tx count, timestamp)
- [x] Block detail view — proposer, parent hash, state root, full tx list
- [x] Transaction detail — from, to, value, gas, calldata, signature, receipt
- [x] Account detail — balance, nonce, Dox_Dev level
- [x] Search bar (by block number, address, tx hash)
- [x] Live block stream via WebSocket

### P3.2 — Dashboard Upgrade ✅
- [x] Real-time stats via WebSocket: block height, TPS, account count
- [x] Network health indicators (connection mode: WS/HTTP/Offline)
- [x] Address/tx lookup with redirect to explorer
- [x] Navigation links to Explorer and Badge pages

### P3.3 — Badge UI ✅
- [x] Badge check form at /badge
- [x] Returns level, balance, nonce for any address
- [x] 4-level breakdown (L0-L3) with descriptions
- [x] Genesis curator list with addresses
- [x] Verification workflow documentation

---

## Phase 4: Real-Time Push ✅ COMPLETE

### P4.1 — WebSocket RPC ✅
- [x] WS support in RPCServer — detects Upgrade header
- [x] eth_subscribe: newHeads, newPendingTransactions
- [x] SubscriptionManager — thread-safe, auto-cleanup on disconnect
- [x] Block broadcasting — every new block pushed to subscribers
- [x] Same port (9545) handles HTTP+RPC and WebSocket
- [x] Verified: local WS and tunnel WS both deliver push notifications

### P4.2 — Dashboard WS Integration ✅
- [x] Dashboard connects via wss://api.waychain.org
- [x] Subscribes to newHeads — pushes block updates in real-time
- [x] Falls back to HTTP polling (3s) if WS disconnects
- [x] Status indicator shows connection mode (WS/HTTP/Offline)

---

## Phase 5: Hardening ✅ COMPLETE

### P5.1 — Multi-Validator ✅
- [x] 3 validators in production (0x01, 0x02, 0x03)
- [x] Sqrt-weighted proposer lottery — equal opportunity
- [x] Proposer rotation confirmed in live blocks
- [x] P2P mesh topology implemented
- [x] Block/tx broadcasting between peers

### P5.2 — Rate Limiting ✅
- [x] Token bucket rate limiter (rate_limit.go)
- [x] 100 requests/second/IP default
- [x] Automatic bucket refill
- [x] Stale entry cleanup (5-minute interval)
- [x] HTTP 429 response with JSON-RPC error
- [x] Middleware integrated into RPC server

### P5.3 — Structured Logging ✅
- [x] slog JSON handler (Go 1.21+ stdlib)
- [x] SetupLogger() initializes structured logging
- [x] LogBlock() — height, proposer, tx_count, accounts, bps
- [x] LogTxSubmission() — hash, from, pool_size
- [x] LogSubscription() — subscription_id, type, active_subs
- [x] All daemon output in JSON format

---

## Architecture (Current)

```
Internet ──► Cloudflare (HTTPS) ──► Vercel :443
                    ├──► / ──► dashboard (index.html)
                    ├──► /explorer ──► explorer page
                    ├──► /badge ──► badge page
                    │
                    └──► api.waychain.org ──► Cloudflare Tunnel ──► cloudflared ──► NGINX :80
                                                                                      ├──► /rpc ──► waychain daemon :9545
                                                                                      ├──► /health ──► health check
                                                                                      │
                         P2P Network :9100 ◄──► Other validators (when deployed)
                         
                         ┌─────────────────────────────┐
                         │  WayChain Daemon             │
                         │  ├── BFT Consensus (1s)      │
                         │  ├── EVM (100+ opcodes)      │
                         │  ├── 12 Precompiles (0x0C-0x17)│
                         │  ├── BoltDB persistence       │
                         │  ├── Tx Pool + Mempool       │
                         │  ├── WebSocket subscriptions │
                         │  ├── Rate limiter             │
                         │  └── Structured logging       │
                         └─────────────────────────────┘
```

## What's Next

| Priority | Item | Description |
|----------|------|-------------|
| 1 | 2WAY Implementation | Multi-collateral stablecoin (spec complete) |
| 2 | Template Registry Deploy | First Solidity contracts on WayChain |
| 3 | Multi-Node Network | Deploy 3 validators on separate machines |
| 4 | Cross-Chain Bridge | LayerZero OFT for 2WAY omnichain |
| 5 | Developer Onboarding | Foundry/viem integration docs |

---

## Verification Checklist

| Item | Pass/Fail Test | Status |
|------|---------------|--------|
| Real tx pipeline | eth_sendRawTransaction → tx in next block → eth_getTransactionReceipt returns 0x1 | ✅ |
| Persistent tx | Restart daemon → eth_getTransactionByHash still returns tx | ✅ |
| Public access | Browser at https://waychain.org shows live block count | ✅ |
| Block explorer | /explorer shows recent blocks with non-zero tx counts | ✅ |
| Badge check | /badge shows correct Dox_Dev level for any address | ✅ |
| WebSocket | wss://api.waychain.org subscribes to newHeads, receives pushes | ✅ |
| Multi-validator | 3 validators producing blocks with rotating proposers | ✅ |
| Rate limiting | >100 req/sec from single IP returns 429 | ✅ |
| Structured logs | Daemon output is valid JSON with msg/level fields | ✅ |
