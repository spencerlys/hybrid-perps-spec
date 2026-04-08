---
doc_id: prd-en-admin
title: L9 Admin & Operations Layer — Three-Mode Routing Config, Risk Dashboard, RBAC
tags: [admin, config, dashboard, rbac, operations, routing-mode, L9]
version: 1.1
lang: en
updated: 2026-04-08
phase: Phase 2-3
---

# L9 Admin & Operations Layer

## Routing Mode Control Panel

### Mode Switch

Admin users with Risk Manager role or above can switch between three routing modes:

```
Current Mode: [ HL Mode ]  [ Normal Mode ✓ ]  [ Betting Mode ]

⚠️ [Orange banner — visible in HL Mode only]
Net exposure is high. Please hedge on Hyperliquid immediately!
```

### Numeric Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Normal mode routing threshold | $10,000 | NORMAL_MODE: orders ≤ this value go to internalization |
| Betting mode routing threshold | $50,000 | BETTING_MODE: orders ≤ this value go to internalization |
| HL mode trigger exposure | $800,000 | Net exposure ≥ this → recommend/auto-switch to HL_MODE |
| Betting mode trigger exposure | $50,000 | Net exposure ≤ this → recommend/auto-switch to BETTING_MODE |
| Current net exposure (read-only) | Live refresh | Aggregate absolute net exposure across all assets; 5s refresh |
| Auto mode switching | Off | When enabled, auto-switches mode when thresholds are crossed + sends notification |
| Per-asset threshold override | Inherit global | Each asset can override the global threshold |

### Mode Switch History

Displays the last 30 mode switches at the bottom of the page:

| Time | Switch Action | Operator / Trigger |
|------|-------------|-------------------|
| 04-07 14:32 | Normal → HL Mode | Manual (Risk Mgr: admin@xbit) |
| 04-07 10:15 | Betting → Normal | Auto (net exposure exceeded $800K) |

### Other Routing & Internalization Configuration

| Feature | Description |
|---------|-------------|
| Internalization whitelist | Recommend BTC/ETH only at launch |
| Client loss reserve ratio | Default 20%; configurable |

## Risk Control Dashboard

### Real-Time Metrics

| Panel | Content |
|-------|---------|
| Net exposure | Per-asset real-time exposure + hedge status |
| Platform B-book P&L | Daily/weekly/monthly P&L curves |
| Routing stats | INTERNAL vs HYPERLIQUID count/volume/ratio |
| Liquidation stats | Today's count, amount, breakdown by INTERNAL/HL |
| HL account status | Margin ratio, available balance, position sizes |
| Risk reserve | Balance, recent changes, alert status |

### Calculation Drift Panel

| Metric | Description |
|--------|-------------|
| Daily drift total | Cumulative Platform calc vs HL actual delta |
| Drift trend chart | 30-day daily drift line chart |
| Per-asset drift ranking | Assets with largest drift |
| Recent alert list | Trades with drift rate > 1% |

### Aggregation & Liquidity Panel

| Metric | Description |
|--------|-------------|
| Per-chain hot wallet balances | TRON / ETH / Solana |
| Aggregation history | Recent aggregation transactions and gas costs |
| Withdrawal queue | Pending withdrawal count and amount |
| Liquidity coverage ratio | Hot wallet balance / total user liability |

## User & Order Query

| Feature | Description |
|---------|-------------|
| User position query | Includes INTERNAL / HYPERLIQUID labels |
| Order routing history | Per-order routing decision log |
| Liquidation records | Liquidation details and margin-zeroing records |
| Drift records | HL receipt vs Platform calculation discrepancy details |
| Manual balance adjustment | Requires approval workflow + full audit log |

## Role-Based Access Control (RBAC)

| Role | Permissions |
|------|-------------|
| Super Admin | Full access |
| Risk Manager | Internalization controls, exposure config, hedge params, reserve, drift thresholds |
| Operations | Read-only user data, withdrawal approval, manual aggregation trigger |
| Data Analyst | Read-only reports and data export |

## Audit Log

All admin operations must be logged:
- Operator, timestamp, IP address
- Operation type and change content
- Approval workflow records (e.g. balance adjustments, cold wallet disbursements)
- Immutable append-only: no delete or modification allowed
