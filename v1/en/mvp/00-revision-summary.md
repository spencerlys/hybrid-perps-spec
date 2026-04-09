# Revision Summary

## Revision Objectives

To achieve the fastest MVP launch, this revision systematically restructures the complete PRD documentation. Core objectives include clarifying **two-domain separation** (Ledger & Trading Domain vs. Risk Control Domain), reinforcing the design purity of **internalization over matching**, optimizing **dual account isolation** strategy, strengthening **risk hedging** as a critical Phase 2 deliverable, and introducing execution-related operational documents (development plan, launch guide, acceptance checklist).

---

## Core Architecture Changes

| Dimension | Before | After | Scope |
|-----------|--------|-------|-------|
| **Domain Separation** | Undefined | Ledger & Trading Domain + Risk Control Domain | All modules |
| **Hedge Engine Phase** | Phase 3 | Phase 2 (Key Deliverable) | L6, Phase plan, new 07a |
| **Terminology** | Mixed "matching" and "internalization" | Remove "matching," unified to "internalization" | 01/02/04/06/07a, etc. |
| **HL Account Model** | Single account (trading + hedging mixed) | Dual accounts: Trading + Hedge | L4, L5, 07a, Infrastructure |
| **New Documents** | None | 07a (Hedge Architecture), 13 (Domain Architecture), 14 (Cost-Revenue), 15 (Dev Plan), 16 (Launch Guide), 17 (Acceptance) | Total docs: 12→17 |

---

## Per-Module Revision Checklist

| File | Type | Key Changes |
|------|------|-------------|
| **01-overview.md** | Enhanced | Phase table updated (L6 moved to Phase 2); clarified matching engine exclusion; MVP scope defined |
| **02-routing.md** | Enhanced | MVP routing constraints (canary: HL_MODE/NORMAL_MODE only); three-mode routing refinement; domain attribution |
| **03-account-assets.md** | Enhanced | Domain attribution (Ledger & Trading Domain); account structure and HL mapping |
| **04-internal-execution.md** | Enhanced | Domain attribution (Ledger & Trading Domain); terminology correction (matching → internalization); cross-domain interface contracts and failure handling |
| **05-hl-execution.md** | Enhanced | Large order splitting moved to Phase 4 (not in MVP); dual account adaptation (Trading vs Hedge); order fill optimization |
| **06-margin-liquidation.md** | Enhanced | Liquidation orchestration triggered by Risk Control Domain, executed by Ledger Domain; cross-domain interface contracts; margin model refinement |
| **07-risk-management.md** | Rewritten | Phase changed to 2 (no longer 3); domain attribution to Risk Control Domain; clear L3-L6 responsibility boundaries; natural hedging concept; staged hedging strategy |
| **07a-hedge-architecture.md** | New | Dual account isolation design; leverage ladder strategy; pre-hedge assessment mechanism; handling of 6 edge cases |
| **08-market-data.md** | No change | Unchanged |
| **09-settlement.md** | Enhanced | Domain attribution (Ledger & Trading Domain); reconciliation engine and Risk Control Domain interface |
| **10-withdrawal-liquidity.md** | No change | Unchanged |
| **11-admin.md** | Enhanced | Three routing mode buttons (HL_MODE/NORMAL_MODE/BETTING_MODE); hedge account monitoring dashboard; permissions and risk control relationship |
| **12-infrastructure.md** | Enhanced | Dual account-related table structures; inter-domain message queue design; event sourcing architecture |
| **13-domain-architecture.md** | New | Two-domain separation definition (concepts, responsibilities, boundaries); six major interface contracts; consistency strategy; invocation rules |
| **14-cost-revenue.md** | New | Cost minimization strategies (HL fee optimization, hedge strategy selection); revenue maximization mechanisms (spread, risk reserve allocation) |
| **15-dev-plan.md** | New | Multi-team parallel execution (hour-level granularity); Phase 1/2/3 critical path; pre-launch completion checks |
| **16-launch-rollback.md** | New | Canary rollout plan (percentage and limit increment); rollback trigger conditions and procedures; emergency handbook |
| **17-acceptance.md** | New | Acceptance checklist (functional, performance, risk dimensions); phase-layered acceptance standards; launch sign-off |
| **scenarios/05-risk-hedge.md** | New | Risk control and hedging scenarios (8: net exposure changes, threshold crossing, multi-asset, leverage changes, etc.) |
| **scenarios/06-edge-cases.md** | New | Edge and error scenarios (12: liquidation failure, hedge delay, price jump, network partition, etc.) |

---

## MVP / Post-MVP Boundary Overview

| Category | MVP Must Deliver | Phase 3 Delivers | Phase 4 / Post-MVP |
|----------|-----------------|-----------------|-------------------|
| **Account System** | ✓ User accounts, assets, multi-chain inflow | ✓ Isolated/cross margin models | Tiering and credit limits |
| **Order Processing** | ✓ Routing decision | ✓ Internalization execution | Large order splitting |
| **Execution Channel** | ✓ HL API relay | ✓ Internal internalization matching | Cross-threshold routing optimization |
| **Market Data** | ✓ HL WebSocket relay | ✓ Local mark price calculation | Market data derivatives |
| **Risk Control** | ✓ Net exposure monitoring, basic hedging | ✓ Unified liquidation, deviation settlement | Pre-hedging, strategy optimization |
| **Liquidation System** | ✓ HL order trigger | ✓ Cross-domain coordination, margin settlement | Intelligent liquidation |
| **Risk Reserve** | ✓ Basic accumulation and monitoring | ✓ Risk assessment and allocation | Dynamic adjustment |
| **Admin System** | ✓ Basic account queries, permissions | ✓ Routing/hedging/risk visualization | Advanced strategy configuration |

---

## Document Usage Guide

### Priority Reading Order

1. **This file (00)** — Revision overview, quick grasp of global changes
2. **01-architecture-baseline.md** (New) — Unified architecture baseline, understand two domains and interfaces
3. **01-overview.md** — Platform design and Phase overview
4. **02-routing.md** → **03-account-assets.md** → **04-internal-execution.md** — Order routing
5. **05-hl-execution.md** → **06-margin-liquidation.md** — Large orders and liquidation
6. **07-risk-management.md** → **07a-hedge-architecture.md** — Hedging and risk control (Phase 2 core)
7. **08-market-data.md** → **09-settlement.md** → **10-withdrawal-liquidity.md** — Support systems
8. **11-admin.md** → **12-infrastructure.md** — Admin and technical foundation
9. **13-domain-architecture.md** — Deep domain design (essential for developers)
10. **14-cost-revenue.md** — Business objectives
11. **15-dev-plan.md** → **16-launch-rollback.md** → **17-acceptance.md** — Execution roadmap

### Role-Based Usage Map

| Role | Must Read | Reference |
|------|-----------|-----------|
| **Product Manager** | 00, 01, 01(arch), 02, 04, 07, 14 | 03, 05, 06, 11 |
| **Backend Engineer** | 01(arch), 13, 04, 05, 06, 07a, 12, 15 | 02, 03, 08, 09, 11 |
| **Risk Engineer** | 07, 07a, 06, 11, 13 | 01, 02, 12 |
| **QA/Testing** | 17, 16, scenarios/*, 15 | 01, 04, 05, 06 |
| **DevOps/Operations** | 12, 16, 15 | 01, 11 |

---

## Key Design Decisions Review

### Why "Approach 2" and Not "Approach 3"?

- **Approach 2** (Internalization + Threshold Hedging): Platform is sole counterparty for all small orders, large orders forwarded to HL
  - MVP completeness: No matching engine needed, minimal code, fastest launch
  - Single risk axis: Platform directly assumes counterparty risk, controlled via hedging and reserve
  - Clear revenue: Spread + Risk Reserve = Income

- **Approach 3** (User-to-User Matching): User orders matched with each other, platform makes markets
  - Full matching engine required: Long development cycle, high complexity
  - Distributed risk: Lower platform risk, but high liquidity risk
  - **Explicitly NOT included in this version**

### Why Move Hedging from Phase 3 to Phase 2?

Net exposure is the moment of maximum risk. If we defer hedging to Phase 3, post-Phase 2 launch could result in catastrophic losses. The hedge engine must launch in sync with the routing engine.

### Why Two Domains vs. Single Domain?

Single domain creates circular dependencies between ledger updates and risk checks, and is hard to test. Two domains communicate asynchronously via event bus, achieving clean decoupling and independent upgradability per domain.

---

## Version Information

| Item | Value |
|------|-------|
| Revision Date | 2026-04-09 |
| Revision Version | v1.1-MVPFocus |
| Scope | Platform Perpetual Futures Trading Engine MVP |
| Review Status | Pending joint product/technology review |
| Next Steps | Initiate 15-dev-plan parallel workflows |
