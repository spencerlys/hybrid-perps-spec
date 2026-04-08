---
doc_id: prd-en-internal-execution
title: L3 Internal (B-Book) Execution Layer — Position Model, Counterparty Accounting
tags: [internal, b-book, execution, position, counterparty, L3]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 3
---

# L3 Internal (B-Book) Execution Layer

## Internalization Model

The platform holds an exact mirror position opposite to the user. Recorded internally only — no HL position is opened (unless hedging is triggered).

```
User opens BTC long $5,000 (INTERNAL)
  └─► Platform auto-holds BTC short $5,000 (counterparty)
      → Internal ledger entry only, no on-chain action
```

## Internal Position Data Model

| Field | Description |
|-------|-------------|
| position_id | Position ID |
| user_id | User |
| symbol | Contract symbol (e.g. BTC-PERP) |
| direction | `LONG` / `SHORT` |
| size | Position size |
| notional_value | size × entry_price |
| entry_price | Fill price (HL mark or best bid/ask at open time) |
| leverage | Leverage multiplier |
| margin_mode | `ISOLATED` / `CROSS` |
| isolated_margin | Isolated margin (null for cross) |
| position_source | Always `INTERNAL` |
| status | `OPEN` / `CLOSED` / `LIQUIDATED` |
| created_at | Open timestamp |
| closed_at | Close/liquidation timestamp |

## Execution Price Rules

| Order Type | Execution Price |
|------------|----------------|
| Market | HL current best ask (long) or best bid (short) |
| Limit | Filled at limit price when touched |
| Mark Price | Used for unrealized PnL and liquidation price calculation |

## Internal Position Operations

### Open
1. Routing decision = INTERNAL
2. Fill at HL mark price / best bid/ask
3. Create fill record
4. Freeze user margin
5. Create position record (position_source = INTERNAL)
6. Create platform counterparty position (internal)
7. Push WebSocket fill notification

### Add to Position (Same Direction)
- Each add-on order creates a new independent position record with its own entry_price
- Each order is routed independently (a $12K add-on may route to HL)

### Close
1. User initiates close (market / TP/SL)
2. INTERNAL positions must close internally — never via HL
3. Fill at current HL mark price / best bid/ask
4. Calculate realized PnL (see [09-settlement.md](09-settlement.md))
5. Release frozen margin
6. Update position status to CLOSED

### Liquidation (Forced Close)
Triggered by L5 liquidation engine (see [06-margin-liquidation.md](06-margin-liquidation.md)):
1. Liquidation engine detects trigger condition
2. Settle internally at HL mark price
3. User margin zeroed
4. 80% → platform profit, 20% → risk reserve
5. Position status → LIQUIDATED

## Internalization Revenue Model

```
Platform Net P&L (INTERNAL) = Σ(user losses) - Σ(user gains)

Per-loss allocation:
  80% → Platform profit
  20% → Risk reserve (ratio configurable in admin)
```

## Internal Limit Order Book
- Maintains a pending queue for INTERNAL limit orders
- HL real-time price used as trigger reference
- Price touched → immediate fill → follow open position flow
- Cancel: remove from queue, release frozen margin → CANCELLED
