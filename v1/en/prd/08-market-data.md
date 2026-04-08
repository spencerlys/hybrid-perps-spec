---
doc_id: prd-en-market-data
title: L7 Market Data Layer — HL Data Passthrough, Precision Sync, Exception Handling
tags: [market-data, websocket, price, funding-rate, mark-price, precision, L7]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 1
---

# L7 Market Data Layer

## Data Sources

All market data in MVP comes from Hyperliquid. XBIT adapts format and passes through.

## Real-Time Data (WebSocket)

| Data Type | HL Channel | Frequency |
|-----------|-----------|-----------|
| Real-time price / mark price | `allMids` | Real-time |
| Order book depth | `l2Book` | Real-time |
| Candlestick / K-line | `candle` | Per interval |
| Funding rate | `activeAssetCtx` | Real-time |
| Recent trades | `trades` | Real-time |
| User position events | `user` | Event-driven |

### WebSocket Multiplexing
- Upstream: single HL WebSocket connection
- Downstream: serve multiple XBIT clients
- Reference-counted subscriptions: upstream subscription cancelled only when last client disconnects
- Heartbeat + auto-reconnect with exponential backoff (max 30s)

## Static Data (REST Sync)

| Data Type | Source | Sync Frequency |
|-----------|--------|----------------|
| Contract metadata (listed assets, leverage range) | HL `/info` | Hourly |
| Precision rules (szDecimals, size step) | HL `/info` | Hourly |
| Maintenance margin rate tiers | HL `/info` | Hourly |
| Predicted funding rate | HL REST | Hourly |
| Current funding rate | HL REST / WS | Real-time |

### Precision Rule Sync

HL `/info` metadata contains:
- `szDecimals`: price/quantity decimal places per asset
- `maxLeverage`: maximum leverage per asset
- `marginTableTiers`: maintenance rate tier table (by notional value bracket)

Usage:
1. L5 liquidation price calculation (tiered maintenance rate)
2. Order quantity/price rounding (align with HL)
3. Minimum order size validation

## HL Mark Price

**XBIT does not compute mark price. It uses HL's pushed mark price directly.**

- HL mark price = oracle_price × (1 + impact funding premium)
- Received via `allMids` WebSocket subscription
- Used for: unrealized PnL, liquidation detection, margin ratio calculation

### Mark Price Latency Monitoring
```
End-to-end latency = HL generation → XBIT receive → user display

Alert: > 200ms
Critical: > 500ms → Pause internalization (no internalization during data lag)
```

## Funding Rate Pass-Through
- Source: HL, never self-computed
- Settlement period: every 8 hours (UTC 00:00, 08:00, 16:00)
- INTERNAL positions: mirror HL rate; XBIT platform is the counterparty
- HYPERLIQUID positions: HL settles on platform account; XBIT mirrors amount to user

See [09-settlement.md](09-settlement.md) for full funding settlement formulas.

## Exception Handling

### HL Disconnect
```
Disconnect → Immediate alert
Reconnect: exponential backoff (1s, 2s, 4s, ... max 30s)
During disconnect:
  - Pause internalization (no real-time price = no safe B-booking)
  - Pause HL order forwarding
  - Users see "data loading"
After recovery: full snapshot refresh, then incremental updates
```

### Abnormal Price Jump
```
Detect: single update > configured threshold (e.g. BTC 5% in one update)
Handle: use last valid price
Notify: alert + pause internalization
Recover: resume after N consecutive normal updates
```

### Data Consistency Check
- Sequence number check: gap detected → trigger full snapshot rebuild
- Periodic comparison: every 5 minutes compare XBIT cached prices vs HL REST API
