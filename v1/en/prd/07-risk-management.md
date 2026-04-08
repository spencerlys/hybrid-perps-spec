---
doc_id: prd-en-risk-management
title: L6 Risk & Exposure Management — Net Exposure, Hedging, Risk Reserve
tags: [risk, exposure, hedge, reserve, monitoring, L6]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 3
---

# L6 Risk & Exposure Management

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
| Drift fallback deduction | Deducted when HL actual loss > XBIT calculated loss |
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

## Daily Loss Circuit Breaker

```
Daily INTERNAL net loss > $100K → Alert
Daily INTERNAL net loss > $500K → Halt all internalization (route everything to HL)

Daily Net Loss = Σ(user gains) - Σ(user losses) (INTERNAL positions, current day)
```
