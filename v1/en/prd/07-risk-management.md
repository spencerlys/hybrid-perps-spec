---
doc_id: prd-en-risk-management
title: L6 Risk & Exposure Management — Net Exposure, Routing Mode Linkage, Hedging, Risk Reserve
tags: [risk, exposure, hedge, reserve, monitoring, routing-mode, L6]
version: 1.2
lang: en
updated: 2026-04-09
phase: Phase 2
---

# L6 Risk & Exposure Management

> **Phase Adjustment Note (v1.2)**: Net exposure monitoring, hedging engine, and risk reserve are advanced from Phase 3 to Phase 2.
> Rationale: Once the Phase 2 routing engine goes live, INTERNAL positions generate net exposure immediately, requiring concurrent hedging capability.
> Routing mode auto-switching and full risk control dashboard remain in Phase 3.

## Responsibility Boundary with L3 Internal Order Book

**L6 hedging engine and L3 internal limit order queue are completely independent modules:**

- **L3 Internal Order Book**: Manages pending limit order queues for INTERNAL positions, triggers fills at HL prices. Platform is always the counterparty; no user-to-user matching occurs.
- **L6 Hedging Engine**: Independent of the order book. When net exposure exceeds thresholds, opens reverse positions on HL to reduce risk. Hedge instructions route through L4.

Flow: Order → L3 internal accounting → L6 real-time net exposure aggregation → exceeds threshold → L6 triggers hedge → L4 proxies to HL

## Net Exposure (B-Book)

**Net Exposure = Platform's accumulated directional risk from internalized orders.**

### Formula (INTERNAL positions only)

```
BTC Net Exposure = Σ(user BTC long INTERNAL notional)
                - Σ(user BTC short INTERNAL notional)
```

- Positive → Platform is net short (loss if BTC rises)
- Negative → Platform is net long (loss if BTC falls)
- Near zero → Balanced book, low risk

> HYPERLIQUID positions are excluded — their P&L flows through HL, no directional risk to the platform.

## Net Exposure & Routing Mode Linkage

Real-time net exposure drives routing mode switch recommendations. Thresholds are configurable in the admin panel (see [11-admin.md](11-admin.md)).

| Net Exposure (absolute value) | System Action |
|------------------------------|--------------|
| ≤ Betting mode trigger threshold (default $50K) | Recommend switch to BETTING_MODE / auto-switch if enabled |
| Between the two thresholds | Stay in NORMAL_MODE; no action |
| ≥ HL mode trigger threshold (default $800K) | Recommend switch to HL_MODE / auto-switch if enabled + hedge reminder |

> **Auto-switch is off by default.** Admin must explicitly enable it. When enabled, the system automatically switches mode when thresholds are crossed and sends a notification.

### Hedge Reminder in HL Mode

When HL_MODE is activated, the admin panel displays a prominent alert:
> ⚠️ Net exposure has reached the high-risk threshold. System has switched to Hyperliquid Mode. Please hedge on Hyperliquid immediately!

A Slack notification is also sent to the Risk Manager with the current net exposure value and recommended hedge size.

## Hedging Strategy

| Single-Asset Net Exposure | Hedge Action |
|--------------------------|-------------|
| < $100,000 | No hedge |
| $100K – $500K | Hedge 50% on HL |
| $500K – $1M | Hedge 80% on HL |
| > $1M | Stop internalization + hedge 80% on HL |

### Hedge Position Design
- Use **low leverage (2x–3x)** to avoid hedge position liquidation
- Batch submission (batchModify) to reduce slippage
- Tracked independently; auto-reduce when exposure decreases
- Hedge costs (fee + funding) charged to risk management cost center

## Risk Reserve Fund

### Funding Sources

| Source | Description |
|--------|-------------|
| Platform initial injection | Pre-launch capital; recommend ≥ $500K |
| Client loss allocation | 20% of each INTERNAL client loss auto-transferred |
| Drift fallback deduction | Deducted when HL actual loss > Platform calculated loss |
| Periodic top-up | Replenished from operating profit when below safety threshold |

### Client Loss Allocation

```
Per INTERNAL client loss:
  80% → Platform profit
  20% → Risk reserve
(Ratio configurable in admin panel)
```

### Reserve Thresholds

| Balance | Action |
|---------|--------|
| ≥ $500K | Normal operations |
| $200K – $500K | Reduce internalization (lower routing threshold) |
| < $200K | Halt all internalization |

## Risk Monitoring Metrics

| Metric | Alert | Critical | Response |
|--------|-------|----------|----------|
| Single-asset net exposure | > $500K | > $1M | Stop internalization + hedge |
| Total internalization exposure | > $2M | > $5M | Reduce internalization |
| Daily net loss | > $100K | > $500K | Pause internalization |
| Risk reserve balance | < $500K | < $200K | Reduce/halt internalization |
| HL account margin ratio | < 300% | < 150% | Top-up / halt HL opens |
| HL channel latency | > 200ms | > 500ms | Pause internalization |
| Single-trade drift rate | > 1% | > 5% | Alert / halt HL for this asset |
| Daily cumulative drift | > $1,000 | > $5,000 | Manual risk review |

## Natural Hedging Effect

In the Platform B-book model, when multiple users trade long and short the same asset simultaneously, the platform's net exposure naturally approaches zero:

```
User A: buys BTC $50,000 (INTERNAL) → Platform holds BTC short $50K
User B: sells BTC $50,000 (INTERNAL) → Platform holds BTC long $50K
Net Exposure = $50K - $50K = $0 (long/short fully offset)
```

This is the core profitability condition for the B-book model:
- When balanced, platform has zero directional risk but still earns trading fees and funding fees
- When users collectively lose (statistical edge), platform captures client loss revenue
- The hedging engine only intervenes when positions become imbalanced (net exposure accumulates)

## Daily Loss Circuit Breaker

```
Daily INTERNAL net loss > $100K → Alert
Daily INTERNAL net loss > $500K → Halt all internalization (route everything to HL)

Daily Net Loss = Σ(user gains) - Σ(user losses) (INTERNAL positions, current day)
```

## Phase Delivery Schedule

| Deliverable | Phase |
|------------|-------|
| Real-time net exposure calculation | Phase 2 |
| Hedging engine (manual + auto-trigger) | Phase 2 |
| Risk reserve basic management | Phase 2 |
| Risk control monitoring metrics + daily loss circuit breaker | Phase 2 |
| Routing mode auto-switching | Phase 3 |
| Full risk control dashboard (Grafana panels) | Phase 3 |
| Hedging strategy optimization (time-sliced execution, slippage optimization) | Phase 4 |
