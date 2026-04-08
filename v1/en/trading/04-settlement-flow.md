---
doc_id: trading-en-settlement-flow
title: Settlement & Liquidation Flow
tags: [settlement, pnl, funding-rate, liquidation, reconciliation, L8, L5]
version: 1.0
lang: en
updated: 2026-04-08
related-prd: [prd/06-margin-liquidation.md, prd/09-settlement.md]
---

# Settlement & Liquidation Flow

## 1. Realized PnL Settlement Flow

```
Position closed (user-initiated or liquidation)
      │
      ▼
Determine close_price source:
  ├─ INTERNAL → HL mark_price / best bid-ask (L3 internal)
  └─ HYPERLIQUID → HL close receipt (L4)
      │
      ▼
Calculate realized PnL:
  ├─ Long: pnl = (close_price - entry_price) × size
  └─ Short: pnl = (entry_price - close_price) × size
      │
      ▼
Deduct trading fee:
  fee = notional × fee_rate
  available_balance -= fee
      │
      ▼
Release margin:
  ├─ Isolated: available_balance += isolated_margin
  └─ Cross: cross_margin_used -= initial_margin
      │
      ▼
Credit PnL:
  ├─ Profit (pnl > 0): available_balance += pnl
  └─ Loss (pnl < 0): deduct from margin; return remainder to user
      │
      ▼
Write to balance_logs (type = realized_pnl)
```

## 2. Funding Rate Settlement Flow

```
Every 8 hours (UTC 00:00 / 08:00 / 16:00):
      │
      ▼
[L7] Fetch HL current funding rate (all symbols)
      │
      ▼
[L8] Scan all open positions (status = OPEN)
      │
      ▼
[L8] For each position, calculate funding:
     payment = size × mark_price × funding_rate
     ├─ payment > 0 (long pays): available_balance -= payment
     └─ payment < 0 (short receives): available_balance += |payment|
      │
      ▼
By position type:
  ├─ INTERNAL:
  │   └─ Internal ledger: debit/credit user and platform counterparty
  │
  └─ HYPERLIQUID:
      ├─ HL settles on platform HL account (automatic)
      └─ Platform mirrors same amount to user account
      │
      ▼
Write to funding_settlements table
      │
      ▼
[L5] Update liquidation price (funding erodes margin → liq price moves adversely)
      │
      ▼
[L8] Reconciliation check (Platform mirror total vs HL actual settlement)
```

## 3. Liquidation Detection & Execution Flow

### Detection (Real-Time)

```
[L7] HL mark_price push (real-time)
      │
      ▼
[L5] Liquidation engine scans affected positions (target: < 1s)
      │
      ├─ Isolated check:
      │   (position_margin + unrealized_pnl) ≤ notional × maintenance_rate
      │   → Trigger liquidation for this position
      │
      └─ Cross check:
          account_equity ≤ cross_maintenance_requirement
          → Trigger account-level liquidation
```

### Execution

```
[L5] Liquidation signal triggered
      │
      ├─ INTERNAL position ──► [L3] Settle at HL mark_price
      │                             [L8] Margin zeroed + 80/20 client loss split
      │                             position.status = LIQUIDATED
      │
      └─ HYPERLIQUID position ──► [L4] Send market close order to HL
                                       [L4] Await HL receipt (close_price)
                                       [L8] Settle at close_price
                                       [L8] Drift detection + fallback
                                       position.status = LIQUIDATED
      │
      ▼
[WS] Push liquidation notification to user
[DB] Write to liquidations table
```

## 4. Three-Way Reconciliation Flow

```
Real-time:
  └─ Σ(balances + margins + unrealized PnL) = platform total user liability

Every 5 minutes:
  └─ HL virtual position sizes vs HL actual position sizes

Hourly:
  └─ Platform B-book net P&L summary (INTERNAL positions only)

Daily:
  └─ Drift accumulation summary → reconciliation_logs
  └─ Per-chain aggregation and withdrawal reconciliation
```

## 5. Settlement Exception Handling

| Exception | Handling |
|-----------|---------|
| HL receipt timeout | Retry 3x; if still no receipt → P0 alert + manual handling |
| Platform vs HL PnL drift > 5% | Halt HL routing for this asset + P0 alert + risk review |
| User asset reconciliation deviation > 0.1% | Halt withdrawals + P0 alert + manual audit |
| Risk reserve < $200K | Halt all INTERNAL positions (route everything to HL) |
| HL account margin ratio < 150% | Halt new HL opens + emergency capital top-up |
