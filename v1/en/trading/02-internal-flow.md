---
doc_id: trading-en-internal-flow
title: Internal (B-Book) Execution Flow
tags: [internal, b-book, flow, margin, pnl, L3]
version: 1.0
lang: en
updated: 2026-04-08
related-prd: [prd/04-internal-execution.md, prd/06-margin-liquidation.md, prd/07-risk-management.md]
---

# Internal (B-Book) Execution Flow

## Open Position Flow

```
L2 routing decision → INTERNAL
      │
      ▼
[L3] Fetch current HL mark_price / best bid-ask
      │
      ▼
[L3] Margin pre-calculation
     ├─ Isolated: initial_margin = notional / leverage
     └─ Cross: check if account equity - current cross req ≥ new initial margin
      │
      ▼
[L3] Freeze user margin
     ├─ Isolated: wallets.frozen_margin += initial_margin
     └─ Cross: cross_margin_used += initial_margin
      │
      ▼
[L3] Create INTERNAL position record
     ├─ position_source = INTERNAL
     ├─ entry_price = current taker price (HL best bid or ask)
     └─ status = OPEN
      │
      ▼
[L3] Create platform counterparty position (opposite direction, same notional)
      │
      ▼
[L3] Write to fills table (source = INTERNAL)
      │
      ▼
[L6] Update net exposure (real-time aggregate)
     └─ Check whether hedge threshold is triggered
      │
      ▼
[WS] Push fill notification to user
```

## Margin Freeze Logic

```
Isolated mode:
  frozen_amount = notional / leverage
  wallets.frozen_margin += frozen_amount
  wallets.available_balance -= frozen_amount

Cross mode:
  Verify: account_equity - cross_maintenance_req ≥ new_initial_margin
  cross_margin_used += initial_margin
  (available_balance computed dynamically from cross formula)
```

## Close Position Flow

```
User initiates close
      │
      ▼
[L3] Fetch current HL mark_price / best bid-ask (close price)
      │
      ▼
[L3] Calculate realized PnL
     ├─ Long: realized_pnl = (close_price - entry_price) × size
     └─ Short: realized_pnl = (entry_price - close_price) × size
      │
      ▼
[L3] Release frozen margin
     └─ available_balance += frozen_margin (returned)
      │
      ▼
[L8] Credit PnL
     ├─ Profit (pnl > 0): available_balance += realized_pnl
     └─ Loss (pnl < 0): balance reduced (margin already covered)
      │
      ▼
[L3] position.status = CLOSED
      │
      ▼
[L6] Update net exposure (subtract closed position)
      │
      ▼
[WS] Push close notification
```

## Liquidation Detail Flow

```
[L5] Detects INTERNAL position liquidation condition
      │
      ▼
[L3] Settle internally at HL mark_price (liquidation price = mark_price)
      │
      ▼
[L8] User margin zeroed
      │
      ▼
[L8] Client loss split
     ├─ 80% → platform_profit account
     └─ 20% → risk_reserve account
      │
      ▼
[L3] position.status = LIQUIDATED
      │
      ▼
[L6] Update net exposure (subtract liquidated position)
      │
      ▼
[WS] Push liquidation notification (including liquidation price)
```

## Net Exposure Linkage

```
After every INTERNAL position change, L6 updates net exposure in real-time:

BTC Net Exposure += user BTC INTERNAL long notional
BTC Net Exposure -= user BTC INTERNAL short notional

if Net Exposure ≥ $100K:
  ├─ $100K ~ $500K → L4: send 50% hedge order to HL
  ├─ $500K ~ $1M   → L4: send 80% hedge order to HL
  └─ > $1M         → Halt new INTERNAL opens for this asset + L4: send 80% hedge
```

## Internal Limit Order Queue

```
User places INTERNAL limit order
      │
      ▼
Add to internal pending queue (sorted by price priority)
      │
      ▼
L7 real-time mark_price push → scan queue
      │
  Price touched ──► Immediate fill (follow open position flow)
      │
  User cancels ──► Remove from queue; release frozen margin → CANCELLED
```
