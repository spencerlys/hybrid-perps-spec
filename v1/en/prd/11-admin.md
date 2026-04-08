---
doc_id: prd-en-admin
title: L9 Admin & Operations Layer — Routing Config, Risk Dashboard, RBAC
tags: [admin, config, dashboard, rbac, operations, L9]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 2-3
---

# L9 Admin & Operations Layer

## Routing & Internalization Configuration

| Feature | Description |
|---------|-------------|
| Global routing threshold | Default $10,000; hot-reload, no restart needed |
| Per-asset threshold override | Independent setting per contract symbol |
| Internalization whitelist | Recommend BTC/ETH only at launch |
| Emergency internalization switch | One-click halt of all B-book activity |
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
| Daily drift total | Cumulative XBIT calc vs HL actual delta |
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
| Drift records | HL receipt vs XBIT calculation discrepancy details |
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
