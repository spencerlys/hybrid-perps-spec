---
doc_id: scenario-en-order-routing
title: Order Routing Scenarios
tags: [routing, internal, hyperliquid, threshold, force-route]
version: 1.0
lang: en
updated: 2026-04-08
related-prd: [prd/02-routing.md, prd/04-internal-execution.md, prd/05-hl-execution.md]
---

# Order Routing Scenarios

---

## SC-RT-001: Small Market Order → Internal Internalization

**Preconditions:**
- BTC HL mark price: $100,000
- Routing threshold: $10,000
- User available balance: $1,000; leverage: 10x

**Steps:**
1. User places BTC long market order, qty 0.05 BTC
2. Router computes notional: 0.05 × 100,000 = **$5,000 ≤ $10,000** → INTERNAL

**Expected Results:**
- Fill at HL current best ask (taker price)
- Create INTERNAL position (position_source = INTERNAL)
- Platform auto-creates mirror short position (internal ledger only)
- User margin frozen: $500 ($5,000 / 10x)
- Routing log: decision = INTERNAL, latency < 5ms

---

## SC-RT-002: Large Market Order → HL Proxy Execution

**Preconditions:**
- ETH HL mark price: $4,000
- Routing threshold: $10,000
- User available balance: $5,000; leverage: 5x

**Steps:**
1. User places ETH long market order, qty 5 ETH
2. Router computes notional: 5 × 4,000 = **$20,000 > $10,000** → HYPERLIQUID
3. Send open order instruction to HL platform account

**Expected Results:**
- HL returns fill receipt: fill_price, filled_size, fee
- entry_price updated to fill_price (fill price correction)
- User margin frozen: $4,000 ($20,000 / 5x)
- HL platform account merged long position increases by 5 ETH

---

## SC-RT-003: Same Asset, Two Opens — Split Routing

**Preconditions:**
- BTC mark price: $100,000; threshold: $10,000
- User has sufficient balance

**Steps:**
1. First open: BTC long, notional $5,000 → **INTERNAL**
2. Second open: BTC long, notional $15,000 → **HYPERLIQUID**

**Expected Results:**
- User holds two independent BTC long positions:
  - BTC INTERNAL ($5K, own entry_price)
  - BTC HYPERLIQUID ($15K, own entry_price)
- Cross mode: both share account equity; each has independent entry_price
- Isolated mode: each liquidated independently, no cross-impact
- Close routing follows position source (INTERNAL closes internally; HYPERLIQUID closes via HL)

---

## SC-RT-004: Force-Route to HL — Net Exposure at Limit

**Preconditions:**
- BTC net exposure at $1M limit (internalization stopped for BTC)
- User places BTC long, notional $3,000 (would normally route INTERNAL)

**Steps:**
1. Router checks BTC net exposure → at limit
2. Force-route to HYPERLIQUID (even though $3,000 < $10,000 threshold)

**Expected Results:**
- HYPERLIQUID position created
- Routing log records force reason: `net_exposure_limit`
- User sees no difference (same price, same receipt format)

---

## SC-RT-005: Force-Route to HL — HL Latency Exceeded

**Preconditions:**
- HL WebSocket latency: 600ms (exceeds 500ms threshold)
- User places ETH long, notional $5,000

**Steps:**
1. Router detects HL channel latency > 500ms → pause internalization
2. Force-route to HYPERLIQUID

**Expected Results:**
- Route to HYPERLIQUID (conservative fallback)
- Risk alert: HL latency > 500ms
- Routing log records force reason: `hl_latency_exceeded`

---

## SC-RT-006: Close Order Routing Follows Original Position

**Preconditions:**
- User holds: BTC INTERNAL long + BTC HYPERLIQUID long

**Steps:**
1. User initiates close of BTC INTERNAL position
2. User initiates close of BTC HYPERLIQUID position

**Expected Results:**
- BTC INTERNAL close → L3 internal settlement (no HL instruction sent)
- BTC HYPERLIQUID close → L4 sends market close order to HL
- Cross-system offset forbidden: INTERNAL cannot offset HYPERLIQUID

---

## SC-RT-007: Consecutive Opens Trigger Hedging

**Preconditions:**
- BTC mark price: $100,000; routing threshold: $10,000
- Current BTC net exposure: $80,000 (long direction)
- Hedging threshold: $100K triggers 50% hedge

**Steps:**
1. User A places BTC long order, notional $5,000 → INTERNAL ($5K < $10K threshold)
2. User B places BTC long order, notional $8,000 → INTERNAL
3. User C places BTC long order, notional $9,000 → INTERNAL

**Expected Results:**
- All three orders route to INTERNAL (each notional < $10K)
- Cumulative net exposure: $80K + $5K + $8K + $9K = $102K
- After the third fill, L6 detects net exposure > $100K
- L6 triggers hedge: opens BTC short ~$51K on HL (50% × $102K)
- Hedge instruction routes through L4 to HL, independent from user orders
- Users remain unaware (still see normal INTERNAL positions)

---

## SC-RT-008: Natural Hedging from Multi-Directional Users

**Preconditions:**
- ETH mark price: $4,000; routing threshold: $10,000
- Current ETH net exposure: $0

**Steps:**
1. User A places ETH long order, notional $8,000 → INTERNAL (platform holds short $8K)
2. User B places ETH short order, notional $8,000 → INTERNAL (platform holds long $8K)

**Expected Results:**
- Both orders route to INTERNAL
- Net exposure: $8K - $8K = $0 (long/short fully offset)
- No hedging triggered
- Platform earns both trading fees + future funding fees
- This is the ideal state for the B-book model: zero directional risk while earning fees
