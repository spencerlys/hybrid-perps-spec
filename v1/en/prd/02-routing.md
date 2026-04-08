---
doc_id: prd-en-routing
title: L2 Order Routing Layer — Three-Mode Routing, Threshold, Close Routing
tags: [routing, order, threshold, hyperliquid, internal, routing-mode, L2]
version: 1.1
lang: en
updated: 2026-04-08
phase: Phase 2
---

# L2 Order Routing Layer

## Routing Mode Overview

The system supports three routing modes, controlled by admin via the admin panel (see [11-admin.md](11-admin.md)).

| Mode | Trigger Scenario | Routing Behavior |
|------|-----------------|-----------------|
| `HL_MODE` (Hyperliquid Mode) | Net exposure too large, high risk | All new opens → HL; no internalization |
| `NORMAL_MODE` (Default) | Net exposure within acceptable range | Route by standard threshold |
| `BETTING_MODE` (Aggressive Mode) | Net exposure very low, book balanced | Route by elevated betting threshold |

## Routing Decision Logic

```
Notional Value = Order Quantity × Current HL Mark Price

switch (current routing mode):

  case HL_MODE:
    → All new open orders → HYPERLIQUID (regardless of size)

  case NORMAL_MODE:
    if Notional Value ≤ Normal Mode Threshold (default $10,000)
      → INTERNAL (platform internalization)
    else
      → HYPERLIQUID

  case BETTING_MODE:
    if Notional Value ≤ Betting Mode Threshold (default $50,000)
      → INTERNAL (platform internalization)
    else
      → HYPERLIQUID
```

Routing decision latency target: **< 5ms P99**

## Routing Rule Details

### Threshold Configuration

| Setting | Default | Applies To |
|---------|---------|-----------|
| Normal mode routing threshold | $10,000 | NORMAL_MODE |
| Betting mode routing threshold | $50,000 | BETTING_MODE |

- Per-asset threshold override supported (overrides the global value)
- Hot-reload via admin panel — no service restart required
- Each order evaluated independently; no cumulative tracking

### Multiple Opens — Same Asset & Direction

> Normal mode, threshold $10,000:
> - BTC long $5,000 → INTERNAL
> - BTC long $15,000 → HYPERLIQUID
>
> Result: User holds two independent BTC long positions, each with its own entry price.
> - Isolated mode: each position liquidated independently
> - Cross mode: both positions share account equity; liquidation computed at account level

### Force-Route to HL (all modes, highest priority)

| Condition | Description |
|-----------|-------------|
| Volatility spike (>5%/hour) | No internalization during extreme price moves |
| HL latency exceeded (>500ms) | Conservative fallback to HL |

> Note: Routing modes already cover "net exposure at limit" and "manual halt" scenarios — these are no longer listed as independent force-route conditions.

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
- Current routing mode (`routing_mode`: HL_MODE / NORMAL_MODE / BETTING_MODE)
- Threshold used
- Decision latency (ms)
- Trigger condition (threshold rule / force-HL reason)
