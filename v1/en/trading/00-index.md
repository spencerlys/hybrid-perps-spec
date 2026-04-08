---
doc_id: trading-en-index
title: Trading Process Flow Index
version: 1.0
lang: en
updated: 2026-04-08
---

# Trading Process Flows

> Parent index: [../index.md](../index.md)
> Authoritative source: [../../zh/trading/00-index.md](../../zh/trading/00-index.md)

These documents describe the complete data flows and processing steps between internal system modules, focusing on **module interaction sequence** and **data propagation paths**.

## Process Files

| File | Process | Key Modules |
|------|---------|-------------|
| [01-order-lifecycle.md](01-order-lifecycle.md) | Order Full Lifecycle | L1, L2, L3/L4, L5, L8 |
| [02-internal-flow.md](02-internal-flow.md) | Internal Execution Flow | L2 → L3 → L8 |
| [03-hl-proxy-flow.md](03-hl-proxy-flow.md) | HL Proxy Execution Flow | L2 → L4 → L8 |
| [04-settlement-flow.md](04-settlement-flow.md) | Settlement & Liquidation Flow | L5 → L3/L4 → L8 |

## System Layer Diagram

```
User Request
  │
  ▼
L1 Account & Assets (balance check, margin validation)
  │
  ▼
L2 Order Routing (< 5ms routing decision)
  │
  ├─ INTERNAL ──► L3 Internal Execution Layer (< 10ms)
  │                    │
  └─ HYPERLIQUID ──► L4 HL Proxy Execution Layer (< 50ms)
                         │
  ◄─────────────────────┤ HL fill receipt (fill_price)
  │
  ▼
L5 Margin & Liquidation Engine (real-time monitoring of all positions)
  │
  ▼
L6 Risk & Exposure Management (net exposure aggregation + hedge orders)
  │
  ▼
L7 Market Data Layer (HL mark price → all layers)
  │
  ▼
L8 Settlement & Reconciliation (PnL, funding, three-way reconciliation)
  │
  ▼
L9 Admin & Operations (config, dashboard, audit)
```

## Key Data Flows

| Data | Flow Direction | Frequency |
|------|---------------|-----------|
| HL mark price | L7 → L5 (liquidation detection) | Real-time |
| HL mark price | L7 → L3 (execution price reference) | Real-time |
| HL fill price | L4 → position record (entry_price correction) | Per fill |
| Liquidation signal | L5 → L3 / L4 | Event-driven |
| Net exposure | L3 → L6 | Real-time aggregate |
| Hedge order | L6 → L4 | Threshold-triggered |
