---
doc_id: prd-en-hl-execution
title: L4 HL Proxy Execution Layer — Account Architecture, Position Mapping, Fill Correction
tags: [hyperliquid, execution, proxy, position-mapping, fill-price, drift, L4]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 2
---

# L4 HL Proxy Execution Layer

## HL Platform Account Architecture

### Why One Unified Account

Hyperliquid constraint: **one position per asset per direction per account.** Multiple users' BTC longs merge into a single BTC long position on HL.

| Design Decision | Choice | Reason |
|----------------|--------|--------|
| Account count | One unified platform account | HL does not support multiple positions per asset |
| HL margin mode | Cross Margin | Mandatory for merged positions; better capital efficiency |
| HL liquidation | Never allow HL to liquidate | All liquidations controlled by Platform |

### HL Account Margin Ratio Management

| Metric | Safe | Warning | Critical |
|--------|------|---------|----------|
| HL account margin ratio | > 500% | < 300% | < 150% |
| Action | Normal | Top-up from capital pool | Halt new HL opens + emergency top-up |

## HL Virtual Position Mapping

Platform maintains per-user virtual position records (HL sees a single merged position):

| Field | Description |
|-------|-------------|
| position_id | Platform internal position ID |
| user_id | User |
| symbol | Contract symbol |
| direction | User's direction |
| size | Position size |
| notional_value | size × entry_price |
| entry_price | **From HL fill receipt (fill_price)** — not estimated |
| leverage | User-selected leverage |
| margin_mode | `ISOLATED` / `CROSS` |
| isolated_margin | Isolated margin amount |
| position_source | Always `HYPERLIQUID` |
| status | `OPEN` / `CLOSED` / `LIQUIDATED` |

## HL Fill Price Correction

### Core Principle

**Platform does not use self-estimated prices as the final position record. HL's actual fill price always overrides the internal estimate.**

This is the key mechanism for eliminating open/close-phase calculation drift.

### Open Position Correction

```
1. Platform estimates cost → temporarily freezes margin using mark_price
2. Send open order to HL
3. HL returns fill receipt: fill_price, filled_size, fee
4. Replace entry_price with fill_price
5. Recompute actual frozen margin using fill_price
6. Release or deduct the delta from available_balance
```

### Close Position Correction

```
1. Platform decides to close → send market close order to HL
2. HL returns close receipt: close_price, filled_size, fee
3. Compute realized PnL using close_price (not the estimate)
4. Release margin based on close_price result
```

### Partial Fill Handling

HL market orders may fill in multiple parts:
- Record fill_price and size for each partial fill
- Final entry_price = Σ(fill_price × size) / Σ(size) (volume-weighted average)

## Calculation Drift Monitoring

### Drift Alert Thresholds

| Level | Condition | Action |
|-------|-----------|--------|
| Log | Single-trade drift > $10 | Write to deviation_logs |
| Alert | Single-trade drift rate > 1% | Slack alert |
| Critical | Single-trade drift rate > 5% | Halt HL routing for this asset + P0 alert |
| Review | Daily cumulative drift > $1,000 | Manual risk review |

### Drift Fallback

```
HL actual loss > Platform calculated loss:
  → delta = HL_actual_loss - Platform_calculated_loss
  → Deduct delta from risk reserve
  → User settled at Platform's calculated value (no user impact)

HL actual loss < Platform calculated loss:
  → Delta becomes platform profit (or returned to user — business decision)
```

## HL Consistency Check (Every 5 Minutes)

```
Σ(user HYPERLIQUID virtual position sizes) == HL account actual position size

Deviation > 0.01% → Alert
Deviation > 0.1%  → P0 emergency — halt new HL opens, trigger manual review
```
