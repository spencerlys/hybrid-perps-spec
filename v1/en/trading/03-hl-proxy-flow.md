---
doc_id: trading-en-hl-proxy-flow
title: HL Proxy Execution Flow
tags: [hyperliquid, proxy, fill-price, drift, position-mapping, L4]
version: 1.0
lang: en
updated: 2026-04-08
related-prd: [prd/05-hl-execution.md, prd/09-settlement.md]
---

# HL Proxy Execution Flow

## Open Position Complete Flow

```
L2 routing decision → HYPERLIQUID
      │
      ▼
[L4] Pre-calculate margin (using current mark_price)
     └─ Temporarily freeze user margin (deduct from available_balance)
      │
      ▼
[L4] Build HL open order instruction
     ├─ Direction (BUY / SELL)
     ├─ Size (rounded per HL szDecimals rules)
     ├─ Type (market / limit)
     └─ Platform Agent Key signature
      │
      ▼
[L4] Send open order to HL (target: < 50ms)
      │
      ▼
[L4] Await HL fill receipt
     ├─ fill_price   actual fill price
     ├─ filled_size  actual filled quantity
     └─ fee          trading fee (includes builder fee)
      │
      ▼
[L4] Fill Price Correction
     ├─ entry_price = fill_price (replaces estimate)
     ├─ Recompute actual frozen margin using fill_price
     └─ Adjust available_balance (release or deduct delta)
      │
      ▼
[L4] Create HYPERLIQUID position record
     ├─ position_source = HYPERLIQUID
     ├─ entry_price = fill_price (from HL receipt)
     └─ status = OPEN
      │
      ▼
[L4] Write to fills table (source = HL, includes hl_fill_price field)
      │
      ▼
[L4] Update HL virtual position mapping (user share → HL merged position)
      │
      ▼
[WS] Push fill notification to user
```

## Close Position Complete Flow

```
User initiates close of HYPERLIQUID position
      │
      ▼
[L4] Build HL close order (market close)
     └─ Size = user virtual position size
      │
      ▼
[L4] Send close order to HL
      │
      ▼
[L4] Await HL close receipt (close_price, filled_size, fee)
      │
      ▼
[L8] Compute realized PnL using close_price
     ├─ Long: pnl = (close_price - entry_price) × size
     └─ Short: pnl = (entry_price - close_price) × size
      │
      ▼
[L8] Drift calculation and logging
     ├─ drift = HL receipt PnL - XBIT pre-calculated PnL
     ├─ drift > $10 → write to deviation_logs
     ├─ drift rate > 1% → Slack alert
     └─ drift rate > 5% → P0 + halt HL routing for this asset
      │
      ▼
[L8] Drift fallback (if HL actual loss > XBIT calculated)
     └─ Deduct delta from risk reserve; settle user at XBIT calculated value
      │
      ▼
[L4] Release user margin → position.status = CLOSED
      │
      ▼
[L4] Update HL virtual position mapping (subtract closed size)
      │
      ▼
[WS] Push close notification
```

## Partial Fill Handling

HL market orders may fill in multiple tranches (common for large orders):

```
Fill 1: price=$100,100, size=0.3
Fill 2: price=$100,050, size=0.5
Fill 3: price=$100,000, size=0.2

Volume-weighted average entry_price:
= (100,100×0.3 + 100,050×0.5 + 100,000×0.2) / 1.0
= $100,055
```

## HL-Position Liquidation Flow

```
[L5] XBIT liquidation engine determines liquidation needed (HYPERLIQUID position)
      │
      ▼
[L4] Send market close order to HL
     └─ Size = user virtual position size
      │
      ▼
[L4] Await HL receipt (close_price)
      │
      ▼
[L8] Settle liquidation at close_price
     └─ User margin zeroed
      │
      ▼
[L4] position.status = LIQUIDATED
```

## HL Consistency Check (Every 5 Minutes)

```
Σ(all user HYPERLIQUID virtual position sizes per asset)
  vs HL account actual merged position size

Deviation > 0.01% → log to reconciliation_logs + alert
Deviation > 0.1%  → P0 emergency + halt new HL opens + manual review
```
