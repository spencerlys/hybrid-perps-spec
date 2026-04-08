---
doc_id: prd-en-margin-liquidation
title: L5 Margin & Liquidation Model — Isolated/Cross Formulas, HL Precision, Engine
tags: [margin, liquidation, isolated, cross, formula, maintenance-rate, L5]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 3
---

# L5 Margin & Liquidation Model

> The most critical module in the system. All formulas must maintain ≥95% consistency with HL.

## Numerical Precision Rules

**All calculations must align with HL precision, fetched from HL meta API and cached:**

| Parameter | Source | Notes |
|-----------|--------|-------|
| Price precision (szDecimals) | HL `/info` meta API | Per-asset; BTC=1 decimal |
| Size step | HL `/info` meta API | Minimum order size and increment |
| Maintenance margin rate tiers | HL `/info` meta API | Tiered by notional size |

Precision rules synced hourly.

## Isolated Margin Mode

Each position has an independent margin pool. Liquidations are isolated and do not affect other positions.

### Initial Margin
```
Initial Margin = Notional Value / Leverage = size × entry_price / leverage
```

### Maintenance Margin Rate (Tiered)
```
Example (BTC):
  Notional < $100K   → Rate 0.4%
  Notional $100K–$1M → Rate 0.6%
  Notional > $1M     → Rate 1.0%
```
**Must be fetched from HL API. Never hardcode.**

### Liquidation Price (With Funding Erosion)

**Isolated Long:**
```
Liq Price = entry_price × (1 - (margin - cumulative_funding_paid) / notional + maintenance_rate)
```

**Isolated Short:**
```
Liq Price = entry_price × (1 + (margin - cumulative_funding_paid) / notional - maintenance_rate)
```

### Isolated Liquidation Trigger
```
When: (position_margin + unrealized_pnl) ≤ notional × maintenance_rate
→ Trigger liquidation for this position; other positions unaffected
```

## Cross Margin Mode

All cross positions share account equity. A loss in one position affects the liquidation threshold of others.

### Key Metrics
```
Account Equity = Wallet Balance + Σ(all cross position unrealized PnL)
Cross Maintenance Req = Σ(notional × maintenance_rate per cross position)
```

### Cross Liquidation Trigger
```
Account Equity ≤ Cross Maintenance Req → Trigger account-level liquidation
```

### Cross Liquidation Order
1. Close the most-losing position first
2. Recalculate account equity after each close
3. If equity recovered > remaining maintenance requirement → stop liquidating
4. If not → close next position, until all are closed

### Cross Liquidation Execution
- INTERNAL positions → L3 internal settlement
- HYPERLIQUID positions → send close order to HL (L4)

## Mixed Mode (Isolated + Cross)

Users can hold both isolated and cross positions simultaneously:
- Isolated position liquidation does not affect cross positions
- Available balance:

```
Available = Wallet Balance - Σ(isolated margins) + Σ(cross unrealized PnL)
```

## Unified Liquidation Engine

Applies to both INTERNAL and HYPERLIQUID positions. XBIT is the sole liquidation authority.

### Detection
- Trigger: mark price update → check all affected positions
- Latency target: **< 1 second** (mark price change → liquidation trigger)
- Frequency: every mark price update

### Execution

| Position Type | Method | Platform P&L |
|--------------|--------|-------------|
| INTERNAL | L3 internal settlement at mark price | Platform earns user margin (80/20 split) |
| HYPERLIQUID | Send market close to HL → await receipt → settle at close_price | No additional client loss |

### Post-Liquidation
```
User margin zeroed
Realized PnL recorded
INTERNAL: 80% → platform profit, 20% → risk reserve
Position status → LIQUIDATED
Push liquidation notification to user
```
