---
doc_id: v1-en-index
title: XBIT Perpetual Futures Engine — Master Document Index
version: 1.0
lang: en
updated: 2026-04-08
---

# XBIT Perpetual Futures Engine · Master Document Index

> **Version**: v1.0 | **Status**: Active | **Language**: English (translated from Chinese) | **Updated**: 2026-04-08

Authoritative source: [../zh/index.md](../zh/index.md)

---

## Document Structure

| Directory | Purpose |
|-----------|---------|
| [`prd/`](prd/00-index.md) | Product Requirements Documents (PRD) |
| [`scenarios/`](scenarios/00-index.md) | Business Scenario Records |
| [`trading/`](trading/00-index.md) | Trading Process Flows |

---

## Core Architecture Principles

**XBIT is the sole risk controller and liquidation authority. Hyperliquid (HL) is only an execution channel.**

| Principle | Description |
|-----------|-------------|
| Dual Routing | ≤$10K → Platform internalized (INTERNAL); >$10K → Hyperliquid (HYPERLIQUID) |
| Dual Position | INTERNAL and HYPERLIQUID positions can coexist for the same user |
| Unified Liquidation | All liquidation decisions made by XBIT, regardless of position location |
| Data Reuse | Prices, funding rates, leverage, and listed assets all sourced from HL |

---

## PRD — Product Requirements

| File | Module | Phase |
|------|--------|-------|
| [00-index.md](prd/00-index.md) | PRD Master Index | — |
| [01-overview.md](prd/01-overview.md) | Project Overview | — |
| [02-routing.md](prd/02-routing.md) | L2 Order Routing Layer | Phase 2 |
| [03-account-assets.md](prd/03-account-assets.md) | L1 Account & Assets Layer | Phase 1 |
| [04-internal-execution.md](prd/04-internal-execution.md) | L3 Internal Execution Layer | Phase 3 |
| [05-hl-execution.md](prd/05-hl-execution.md) | L4 HL Proxy Execution Layer | Phase 2 |
| [06-margin-liquidation.md](prd/06-margin-liquidation.md) | L5 Margin & Liquidation Model | Phase 3 |
| [07-risk-management.md](prd/07-risk-management.md) | L6 Risk & Exposure Management | Phase 3 |
| [08-market-data.md](prd/08-market-data.md) | L7 Market Data Layer | Phase 1 |
| [09-settlement.md](prd/09-settlement.md) | L8 Settlement & Reconciliation | Phase 3 |
| [10-withdrawal-liquidity.md](prd/10-withdrawal-liquidity.md) | Withdrawal Liquidity Guarantee | Phase 1 |
| [11-admin.md](prd/11-admin.md) | L9 Admin & Operations Layer | Phase 2–3 |
| [12-infrastructure.md](prd/12-infrastructure.md) | Infrastructure | Phase 1 |

---

## Scenario Records

| File | Category |
|------|----------|
| [00-index.md](scenarios/00-index.md) | Scenario Index |
| [01-order-routing.md](scenarios/01-order-routing.md) | Order Routing Scenarios |
| [02-liquidation.md](scenarios/02-liquidation.md) | Liquidation Scenarios |
| [03-funding-settlement.md](scenarios/03-funding-settlement.md) | Funding Rate Settlement |
| [04-deposit-withdrawal.md](scenarios/04-deposit-withdrawal.md) | Deposit & Withdrawal |

---

## Trading Process Flows

| File | Process |
|------|---------|
| [00-index.md](trading/00-index.md) | Process Index |
| [01-order-lifecycle.md](trading/01-order-lifecycle.md) | Order Full Lifecycle |
| [02-internal-flow.md](trading/02-internal-flow.md) | Internal Execution Flow |
| [03-hl-proxy-flow.md](trading/03-hl-proxy-flow.md) | HL Proxy Execution Flow |
| [04-settlement-flow.md](trading/04-settlement-flow.md) | Settlement & Liquidation Flow |

---

## Glossary

| Term | Definition |
|------|------------|
| INTERNAL | Platform internalized position — not opened on HL |
| HYPERLIQUID | Position routed to and executed on Hyperliquid |
| Internalization (B-Book) | Platform acts as user's counterparty |
| Client Loss (客损) | User loss = platform's counterparty gain |
| Routing Threshold | Notional value cutoff (default $10,000) |
| Net Exposure | Platform's accumulated directional risk per asset (INTERNAL only) |
| Calculation Drift | Difference between XBIT's calculated PnL and HL's actual PnL |
| Aggregation | Moving assets from user deposit addresses to platform hot wallet |
| Hot Wallet | Online-signing wallet for daily withdrawals |
| Cold Wallet | Offline-signing wallet for large fund storage |
