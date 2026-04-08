---
doc_id: trading-en-order-lifecycle
title: Order Full Lifecycle
tags: [order, lifecycle, open, close, cancel, routing, fill]
version: 1.0
lang: en
updated: 2026-04-08
related-prd: [prd/02-routing.md, prd/04-internal-execution.md, prd/05-hl-execution.md]
---

# Order Full Lifecycle

## Order State Machine

```
PENDING_ROUTING
    │
    ▼ Routing decision (< 5ms)
ROUTED (INTERNAL / HYPERLIQUID)
    │
    ├─ Market order ──► FILLING ──► FILLED
    │
    └─ Limit order ──► OPEN (waiting for price)
                         │
                         ├─ Price touched ──► FILLING ──► FILLED
                         └─ User cancelled ──► CANCELLED
```

## Open Position Complete Flow

### Step 1: Order Reception & Validation

```
User submits order
  ├─ Symbol validation (is contract listed?)
  ├─ Leverage range check (≤ maxLeverage from HL meta)
  ├─ Size precision check (szDecimals from HL meta)
  ├─ Minimum size check
  └─ Available balance check (sufficient margin?)
```

### Step 2: Routing Decision (L2, < 5ms)

```
Notional = size × HL mark_price
Notional ≤ threshold → INTERNAL
Notional > threshold → HYPERLIQUID
Force-HL condition checks (exposure / latency / volatility / manual override)
```

### Step 3A: INTERNAL Execution (L3, < 10ms)

```
Fill at HL mark price / best bid-ask
Freeze user margin (isolated: frozen_margin; cross: cross_margin_used)
Create position (position_source = INTERNAL)
Create platform mirror counterparty position (internal ledger)
Write to fills table
Push WebSocket fill notification
↓
L6 net exposure updated in real-time (check hedge thresholds)
```

### Step 3B: HYPERLIQUID Execution (L4, < 50ms)

```
Pre-estimate margin; temporarily freeze
Send open order to HL
Await HL fill receipt (fill_price, filled_size, fee)
Replace entry_price with fill_price (fill price correction)
Recompute margin; release or deduct delta
Create position (position_source = HYPERLIQUID)
Write to fills table (includes hl_fill_price field)
Push WebSocket fill notification
```

## Close Position Complete Flow

### User-Initiated Close

```
User initiates close (full / partial)
  │
  ├─ INTERNAL position ──► L3 internal close
  │   ├─ Fill at HL mark price / best bid-ask
  │   ├─ Compute realized PnL
  │   ├─ Release margin
  │   └─ position.status = CLOSED
  │
  └─ HYPERLIQUID position ──► L4 send close order to HL
      ├─ Await HL receipt (close_price)
      ├─ Compute realized PnL using close_price
      ├─ Release margin
      └─ position.status = CLOSED
```

### TP/SL Triggered Close

```
L7 real-time mark_price push
Platform scans all TP/SL orders
Price touches TP/SL level → trigger close
Execution path same as above (INTERNAL / HYPERLIQUID handled separately)
```

## Limit Order Lifecycle

```
User places limit order
  │
  ▼
Routing decision (INTERNAL / HYPERLIQUID)
  │
  ├─ INTERNAL ──► Added to internal pending queue (OPEN)
  │                L7 real-time price touches limit → immediate fill → FILLED
  │                User cancels → remove from queue; release margin → CANCELLED
  │
  └─ HYPERLIQUID ──► Submit limit order to HL
                       HL fills → receipt pushed → FILLED
                       User cancels → send cancel to HL → CANCELLED
```

## Order Data Model

| Field | Description |
|-------|-------------|
| id | Order unique ID |
| user_id | User |
| symbol | Contract symbol |
| side | BUY / SELL |
| size | Quantity |
| price | Limit order price (null for market) |
| route | INTERNAL / HYPERLIQUID |
| status | PENDING_ROUTING / ROUTED / FILLING / FILLED / CANCELLED |
| routing_latency_ms | Routing decision latency (ms) |

## Key Performance Requirements

| Operation | Target |
|-----------|--------|
| Routing decision | < 5ms P99 |
| INTERNAL fill | < 10ms |
| HL order forwarding | < 50ms |
| WebSocket fill notification | < 100ms |
