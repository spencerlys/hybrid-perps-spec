---
doc_id: scenarios-en-index
title: Business Scenario Records Index
version: 1.0
lang: en
updated: 2026-04-08
---

# Business Scenario Records

> Parent index: [../index.md](../index.md)
> Authoritative source: [../../zh/scenarios/00-index.md](../../zh/scenarios/00-index.md)

Scenario records capture complete processing flows for specific business cases, including happy paths and edge cases. Each scenario has a unique ID for direct reference.

## Scenario Files

| File | Category | Count |
|------|----------|-------|
| [01-order-routing.md](01-order-routing.md) | Order Routing | 6 |
| [02-liquidation.md](02-liquidation.md) | Liquidation | 6 |
| [03-funding-settlement.md](03-funding-settlement.md) | Funding Rate Settlement | 5 |
| [04-deposit-withdrawal.md](04-deposit-withdrawal.md) | Deposit & Withdrawal | 6 |

## Scenario Format

```
Scenario ID   : SC-XXX-001
Name          : Short description
Preconditions : Initial user/system state
Steps         : 1. 2. 3. ...
Expected      : System behavior and final state
Related PRD   : Linked PRD modules
```

## Quick Reference

### By Routing Type
- INTERNAL B-book scenarios → [01-order-routing.md](01-order-routing.md) SC-RT-001 ~ SC-RT-003
- HYPERLIQUID routing scenarios → [01-order-routing.md](01-order-routing.md) SC-RT-004 ~ SC-RT-006

### By Margin Mode
- Isolated liquidation → [02-liquidation.md](02-liquidation.md) SC-LQ-001 ~ SC-LQ-002
- Cross margin liquidation → [02-liquidation.md](02-liquidation.md) SC-LQ-003 ~ SC-LQ-004
- Mixed mode → [02-liquidation.md](02-liquidation.md) SC-LQ-005 ~ SC-LQ-006

### By Asset Operation
- Deposits → [04-deposit-withdrawal.md](04-deposit-withdrawal.md) SC-DW-001 ~ SC-DW-002
- Withdrawals → [04-deposit-withdrawal.md](04-deposit-withdrawal.md) SC-DW-003 ~ SC-DW-006
