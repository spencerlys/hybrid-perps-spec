---
doc_id: scenario-en-funding-settlement
title: Funding Rate Settlement Scenarios
tags: [funding-rate, settlement, internal, hyperliquid, reconciliation]
version: 1.0
lang: en
updated: 2026-04-08
related-prd: [prd/08-market-data.md, prd/09-settlement.md]
---

# Funding Rate Settlement Scenarios

Funding rate settles every 8 hours (UTC 00:00 / 08:00 / 16:00).

---

## SC-FS-001: INTERNAL Position Funding — Long Pays Short

**Preconditions:**
- User holds BTC INTERNAL long: size=0.1 BTC, mark_price=$100,000
- Current funding rate = +0.01% (positive = longs pay shorts)

**Calculation:**
```
Funding Payment = 0.1 × 100,000 × 0.0001 = $1.00
```

**Expected Results:**
- User account deducted $1.00 (long pays)
- Platform counterparty account credited $1.00
- balance_logs entry written: type = "funding_fee"
- funding_settlements table entry written
- Liquidation price recalculated (funding erodes margin; liq price moves adversely)

---

## SC-FS-002: HYPERLIQUID Position Funding — Short Receives Funding

**Preconditions:**
- User holds ETH HYPERLIQUID short: size=5 ETH, mark_price=$4,000
- Current funding rate = +0.005% (positive = longs pay, shorts receive)

**Calculation:**
```
Funding Receipt = 5 × 4,000 × 0.00005 = $1.00 (short receives)
```

**Expected Results:**
- HL settles +$1.00 on platform HL account (automatic)
- Platform mirrors same amount to user account: user available_balance +$1.00
- Platform mirror amount = HL actual settlement amount (must match exactly)
- If Platform mirror ≠ HL actual → trigger reconciliation alert

---

## SC-FS-003: Negative Funding Rate — Short Pays Long

**Preconditions:**
- BTC funding rate = -0.005% (negative = shorts pay longs)
- User holds BTC INTERNAL long: size=0.1, mark_price=$100,000

**Calculation:**
```
Long receives = 0.1 × 100,000 × 0.00005 = $0.50
```

**Expected Results:**
- User (long) available_balance +$0.50
- Platform counterparty (short) pays out $0.50
- Liquidation price moves slightly down (funding adds margin buffer for longs)

---

## SC-FS-004: Mid-Period Open — Funding Settlement Behavior

**Preconditions:**
- Funding period: UTC 00:00 ~ 08:00
- User opens BTC INTERNAL long at UTC 04:00 (mid-period)

**Expected Results:**
- Position participates in UTC 08:00 funding settlement at the full period rate
- No pro-rata calculation based on time held in period (mirrors HL behavior)
- First settlement participation: UTC 08:00

---

## SC-FS-005: Funding Reconciliation — Platform vs HL Drift Detection

**Preconditions:**
- Multiple users hold HYPERLIQUID positions
- UTC 00:00 funding settlement

**Steps:**
1. HL completes funding settlement; platform HL account changes by +$500
2. Platform mirrors funding for each user's HYPERLIQUID position using HL's published rate
3. Platform mirrored total = $498
4. Drift detected = $2 (0.4%)

**Expected Results:**
- Drift 0.4% < 1% → write deviation_log, no alert triggered
- Drift > 1% → Slack alert
- Drift > 5% → P0 emergency; halt HL routing for this asset
- Daily drift summary → reconciliation_logs
