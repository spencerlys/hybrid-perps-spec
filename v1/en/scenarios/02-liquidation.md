---
doc_id: scenario-en-liquidation
title: Liquidation Scenarios
tags: [liquidation, isolated, cross, margin, mark-price, L5]
version: 1.0
lang: en
updated: 2026-04-08
related-prd: [prd/06-margin-liquidation.md, prd/04-internal-execution.md, prd/05-hl-execution.md]
---

# Liquidation Scenarios

---

## SC-LQ-001: Isolated INTERNAL Position Liquidation

**Preconditions:**
- BTC INTERNAL long: size=0.1, entry_price=$80,000, leverage=10x, isolated mode
- Margin: $800; maintenance rate: 0.4% (from HL meta API)

**Liquidation Price Calculation:**
```
Liq Price = 80,000 × (1 - 800/8,000 + 0.004) = 80,000 × 0.904 = $72,320
```

**Steps:**
1. HL mark price drops to $72,300 (below liq price $72,320)
2. Platform liquidation engine detects trigger (target: < 1s)
3. Settle internally at HL mark price $72,300

**Expected Results:**
- User margin zeroed ($800)
- Client loss split: $640 (80%) → platform profit; $160 (20%) → risk reserve
- Position status → LIQUIDATED
- Liquidation notification pushed to user
- Other positions unaffected

---

## SC-LQ-002: Isolated HYPERLIQUID Position Liquidation

**Preconditions:**
- ETH HYPERLIQUID short: size=5, entry_price=$4,000, leverage=5x, isolated
- Margin: $4,000; maintenance rate: 0.5%

**Liquidation Price Calculation:**
```
Liq Price = 4,000 × (1 + 4,000/20,000 - 0.005) = 4,000 × 1.195 = $4,780
```

**Steps:**
1. HL mark price rises to $4,790 (above liq price)
2. Platform liquidation engine detects trigger
3. Send market close instruction to HL (size=5 ETH)
4. Await HL fill receipt (close_price)

**Expected Results:**
- HL returns close_price (e.g. $4,792 with slippage)
- Realized PnL computed using close_price
- If HL actual loss > Platform estimate → delta deducted from risk reserve
- User margin zeroed
- HL platform merged position reduced by corresponding ETH amount

---

## SC-LQ-003: Cross Margin — Account-Level Liquidation Trigger

**Preconditions:**
- User cross-margin account: wallet balance $5,000
  - BTC long INTERNAL: notional=$10,000; unrealized PnL=-$4,000 (BTC falling)
  - ETH short HYPERLIQUID: notional=$8,000; unrealized PnL=-$500 (ETH rising)
  - Maintenance rates: BTC 0.4%, ETH 0.5%

**Calculation:**
```
Account Equity = $5,000 + (-$4,000) + (-$500) = $500
Cross Maintenance Req = $10,000×0.4% + $8,000×0.5% = $40 + $40 = $80

Equity $500 > $80 → Not triggered yet (price continues moving...)
Equity drops to $80 → Liquidation triggered
```

**Expected Results:**
- Close most-losing position first (BTC INTERNAL, loss=-$4,000)
- BTC INTERNAL → L3 internal settlement
- Recalculate equity; if recovered > remaining maintenance req → stop
- If still insufficient → close ETH HYPERLIQUID position

---

## SC-LQ-004: Cross Margin — Sequential Close Until Account Zeroed

**Preconditions:**
- Account equity has dropped to maintenance requirement
- User holds 3 losing positions

**Steps:**
1. Close most-losing position → recalculate equity
2. Equity still ≤ maintenance req → close second position
3. Equity still ≤ maintenance req → close third position
4. All positions closed, account zeroed

**Expected Results:**
- INTERNAL positions → internal settlement, 80/20 client loss split
- HYPERLIQUID positions → send close to HL, await receipt
- Recalculate after each close before deciding to continue
- User balance zeroed; liquidation notification sent

---

## SC-LQ-005: Mixed Mode — Isolated Liquidation Does Not Affect Cross

**Preconditions:**
- User holds: BTC isolated long ($500 margin, INTERNAL) + ETH cross long (CROSS)
- Cross available balance: $8,000

**Steps:**
1. BTC price drops; isolated BTC position hits liquidation
2. BTC INTERNAL position settled at mark price internally

**Expected Results:**
- BTC isolated position → LIQUIDATED
- ETH cross position completely unaffected (separate margin pool)
- Cross available balance remains $8,000
- Only BTC liquidation notification sent

---

## SC-LQ-006: Funding Rate Erosion Raises Liquidation Price

**Preconditions:**
- BTC isolated long: entry=$80,000, margin=$800, maintenance rate=0.4%
- Held 30 days; cumulative funding paid: $200 (long paid out)

**Liquidation Price Comparison:**
```
Without funding erosion:
  Liq Price = 80,000 × (1 - 800/8,000 + 0.004) = $72,320

With funding erosion:
  Liq Price = 80,000 × (1 - (800-200)/8,000 + 0.004) = $74,320
```

**Expected Results:**
- Actual liquidation price: $74,320 ($2,000 higher than without erosion)
- User's effective margin reduced by $200 in funding payments
- System automatically updates displayed liquidation price after each funding settlement
