# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This is the **Platform Perpetual Futures Trading Engine** — currently in the specification phase. No implementation code exists yet.

**Design model: 方案二（内部对赌 + 阈值对冲 / Order-level Internalization + Threshold Hedging）**

- NOT a matching engine (撮合系统 / 方案三) — there is no user-to-user order matching
- Platform is the sole counterparty for all INTERNAL positions (B-book model)
- Orders are selectively hedged on Hyperliquid when net exposure exceeds thresholds

## Document Structure

```
v1/
├── zh/                      # 中文（权威版本）
│   ├── index.md             # 文档总索引
│   ├── prd/                 # 产品需求文档（12 个模块）
│   ├── scenarios/           # 业务场景记录（23 个场景）
│   └── trading/             # 交易处理流程（4 个流程）
└── en/                      # English（翻译版）
mindmap/
└── index.html               # 全模块架构脑图（浏览器打开）
```

Chinese version is authoritative. In case of discrepancies, Chinese prevails.

## What This System Is

Platform is a perpetual futures trading platform that sits in front of Hyperliquid (HL). The core strategy:
- **Small orders (≤$10K notional)**: Platform acts as counterparty ("internalized"). Users trade against Platform's own capital. User losses = platform revenue.
- **Large orders (>$10K notional)**: Forwarded to Hyperliquid. Platform earns builder fees.

Platform is the sole risk controller and liquidation authority. HL is only an execution channel.

## System Architecture (R0 + 9 Layers)

| Layer | Name | Phase | Role |
|-------|------|-------|------|
| R0 | API Relay | Phase 1 | HL REST/WS proxy, format adaptation, transparent migration |
| L1 | Account & Asset | Phase 1 | User registration, wallets, deposits/withdrawals, multi-chain aggregation |
| L2 | Order Routing | Phase 2 | Three-mode routing (HL_MODE / NORMAL_MODE / BETTING_MODE), threshold decisions |
| L3 | Platform B-book Execution | Phase 3 | Internal position management, counterparty accounting (NOT matching engine) |
| L4 | HL Proxy Execution | Phase 2 | API wrapper for large orders, fill price correction, position mapping |
| L5 | Margin & Liquidation | Phase 3 | Isolated/cross margin calculations, unified liquidation engine |
| L6 | Risk & Hedge Engine | Phase 2 | Net exposure monitoring, hedge engine, risk reserve management |
| L7 | Market Data | Phase 1 | Real-time HL data via WebSocket (prices, funding, order book) |
| L8 | Settlement | Phase 3 | PnL calculation, funding rate settlement, cross-system reconciliation |
| L9 | Backend Management | Phase 2–3 | Admin dashboard, routing mode controls, risk config, RBAC |

## Critical Design Decisions

### Design Model — 方案二, NOT 方案三

This system implements **方案二：内部对赌 + 阈值对冲** (Order-level Internalization):
- All orders default to internal counterparty (B-book)
- When net exposure exceeds thresholds → hedge on HL (A-book)
- **No matching engine** — no user-to-user order matching exists in MVP scope
- Matching engine (撮合引擎) is a future upgrade path (方案三), not current scope

### Dual Position System
Users can hold positions simultaneously in two systems: **INTERNAL** (small orders, platform counterparty) and **HYPERLIQUID** (large orders, HL executes). Each order and position must always carry a `route` tag indicating which system owns it.

### Three-Mode Routing (L2)

| Mode | Trigger | Behavior |
|------|---------|----------|
| `HL_MODE` | Net exposure too high | All new orders → HL |
| `NORMAL_MODE` (default) | Normal conditions | ≤$10K → INTERNAL; >$10K → HL |
| `BETTING_MODE` | Net exposure very low | ≤$50K → INTERNAL; >$50K → HL |

Force-route to HL when: volatility >5%/hour, HL latency >500ms.

### HL Account Architecture
- Platform uses a **single unified HL account** (HL's technical constraint: same asset + direction merges into one position)
- Cross-margin mode with ≥$500K platform capital
- Must maintain >500% margin ratio on HL account — HL liquidation of this account is a critical failure scenario to prevent

### Order Book vs Hedge System (Separate Concerns)

- **L3 Internal Order Book**: Only manages INTERNAL limit order queue, triggers execution when HL price reaches limit price. This is NOT a matching engine — platform is always the counterparty.
- **L6 Hedge Engine**: Independent module. When net exposure exceeds thresholds, sends hedge orders to HL via L4. Completely separate from L3's limit order queue.

Flow: **Order → L3 internal accounting → L6 aggregates net exposure → threshold exceeded → L6 triggers hedge → hedge order via L4 to HL**

### Margin Models
- **Isolated (逐仓)**: Each position has its own collateral; independent liquidation
- **Cross (全仓)**: All cross-margin positions share account balance; account-level liquidation when `Account Equity ≤ Total Maintenance Margin Requirement`
- Mixed mode is supported: user can have some isolated and some cross positions simultaneously

### Liquidation Control
Regardless of position location (INTERNAL or HYPERLIQUID), Platform always:
1. Detects liquidation trigger using HL mark prices (<1s latency required)
2. For INTERNAL positions: settle internally (platform captures user loss)
3. For HYPERLIQUID positions: send market close order to HL

### Hedging (L6)
Triggered by net internal exposure: `Net Exposure = Σ(user long INTERNAL) - Σ(user short INTERNAL)`

| Net Exposure | Action |
|--------------|--------|
| < $100K | No hedge |
| $100K–$500K | 50% hedge on HL |
| $500K–$1M | 80% hedge on HL |
| > $1M | 80% hedge + stop new internal orders |

### Risk Reserve Fund
- Funded by 20% of all user losses from internal trading
- Thresholds: ≥$500K normal; $200K–$500K reduce; <$200K halt all internal trading

## Performance Targets

| Operation | Latency Target |
|-----------|----------------|
| Route decision | <5ms P99 |
| HL order forwarding | <50ms |
| Internal order execution | <10ms |
| Liquidation detection | <1s |
| API response | <100ms P95 |

## MVP Implementation Phases

| Phase | Name | Core Deliverables |
|-------|------|-------------------|
| Phase 1 | Foundation | Account system + HL API relay (R0) + market data passthrough + multi-chain deposit + withdrawal guarantee |
| Phase 2 | Routing + Hedging | Order routing engine + HL proxy execution + **hedge engine + net exposure monitoring + risk reserve** (canary: all orders to HL) |
| Phase 3 | B-book Live | Internal execution engine + unified liquidation + margin models + HL calculation replication + deviation handling |
| Phase 4 | Optimization | Route threshold tuning + hedge strategy optimization + large order splitting + cross-threshold routing (reserved) |

**Key: Hedge engine is Phase 2, not Phase 3. Routing creates exposure; exposure requires hedging.**

## Data Sources

All market data comes from Hyperliquid:
- Prices, mark prices, funding rates, order book: WebSocket subscriptions
- Instrument metadata (leverage/margin rates): REST, synced hourly

Resilience requirements: exponential backoff on disconnect; pause internal trading if latency >500ms.

## Observability Stack

Prometheus + Grafana for metrics; ELK for logs. P0 alerts require 5-minute response.

## Deposit Chains (MVP)

| Chain | Confirmations | Assets |
|-------|--------------|--------|
| TRON | 19 | USDT (TRC20) |
| Ethereum | 12 | USDT/USDC (ERC20) |
| Solana | 32 slots | USDT/USDC (SPL) |

## Admin Role Permissions

- **Super Admin**: Full access
- **Risk Manager**: Internal/hedge controls, exposure config, reserve management, routing mode switching
- **Operations**: Read-only user queries, withdrawal approval
- **Data Analyst**: Read-only reporting
