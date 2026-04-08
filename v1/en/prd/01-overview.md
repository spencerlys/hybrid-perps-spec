---
doc_id: prd-en-overview
title: Project Overview — Background, Goals, Scope, Phases
tags: [overview, goals, scope, phases, revenue]
version: 1.0
lang: en
updated: 2026-04-08
---

# Project Overview

## Background & Motivation

XBIT currently relies on Hyperliquid as the underlying matching and settlement engine for perpetual futures. Platform revenue is limited to a 0.02% builder fee, resulting in a very low profit ceiling.

**Core MVP Insight:** Retail traders (small orders) have a statistically negative win rate. By internalizing small orders (acting as counterparty) and forwarding large orders to Hyperliquid, XBIT captures retail losses while avoiding concentrated risk from large directional positions.

## MVP Goals

1. **Per-order routing decisions**: Order notional ≤ threshold → platform internalization; > threshold → forward to Hyperliquid
2. **Reuse all Hyperliquid data and rules**: Prices, funding rates, leverage, and listed assets fully mirrored from HL
3. **Dual position system**: A single user can simultaneously hold XBIT-internal and HL positions
4. **Unified liquidation by XBIT**: All liquidation decisions are made by XBIT regardless of position location
5. **Isolated and cross margin support**: Cross positions share account equity; unrealized PnL from one affects liquidation of others
6. **High-fidelity HL calculation replication**: XBIT's self-computed PnL, liquidation price, and funding must achieve ≥95% sync with HL

## Scope

| In Scope | Out of Scope |
|----------|-------------|
| Order notional routing engine | Custom matching engine |
| Platform B-book position management | Custom price/index price system |
| HL proxy execution and state sync | Independent market maker |
| Isolated/cross margin models | Custom funding rates, leverage, or listed assets |
| Unified liquidation engine | On-chain settlement contracts |
| Full HL calculation replication | Smart contract custody of user funds |
| Multi-chain asset aggregation (ops layer) | Options, spot, copy trading |
| 5-minute withdrawal SLA | User classification/risk profiling system |
| Exposure control and hedging | — |

## MVP Implementation Phases

| Phase | Name | Key Deliverables |
|-------|------|-----------------|
| Phase 1 | Foundation | Account system + full HL API wrapper + market data passthrough + multi-chain aggregation + withdrawal guarantee |
| Phase 2 | Routing MVP | Order notional routing + HL fill correction + drift logging (canary: all orders to HL initially) |
| Phase 3 | Go Live | B-book execution layer + unified liquidation + isolated/cross margin + HL calc replication + drift fallback |
| Phase 4 | Optimization | Routing threshold tuning + hedging optimization + drift reduction + aggregation optimization |

## Revenue Sources

| Source | Scenario | Description |
|--------|----------|-------------|
| Client loss (B-book) | INTERNAL positions, user loss/liquidation | User loss = platform profit (core revenue) |
| Builder Fee | Large orders forwarded to HL | 0.02% per trade |
| Trading fee | All orders | Taker/maker spread |
| Funding rate | INTERNAL position holders | Mirror HL rate; platform is counterparty |

## Dual Position P&L Logic

| Position Type | Liquidated By | Platform P&L |
|--------------|---------------|-------------|
| `INTERNAL` (B-book) | XBIT internal settlement | Platform earns client loss |
| `HYPERLIQUID` (HL position) | XBIT triggers → sends close order to HL | No additional client loss |

## Calculation Drift Risk

For HYPERLIQUID positions, XBIT self-computes PnL and liquidation price. If XBIT's calculation differs from HL's actual:
- XBIT calculated loss < HL actual loss → delta covered by risk reserve
- XBIT calculated loss > HL actual loss → delta becomes platform profit (or returned to user)

Calculation accuracy directly affects platform risk. Target sync rate: ≥95%.
See [05-hl-execution.md](05-hl-execution.md) and [09-settlement.md](09-settlement.md).
