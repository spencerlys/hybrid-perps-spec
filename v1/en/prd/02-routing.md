---
doc_id: prd-en-routing
title: L2 Order Routing Layer — Rules, Threshold, Close Routing
tags: [routing, order, threshold, hyperliquid, internal, L2]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 2
---

# L2 Order Routing Layer

## Routing Decision Logic

```
Notional Value = Order Quantity × Current HL Mark Price

Notional Value ≤ Routing Threshold (default $10,000)
  └─► Platform Internalization (L3 Internal Execution)

Notional Value > Routing Threshold
  └─► Forward to Hyperliquid (L4 Proxy Execution)
```

Routing decision latency target: **< 5ms P99**

## Routing Rule Details

### Threshold Configuration
- Default global threshold: $10,000
- Per-asset threshold override supported (e.g. BTC $10K, DOGE $5K)
- Hot-reload via admin panel — no service restart required
- Each order evaluated independently; no cumulative tracking

### Multiple Opens — Same Asset & Direction

> Threshold $10,000:
> - BTC long $5,000 → INTERNAL
> - BTC long $15,000 → HYPERLIQUID
>
> Result: User holds two independent BTC long positions, each with its own entry price.
> - Isolated mode: each position liquidated independently
> - Cross mode: both positions share account equity; liquidation computed at account level

### Force-Route to HL (even if notional ≤ threshold)

| Condition | Description |
|-----------|-------------|
| Asset net exposure at limit | Stop new internalization for this asset |
| Volatility spike (>5%/hour) | No internalization during extreme price moves |
| HL latency exceeded (>500ms) | Conservative fallback to HL |
| Manual admin override | One-click halt of all internalization |

## Close Order Routing

**Close order must follow the original position source:**
- INTERNAL position → close internally only (never via HL)
- HYPERLIQUID position → send close order to HL only

Cross-system offset is forbidden: an INTERNAL position cannot offset a HYPERLIQUID position.

## Cancel Order Routing

| Order Type | Handling |
|------------|----------|
| INTERNAL pending limit order | Remove from internal queue; release frozen margin |
| HYPERLIQUID pending limit order | Send cancel instruction to HL |

## Routing Transparency

Users cannot detect routing differences:
- Same price
- Same receipt format
- Same funding rate
- Same liquidation parameters

## Routing Log

Every routing decision must be logged:
- Order ID
- Notional value
- Route result (INTERNAL / HYPERLIQUID)
- Decision latency (ms)
- Trigger condition (threshold rule / force-HL reason)
