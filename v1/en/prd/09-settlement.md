---
doc_id: prd-en-settlement
title: L8 Settlement & Reconciliation — PnL Formulas, Funding, Drift Fallback, Reconciliation
tags: [settlement, pnl, funding-rate, reconciliation, deviation, L8]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 3
---

# L8 Settlement & Reconciliation Layer

## PnL Calculation Formulas

### Unrealized PnL (Real-Time)
```
Long:  unrealized_pnl = (mark_price - entry_price) × size
Short: unrealized_pnl = (entry_price - mark_price) × size
```
- `mark_price`: from HL real-time push
- `entry_price`: **from HL fill receipt** — never estimated
- Applies to both INTERNAL and HYPERLIQUID positions

### Realized PnL (On Close/Liquidation)

**Normal Close:**
```
Long:  realized_pnl = (close_price - entry_price) × closed_size
Short: realized_pnl = (entry_price - close_price) × closed_size
```
- `close_price`: from HL close receipt (HYPERLIQUID) or internal execution price (INTERNAL)

**Partial Close:**
```
Remaining position entry_price unchanged (mirrors HL behavior)
Realized PnL for closed portion = (close_price - entry_price) × closed_size
```

**Average Entry After Add-On:**
```
new_entry_price = (old_entry × old_size + fill_price × new_size) / (old_size + new_size)
```

### Fee Handling
```
Fee = notional_value × fee_rate
```
- Deducted from available_balance on both open and close
- HL positions: actual fee read from HL receipt (includes builder fee)

## Funding Rate Settlement

### Formula
```
Funding Payment = position_size × mark_price × funding_rate
```
- `funding_rate` > 0 → longs pay shorts
- `funding_rate` < 0 → shorts pay longs

### Settlement Rules

| Rule | Description |
|------|-------------|
| Period | Every 8 hours (UTC 00:00, 08:00, 16:00) |
| Mid-period opens | Wait for next settlement point; no pro-rata calculation |
| Timing | Scan all open positions at each settlement point |
| Precision | Align with HL's published raw funding_rate value |

### INTERNAL vs HYPERLIQUID Funding Treatment

**INTERNAL:**
```
User pays → Platform account (platform is counterparty)
Platform pays → User account
Internal ledger adjustment only
```

**HYPERLIQUID:**
```
HL settles on platform HL account (automatic)
Platform mirrors same amount to user account:
  User owed funding → deduct from available_balance
  User receives funding → add to available_balance
```

## Settlement Operations Summary

| Operation | Settlement Logic |
|-----------|----------------|
| INTERNAL open | Freeze margin (isolated) or consume cross allocation |
| INTERNAL close | Release margin + compute realized PnL |
| INTERNAL liquidation | Margin zeroed → 80% platform profit + 20% reserve |
| HL open | Freeze user margin + open on HL platform account |
| HL close | Send close to HL → await receipt → settle at close_price → release margin |
| HL liquidation | Platform triggers → send close → await receipt → user margin zeroed |
| Funding | Every 8h settle all open positions (INTERNAL + HL mirror) |
| Trading fee | Deduct from available_balance on open/close |

## Drift Fallback Settlement

Applies only when a HYPERLIQUID position is closed/liquidated:

```
drift = HL receipt actual PnL - Platform pre-calculated PnL

if drift > 0 (HL actual loss > Platform calc):
  → Deduct delta from risk reserve
  → User settled at Platform calculated value (no user impact)
  → Record in deviation_logs

if drift < 0 (HL actual loss < Platform calc):
  → Delta goes to platform profit
  → Record in deviation_logs
```

## Reconciliation Engine

### Three-Way Reconciliation

| Dimension | Formula | Frequency |
|-----------|---------|-----------|
| User assets | `Σ(balances) + Σ(margins) + Σ(unrealized PnL)` = total user liability | Real-time |
| Platform B-book | `Net P&L = Σ(user losses) - Σ(user gains)` (INTERNAL only) | Hourly |
| HL account | `HL total assets ≈ injected capital + Σ(HL position P&L) + Σ(hedge P&L)` | Every 5 min |
| Hot wallets | `hot wallet balance ≥ daily withdrawal demand` | Real-time |
| Drift accumulation | `daily drift = Σ(per-trade drift)` | Daily summary |

### Alert Thresholds

| Dimension | Alert | Critical |
|-----------|-------|----------|
| User asset deviation | > 0.01% | > 0.1% |
| HL position size deviation | > 0.01% | > 0.1% |
| Single-trade drift rate | > 1% | > 5% |
| Daily cumulative drift | > $1,000 | > $5,000 |

Critical alert → auto-pause related operations + P0 alert + 5-minute response SLA
