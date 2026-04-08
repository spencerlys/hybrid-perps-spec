# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This is the **Platform Perpetual Futures Trading Engine** — currently in the specification phase. The full MVP design spec is at `docs/superpowers/specs/2026-03-28-perpengine-design.md`. No implementation code exists yet.

## What This System Is

Platform is a perpetual futures trading platform that sits in front of Hyperliquid (HL). The core strategy:
- **Small orders (≤$10K notional)**: Platform acts as counterparty ("internalized"). Users trade against Platform's own capital. User losses = platform revenue.
- **Large orders (>$10K notional)**: Forwarded to Hyperliquid. Platform earns builder fees.

Platform is the sole risk controller and liquidation authority. HL is only an execution channel.

## System Architecture (9 Layers)

| Layer | Name | Role |
|-------|------|------|
| L1 | Account & Asset | User registration, wallets, deposits/withdrawals |
| L2 | Order Routing | Route by notional value threshold to internal or HL |
| L3 | Platform Prop Trading | Internal position management, counterparty accounting |
| L4 | HL Proxy Execution | API wrapper for large orders, sync HL position state |
| L5 | Margin & Liquidation | Isolated/cross margin calculations, unified liquidation engine |
| L6 | Risk Control | Net exposure tracking, auto-hedging, risk reserve management |
| L7 | Market Data | Real-time HL data via WebSocket (prices, funding, order book) |
| L8 | Settlement | PnL calculation, cross-system fund reconciliation |
| L9 | Backend Management | Admin dashboard, risk config, operational controls |

## Critical Design Decisions

### Dual Position System
Users can hold positions simultaneously in two systems: **INTERNAL** (small orders, platform counterparty) and **HYPERLIQUID** (large orders, HL executes). Each order and position must always carry a `route` tag indicating which system owns it.

### HL Account Architecture
- Platform uses a **single unified HL account** (HL's technical constraint: same asset + direction merges into one position)
- Cross-margin mode with ≥$500K platform capital
- Must maintain >500% margin ratio on HL account — HL liquidation of this account is a critical failure scenario to prevent

### Margin Models
- **Isolated (逐仓)**: Each position has its own collateral; independent liquidation
  - Liquidation price (long): `Entry × (1 - Margin/Notional + Maintenance Rate)`
- **Cross (全仓)**: All cross-margin positions share account balance; account-level liquidation when `Account Equity ≤ Total Maintenance Margin Requirement`
- Mixed mode is supported: user can have some isolated and some cross positions simultaneously

### Liquidation Control
Regardless of position location (INTERNAL or HYPERLIQUID), Platform always:
1. Detects liquidation trigger using HL mark prices (<1s latency required)
2. For INTERNAL positions: settle internally (platform captures user loss)
3. For HYPERLIQUID positions: send market close order to HL

### Routing Rules (L2)
Default threshold: $10,000 notional (`quantity × HL mark price`). Force-route to HL when:
- Volatility >5%/hour for the asset
- Single-asset net internal exposure at limit
- HL connection latency >500ms
- Manual admin override active

### Risk Reserve Fund
- Funded by 20% of all user losses from internal trading
- Thresholds:
  - ≥$500K: Normal operations
  - $200K–$500K: Reduce internal trading
  - <$200K: Halt all internal trading

### Hedging (L6)
Triggered by net internal exposure: `Net Exposure = Σ(user long INTERNAL) - Σ(user short INTERNAL)`

| Net Exposure | Action |
|--------------|--------|
| < $100K | No hedge |
| $100K–$500K | 50% hedge on HL |
| $500K–$1M | 80% hedge on HL |
| > $1M | 80% hedge + stop new internal orders |

## Data Sources

All market data comes from Hyperliquid:
- Prices, mark prices, funding rates, order book: WebSocket subscriptions
- Instrument metadata (leverage/margin rates): REST, synced hourly

Resilience requirements: exponential backoff on disconnect; pause internal trading if latency >500ms.

## Performance Targets

| Operation | Latency Target |
|-----------|----------------|
| Route decision | <5ms P99 |
| HL order forwarding | <50ms |
| Internal order execution | <10ms |
| Liquidation detection | <1s |
| API response | <100ms P95 |

## MVP Implementation Phases

1. **Phase 1**: Account system + full HL API wrapper + market data passthrough
2. **Phase 2**: Routing engine + canary testing (forward all orders to HL initially)
3. **Phase 3**: Internal execution engine + unified liquidation + margin models
4. **Phase 4**: Route threshold tuning + hedging optimization

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
- **Risk Manager**: Internal/hedge controls, exposure config, reserve management
- **Operations**: Read-only user queries
- **Data Analyst**: Read-only reporting
