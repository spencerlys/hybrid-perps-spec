---
doc_id: prd-en-index
title: Perp Engine PRD v1.0 — Document Index
tags: [index, overview, prd]
version: 1.0
lang: en
updated: 2026-04-08
---

# Perp Engine PRD v1.0 — Product Requirements Index

> **v1.0** | Translated from authoritative Chinese source: [../../zh/prd/00-index.md](../../zh/prd/00-index.md)

Each file covers one independent module and can be read standalone.

## Document Structure

| File | Module | Core Content | Phase |
|------|--------|-------------|-------|
| [01-overview.md](01-overview.md) | Project Overview | Background, goals, scope, phases, revenue model | — |
| [02-routing.md](02-routing.md) | L2 Order Routing | Routing rules, thresholds, force-HL conditions | Phase 2 |
| [03-account-assets.md](03-account-assets.md) | L1 Account & Assets | Registration, deposits/withdrawals, multi-chain aggregation | Phase 1 |
| [04-internal-execution.md](04-internal-execution.md) | L3 Internal Execution | Internal position model, counterparty accounting, execution price | Phase 3 |
| [05-hl-execution.md](05-hl-execution.md) | L4 HL Proxy Execution | Account architecture, position mapping, fill correction, drift monitoring | Phase 2 |
| [06-margin-liquidation.md](06-margin-liquidation.md) | L5 Margin & Liquidation | Isolated/cross formulas, HL precision replication, liquidation engine | Phase 3 |
| [07-risk-management.md](07-risk-management.md) | L6 Risk Management | Net exposure, hedging strategy, risk reserve | Phase 2 |
| [08-market-data.md](08-market-data.md) | L7 Market Data | HL data passthrough, precision rule sync, exception handling | Phase 1 |
| [09-settlement.md](09-settlement.md) | L8 Settlement | PnL formulas, funding settlement, drift fallback, three-way reconciliation | Phase 3 |
| [10-withdrawal-liquidity.md](10-withdrawal-liquidity.md) | Withdrawal Liquidity | 5-min SLA, tiered processing, liquidity monitoring | Phase 1 |
| [11-admin.md](11-admin.md) | L9 Admin | Routing config, risk dashboard, RBAC | Phase 2–3 |
| [12-infrastructure.md](12-infrastructure.md) | Infrastructure | Database, observability, security, deployment, performance | Phase 1 |

## Core Architecture Principles

**Platform is the sole risk controller and liquidation authority. HL is only an execution channel.**

- Platform maintains complete position records per user (both INTERNAL and HYPERLIQUID)
- Platform uses HL mark prices to compute margin ratios and liquidation conditions
- INTERNAL positions: settled internally; platform earns client losses
- HYPERLIQUID positions: Platform triggers timing → sends close order to HL
- Platform HL account always maintains excess margin to prevent HL from liquidating it

## Glossary

| Term | Definition |
|------|------------|
| INTERNAL | Platform internalized position — not opened on HL |
| HYPERLIQUID | Position routed to HL for execution |
| Internalization | Platform acts as user's direct counterparty |
| Client Loss | User loss = platform's counterparty gain (core B-book revenue) |
| Routing Threshold | Notional value cutoff (default $10,000 per order) |
| Net Exposure | Platform's accumulated directional risk per asset (INTERNAL only) |
| Calculation Drift | Delta between Platform's computed PnL and HL's actual PnL |
| Aggregation | Moving assets from user deposit addresses into platform hot wallet |
| Hot Wallet | Online-signing wallet for daily withdrawal processing |
| Cold Wallet | Offline-signing wallet for secure storage of large balances |
