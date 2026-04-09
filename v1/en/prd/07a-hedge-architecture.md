---
doc_id: prd-en-hedge-architecture
title: L6 Hedge System Architecture Design — Account Model, Capital Efficiency, Pre-hedge Assessment, Edge Cases
tags: [hedge, architecture, hl-account, capital-efficiency, pre-hedge, L6]
version: 1.0
lang: en
updated: 2026-04-09
phase: Phase 2
---

# L6 Hedge System Architecture Design

> This document supplements [07-risk-management.md](07-risk-management.md), focusing on the underlying architecture design of the hedge system.

---

## I. The Essence of Hedging: What is the Platform Doing?

When a user goes long BTC (INTERNAL), the platform as counterparty automatically holds a BTC short position. If BTC rises, the platform loses money.

**Hedging = Platform goes to HL and opens a position opposite to the platform's internal position to offset risk.**

```
User goes long BTC $500K (INTERNAL)
  → Platform internally holds BTC short $500K (internalization counterparty)
  → BTC rises → Platform loses
  → Hedge action: Go long BTC $400K on HL (hedge 80%)
  → BTC rises → Platform earns on HL, loses internally → Offsets
```

That is: **Hedge direction = User direction (= Opposite of platform's internal exposure)**.

---

## II. HL Account Model: Single Account vs. Multiple Accounts

### Current Plan: Single Unified Account

HL technical constraint: Same account + same asset + same direction = only one position possible.

```
Platform HL Account
├── BTC long $400K (hedge position)
├── ETH short $200K (hedge position)
├── BTC long $50K (user large order routed) ← Auto-merges with hedge position!
└── Cross-margin mode, shares $500K margin
```

**Problem**: User orders routed to HL merge with hedge positions on HL (same asset, same direction), cannot be managed separately.

### Recommended Plan: Dual Account Separation

| Account | Purpose | Capital | Leverage Strategy |
|---------|---------|---------|-------------------|
| **Trading Account** | User large orders routed to HL (L4 proxy execution) | ≥ $500K | Follow user choice (5x–50x) |
| **Hedge Account** | L6 hedge engine exclusive | ≥ $200K | Fixed low leverage (2x–3x) |

**Benefits of separation**:

1. **Risk isolation**: Hedge positions not affected by large user orders, and vice versa
2. **Leverage independence**: Hedge account always low leverage (safe), trading account adjustable
3. **Independent margin ratio monitoring**: Pressure on one account doesn't drag down the other
4. **Clear accounting**: Hedge PnL and user routing PnL separate

**HL multi-account feasibility**: HL supports multiple independent sub-accounts via different wallet addresses. Each address is a separate HL trading account. Platform uses two different Turnkey custody addresses to enable this.

### Dual Account Capital Management

```
Platform Capital Pool
  ├── Trading Account ≥ $500K
  │     └── Margin ratio > 500% (safety line)
  ├── Hedge Account ≥ $200K
  │     └── Margin ratio > 500% (safety line)
  ├── Risk Reserve ≥ $500K
  └── Hot Wallet (withdrawal funds per chain)
```

Inter-account fund transfers:
- Trading account margin ratio < 300% → Supplement from capital pool
- Hedge account margin ratio < 300% → Supplement from capital pool
- All via Arbitrum on-chain operations, latency ~10–30 minutes

---

## III. Small INTERNAL Orders: Monitor, Don't Ignore

### System Behavior After Each INTERNAL Execution

```
User orders $5,000 BTC long (INTERNAL)
  │
  ▼
[L3] Internal accounting → Execution confirmed
  │
  ▼
[L6] Receives exposure change event
  │
  ▼
[L6] Recalculate BTC net exposure (incremental update, not full scan)
  │   Net exposure += $5,000 (user long direction)
  │
  ▼
[L6] Hedge threshold check
  │
  ├── Net exposure < $100K → Only record, no hedge (likely current state)
  ├── $100K ~ $500K → Flag "pending 50% hedge"
  ├── $500K ~ $1M   → Flag "pending 80% hedge"
  └── > $1M         → Stop internalization + hedge 80%
```

**Key points**:
- Each execution triggers exposure recalculation, but not necessarily hedging
- Hedging has a **debounce mechanism**: After exposure changes, wait 5 seconds before deciding to hedge. Avoids many small changes triggering many hedge operations.
- Hedging is **batch executed**: Not every $5K goes to HL as a separate order; instead, accumulate changes and batch execute.

---

## IV. Large Exposure Scenario Deep Dive

### Scenario: 200 Users Each $5,000 Long BTC → $1M Net Exposure

```
Timeline:
T1: First 20 users ($100K) → No hedge (< $100K threshold)
T2: Users 21-100 ($500K) → Hedge 50% = HL long BTC $250K
T3: Users 101-200 ($1M) → Hedge 80% = Target HL position $800K
    → Incremental hedge: $800K - $250K = Add $550K
T4: Exceeds $1M → Stop new BTC INTERNAL opens
```

### Hedge Capital Requirement Calculation

```
Hedge $800K notional BTC long:
  2x leverage: Margin = $400K
  3x leverage: Margin = $266K
  5x leverage: Margin = $160K ← Higher risk but better capital efficiency

Hedge account $200K capital:
  2x can hedge = $400K notional
  3x can hedge = $600K notional
  5x can hedge = $1M notional
```

### Leverage Ladder Strategy (Recommended)

| Hedge Notional | Leverage Used | Rationale |
|---------------|---------------|-----------|
| ≤ $300K | 2x | Safety first, BTC needs 50% drop to liquidate |
| $300K ~ $600K | 3x | Moderate risk, BTC needs 33% drop to liquidate |
| $600K ~ $1M | 5x | Capital efficiency, but requires closer monitoring |
| > $1M | 5x + stop internalization | No new exposure, wait for reduction |

**Core principle: Better to hedge partially than to have hedge position liquidated. Liquidating hedge position = risk instantly reverts to naked + realizes loss.**

### When Hedge Account Capital Is Insufficient

```
Hedge requirement > Hedge account capacity:
  │
  ├── Step 1: Increase hedge leverage (3x → 5x)
  │
  ├── Step 2: Emergency supplement from capital pool
  │           (on-chain operation, 10–30 min delay)
  │
  ├── Step 3: Switch to HL_MODE (all orders to HL, no new INTERNAL)
  │
  └── Step 4: Resume normal after capital top-up completes
```

---

## V. Pre-hedge Model (Standing Hedge / Delta Pool)

### Concept

Platform pre-maintains a **base hedge position** (Delta Pool) on HL hedge account, rather than waiting for net exposure to accumulate to threshold before hedging begins.

```
Platform startup:
  Open BTC long $100K (3x, requires $33K margin)
  Open BTC short $100K (3x, requires $33K margin)
  → Initial net position = 0 (Delta Neutral)
  → But already reserved ±$100K hedge capacity
```

### How It Works

```
Initial: HL long $100K + HL short $100K = net position $0

User goes long BTC $50K (INTERNAL):
  → Platform internal exposure = short $50K
  → Needs hedge = go long
  → Action: HL reduce short by $50K (not increase long)
  → HL becomes: long $100K + short $50K = net long $50K
  → Hedge complete, no new position needed

User goes short BTC $30K (INTERNAL):
  → Platform internal exposure changes = short $50K - $30K = short $20K
  → Action: HL increase short by $30K
  → HL becomes: long $100K + short $80K = net long $20K
```

### Advantages

1. **Faster response**: Reducing position is faster and has lower slippage than opening
2. **Bidirectional flexibility**: Whether users go long or short, can hedge by adjusting existing positions
3. **Avoid frequent opens/closes**: Reduces HL fees

### Disadvantages

1. **Capital cost**: Holding bidirectional positions requires double margin ($33K × 2 = $66K)
2. **Funding rate cost**: Holding long and short simultaneously incurs continuous funding rate costs
3. **HL single account same asset single direction limit**: HL doesn't support same account, same asset, both long and short!

### HL Technical Constraint

**HL single account, single asset, only one position (long or short), cannot be bidirectional simultaneously.**

This means pure Delta Pool (simultaneously long and short) is not viable in a single account.

### Viable Alternative: Single-Direction Pre-hedge + Dynamic Adjustment

```
Method 1: Single-direction base position

  If historical data shows users prefer going long (retail common):
    → Pre-open BTC long $100K (pre-hedge for user longs)
    → Users go long: Already covered by hedge, no action needed
    → Users go short: Reduce HL long

  Advantage: High capital efficiency (only one-direction margin)
  Disadvantage: Direction reversal if wrong direction assumed, high cost

Method 2: Cross-account bidirectional (if using dual accounts)

  Hedge Account A: BTC long $100K
  Hedge Account B: BTC short $100K
  → Achieve true Delta Neutral
  → But requires 3 HL accounts (1 trading + 2 hedge), complex management
```

### MVP Recommendation: Skip Pre-hedging

**Pre-hedging increases capital occupation and operational complexity, but MVP exposure is limited (few users initially). Passive hedging (action only when exposure exceeds threshold) is sufficient.**

Reserve pre-hedging for Phase 4 optimization, when enough user behavior data exists to decide pre-hedge direction.

---

## VI. Uncovered Edge Cases

### Case A: Hedge Position Gets HL Liquidated (Catastrophic)

```
Trigger:
  Hedge account uses 5x leverage for BTC long hedge
  BTC drops 20% → Hedge position approaches liquidation price
  HL liquidates hedge position → Hedge disappears
  Meanwhile user INTERNAL long also losing → Platform double loss

Protective measures:
  1. Hedge account margin ratio < 300% → Auto-supplement
  2. Hedge account margin ratio < 200% → P0 alert + emergency de-leverage
  3. Hedge position unrealized loss > 50% of margin → Auto-reduce (stop loss)
  4. Hedge position never uses > 5x leverage (hard cap)
```

### Case B: Extreme HL Volatility Causes Massive Hedge Slippage

```
Trigger:
  Need to hedge $500K BTC long
  BTC drops 10%, HL order book thin
  Market hedge order slippage 3–5%

Protective measures:
  1. Large hedges use limit orders + timeout to market
  2. Hedge orders set slippage ceiling (default 1%), exceed triggers split
  3. Pause hedging in extreme conditions + switch HL_MODE (stop new exposure)
```

### Case C: Hedge Direction Conflicts with User Route Direction

```
Trigger:
  BTC net exposure = user long $800K → Hedge: HL long $640K
  Large user orders BTC long $50K, routed to HL (> $10K)
  HL trading account also opens BTC long

  → Both accounts long BTC
  → If BTC drops, both lose simultaneously

This is not a problem, because:
  - Trading account long = user required (user bears PnL)
  - Hedge account long = covers INTERNAL counterparty exposure (platform risk control)
  - Separate accounting, no "conflict"
```

### Case D: Multiple Assets Trigger Hedge Simultaneously

```
Trigger:
  BTC net exposure $600K → need hedge $480K
  ETH net exposure $400K → need hedge $200K
  SOL net exposure $200K → need hedge $100K
  Total hedge need $780K

  Hedge account $200K, 3x leverage max capacity = $600K

Response strategy:
  1. Rank by net exposure size, prioritize largest
  2. Uncoverable assets → switch to HL_MODE for that asset
  3. Trigger capital pool supplement to hedge account
  4. Alert Risk Manager
```

### Case E: Concentrated User Close Causes Excessive Hedge

```
Trigger:
  User INTERNAL long $800K BTC → Hedged with HL long $640K (80% + 10% buffer)
  Large user closes all in 1 hour
  User INTERNAL long drops $800K → $0
  HL hedge position $640K now oversized

Response strategy:
  1. Each INTERNAL reduction → L6 recalculates net exposure
  2. Hedge position > net exposure × hedge ratio → Trigger reduction
  3. Batch reduce, control slippage
  4. Set max de-hedge rate (e.g., max $200K/min de-hedge)
```

### Case F: Funding Rate Erodes Hedge Profit

```
Trigger:
  Hedge account long BTC $500K for extended period
  Funding rate continuously positive (longs pay shorts)
  Every 8 hours pays: $500K × 0.01% = $50
  Daily = $150, Monthly = $4,500

Response strategy:
  1. Accumulate hedge costs (fees + funding + slippage) in real time
  2. If hedge cost > expected client loss revenue → Reduce internalization (lower thresholds)
  3. Hedge position funding fees included in risk control cost report
  4. Backend dashboard displays "hedge ROI" metric
```

---

## VII. Complete Hedge System Architecture Diagram

```
INTERNAL Position Change Events (open/close/liquidate)
  │
  ▼
[L6 Exposure Aggregator] ← Incremental updates, triggered per execution
  │
  ├── Each asset's net exposure (real-time)
  ├── Each asset's hedge position (read from hedge account)
  └── Hedge gap = Target hedge - Current hedge
  │
  ▼
[Debounce + Batch Merge] (5 second window)
  │
  ▼
[Hedge Decision Engine]
  │
  ├── Gap > 0 → Need to add hedge
  │     ├── Check hedge account available margin
  │     ├── Select leverage (ladder strategy)
  │     └── Generate hedge instruction → [L4 hedge account proxy] → HL
  │
  ├── Gap < 0 → Need to reduce hedge
  │     └── Generate de-hedge instruction → [L4 hedge account proxy] → HL
  │
  └── Gap ≈ 0 → No action
  │
  ▼
[Hedge Position Manager]
  ├── Track each asset's hedge position
  ├── Monitor hedge position margin ratio
  ├── Monitor hedge position unrealized PnL
  └── Accumulate hedge costs (fees + funding + slippage)
```

---

## VIII. Final HL Account Architecture Plan

### MVP Recommended: Dual Account

```
┌─────────────────────────────────────────┐
│         Platform Capital Pool             │
│  (Hot Wallet / Cold Wallet / Reserve)     │
└────────┬───────────────┬────────────────┘
         │               │
    ┌────▼─────┐   ┌────▼──────┐
    │ Trading  │   │ Hedging    │
    │ Account  │   │ Account    │
    │ ≥$500K   │   │ ≥$200K     │
    ├──────────┤   ├───────────┤
    │ L4 proxy │   │ L6 hedge   │
    │ user     │   │ exposure   │
    │ large    │   │ hedge low  │
    │ order    │   │ leverage   │
    │ routing  │   │ 2x–5x      │
    │ follow   │   │ ladder     │
    │ user     │   │ leverage   │
    │ leverage │   │            │
    └──────────┘   └───────────┘
```

### Configuration Requirements

| Parameter | Trading Account | Hedge Account |
|-----------|-----------------|---------------|
| HL Address | Separate Turnkey custody address A | Separate Turnkey custody address B |
| Margin Mode | Cross (full account) | Cross (full account) |
| Min Margin Ratio | > 500% | > 500% |
| Alert Margin Ratio | < 300% | < 300% |
| Emergency Margin Ratio | < 150% | < 150% |
| Max Leverage | Follow user (≤ HL max) | 5x (hard cap) |
| Agent Key | Dedicated Agent Key A | Dedicated Agent Key B |

---

## IX. Core Design Principles

1. **Hedge position must never be liquidated**: Better partial hedge than liquidated hedge position
2. **Hedging is a cost, not profit center**: Hedge purpose is to protect client loss revenue; hedging itself incurs fees, funding costs, slippage
3. **Progressive hedging**: Don't hedge all at once; execute in batches across time
4. **Hedge ≠ 1:1 coverage**: Only hedge exposure above threshold, retain some naked exposure (statistically users are net losers)
5. **Capital efficiency first**: Use minimum capital for sufficient risk coverage, not pursuit of perfect neutrality
