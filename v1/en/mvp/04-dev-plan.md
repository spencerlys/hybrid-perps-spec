---
title: Multi-Team Development Plan (Hour-Based)
slug: dev-plan
category: MVP Planning
phase: Planning
updated: 2026-04-09
---

# Multi-Team Development Plan (Hour-Based)

This document outlines the multi-team parallel development hours and milestones for the perpetual futures platform (Approach 2). Based on 3.5-week delivery cycle and 6-team parallel architecture.

---

## Team Breakdown (6 Teams)

| Team | Name | Recommended Headcount | Responsible Modules |
|------|------|-----------|-----------|
| **T1** | Account Basics | 2 backend + 1 frontend | L1 Account & Assets, Deposit/Withdrawal, Multi-chain Aggregation, Hot/Cold Wallets |
| **T2** | Trading Core | 3 backend | L2 Order Routing, L3 Internalization Execution, Order Lifecycle, Fill Accounting |
| **T3** | HL Integration | 2 backend | R0 API Relay, L4 HL Proxy Execution, L7 Market Data Passthrough, WebSocket |
| **T4** | Risk Engine | 2 backend | L5 Margin & Liquidation, L6 Exposure Monitoring + Hedge Engine, Risk Reserve, Circuit Breaker |
| **T5** | Settlement & Reconciliation | 1 backend + 1 data analyst | L8 Settlement & Reconciliation, PnL Calculation, Funding Rate Settlement, Multi-party Reconciliation |
| **T6** | Admin & Operations | 1 backend + 1 frontend | L9 Admin, Dashboard, RBAC, Audit Logs |

**Total Headcount**: ~14 people (10 backend, 2 frontend, 1 data analyst, 1 DevOps)

---

## Phase 1 Detailed Hours (Foundation & Onboarding)

**Target Cycle**: 2.5 weeks (80 hours/person)
**Target Deliverable**: Users can deposit/withdraw, HL data passthrough normal, admin usable

| Task | Owner | Dev(h) | Integration(h) | Test(h) | Total(h) | Dependencies |
|------|-------|--------|----------------|---------|----------|--------------|
| User registration & login (Web2+Web3) | T1 | 24 | 8 | - | 32 | Foundation |
| Wallet balance model + ledger | T1 | 16 | 4 | 8 | 28 | T1 signup |
| TRON deposit (chain scan + confirm + book) | T1 | 32 | 8 | 16 | 56 | T1 wallet |
| ETH deposit (chain scan + confirm + book) | T1 | 24 | 8 | 12 | 44 | T1 wallet |
| Withdrawal flow (tiered approval + disbursement) | T1 | 32 | 8 | 16 | 56 | T1 wallet |
| Multi-chain aggregation + hot/cold wallet mgmt | T1 | 24 | 4 | 8 | 36 | T1 withdrawal |
| R0 REST relay proxy | T3 | 32 | 8 | 12 | 52 | Foundation |
| R0 WebSocket relay + multiplexing | T3 | 40 | 8 | 16 | 64 | T3 REST |
| HL connection management (heartbeat/reconnect/backoff) | T3 | 16 | 4 | 8 | 28 | T3 WS |
| L7 Market data caching + sync | T3 | 24 | 4 | 8 | 36 | T3 WS |
| Basic admin framework + RBAC | T6 | 32 | 8 | 12 | 52 | Foundation |
| User query page | T6 | 16 | 4 | 8 | 28 | T6 framework |
| **Infrastructure** | | | | | | |
| DB design + deploy + CI/CD | T3 | 40 | - | - | 40 | Foundation |
| Redis + message queue deploy | T3 | 16 | - | - | 16 | Foundation |
| Monitoring (Prometheus + Grafana) | T3 | 24 | - | - | 24 | Foundation |

**Phase 1 Total Hours**: ~550 hours
**Parallel Efficiency**: 6 teams parallel, ~2.5 weeks to complete
**Average per Person**: ~40 hours

---

## Phase 2 Detailed Hours (Routing + Hedging)

**Target Cycle**: 3 weeks
**Target Deliverable**: Routing + HL proxy execution complete, hedge engine basic operation, 7-day canary zero failures

| Task | Owner | Dev(h) | Integration(h) | Test(h) | Total(h) | Dependencies |
|------|-------|--------|----------------|---------|----------|--------------|
| L2 routing decision engine (three modes) | T2 | 32 | 8 | 16 | 56 | T3 market data |
| Order manager (create / state machine / receipt) | T2 | 40 | 8 | 16 | 64 | T2 routing |
| Close & cancel routing | T2 | 16 | 4 | 8 | 28 | T2 order mgr |
| L4 HL proxy open (unified account) | T3 | 32 | 8 | 12 | 52 | T3 R0 |
| L4 HL receipt price correction | T3 | 24 | 8 | 12 | 44 | T3 HL proxy |
| L4 HL position mapping + consistency check | T3 | 24 | 8 | 12 | 44 | T3 correction |
| HL dual account adaptation (trading + hedge) | T3 | 16 | 4 | 8 | 28 | T3 mapping |
| Pre-trade risk control (pre-order check) | T4 | 24 | 4 | 8 | 36 | T1 wallet |
| Real-time net exposure calculation | T4 | 24 | 4 | 12 | 40 | T2 routing |
| L6 hedge engine (threshold + instruction generation) | T4 | 40 | 8 | 16 | 64 | T4 exposure |
| Hedge position management (track + reduce) | T4 | 24 | 8 | 12 | 44 | T4 hedge |
| Risk reserve management | T4 | 16 | 4 | 8 | 28 | T4 hedge |
| Daily net loss circuit breaker | T4 | 8 | 4 | 8 | 20 | T5 PnL |
| Deviation monitoring + logging | T5 | 16 | 4 | 8 | 28 | T3 HL proxy |
| Routing mode control page | T6 | 16 | 4 | 8 | 28 | T6 framework |
| HL account status monitor page | T6 | 16 | 4 | 8 | 28 | T3 positions |
| **Full-path integration** | All | - | 40 | - | 40 | All modules |
| **Canary stress test** | All | - | - | 24 | 24 | Full-path |

**Phase 2 Total Hours**: ~720 hours
**Parallel Efficiency**: 6 teams parallel, ~3 weeks to complete
**Average per Person**: ~50 hours

---

## Phase 3 Detailed Hours (Internalization Launch)

**Target Cycle**: 3.5 weeks
**Target Deliverable**: Internalization + liquidation + settlement end-to-end, canary verified, BTC/ETH internalization open

| Task | Owner | Dev(h) | Integration(h) | Test(h) | Total(h) | Dependencies |
|------|-------|--------|----------------|---------|----------|--------------|
| L3 internalization engine (market + limit) | T2 | 48 | 12 | 16 | 76 | T2 routing |
| Internal limit order queue | T2 | 24 | 8 | 12 | 44 | T3 market data |
| Platform counterparty position mgmt | T2 | 16 | 4 | 8 | 28 | T2 internal |
| L5 margin calculation engine (isolated + cross) | T4 | 48 | 12 | 24 | 84 | T2 internal |
| L5 unified liquidation engine | T4 | 48 | 16 | 24 | 88 | T4 margin |
| HL calculation precision replication (tiered maint rates) | T4 | 32 | 8 | 16 | 56 | T4 liquidation |
| L8 complete PnL settlement | T5 | 32 | 8 | 16 | 56 | T4 liquidation |
| Funding rate settlement (INTERNAL + HL) | T5 | 24 | 8 | 12 | 44 | T3 HL data |
| Multi-party reconciliation engine | T5 | 32 | 8 | 16 | 56 | T5 PnL |
| Deviation settlement backstop | T5 | 16 | 4 | 8 | 28 | T4 reserve |
| Risk dashboard (exposure / PnL / liquidation stats) | T6 | 32 | 8 | 12 | 52 | T4 exposure |
| Deviation panel + aggregation panel | T6 | 16 | 4 | 8 | 28 | T5 reconciliation |
| **Full-path integration** | All | - | 60 | - | 60 | All modules |
| **Canary internal test + gradual ramp** | All | - | - | 40 | 40 | Full-path |

**Phase 3 Total Hours**: ~730 hours
**Parallel Efficiency**: 6 teams parallel, ~3.5 weeks to complete
**Average per Person**: ~52 hours

---

## Critical Path Analysis

```
Phase 1 (W1-W2.5): Foundation
├─ T3 (Infra + R0 + WS)
├─ T1 (Account + Deposits/Withdrawals)
└─ T6 (Admin Framework + RBAC)
    └─ Phase 1 Pass + Admin Usable

                    ↓

Phase 2 (W2.5-W5.5): Routing + Hedging
├─ T2 (L2 Routing) → T3 (L4 HL Proxy)
├─ T4 (Risk + Hedge)
└─ T5 (Deviation Monitoring)
    └─ Phase 2 Pass + 7-Day Canary Verified

                    ↓

Phase 3 (W5.5-W9): Internalization Launch
├─ T2 (L3 Internalization) → T4 (L5 Liquidation)
├─ T5 (L8 Settlement + Reconciliation)
└─ T6 (Dashboard + Audit)
    └─ Phase 3 Pass + BTC/ETH Internalization Open
```

**Critical Dependencies**:
- Phase 1 ✓ → Phase 2 must complete (T3 infrastructure)
- T2 routing ✓ → T3 HL proxy start (depends on routing decision)
- T3 HL proxy ✓ → T4 hedge start (depends on position data)
- T4 liquidation ✓ → T5 settlement start (depends on liquidation trigger)

---

## Milestones & Acceptance

| Timeline | Milestone | Acceptance Criteria | Owner |
|----------|-----------|-------------------|-------|
| **T+2.5w** | Phase 1 Complete | Users can register/deposit/withdraw, HL data passthrough OK, admin usable | T1/T3/T6 |
| **T+5w** | Phase 2 Complete | Routing engine works, HL proxy execution correct, hedge basic operation | T2/T3/T4 |
| **T+6w** | Phase 2 Canary Pass | Full HL routing 7 days zero failures, risk calc accuracy >99.9% | QA/Risk |
| **T+9w** | Phase 3 Complete | Internalization + liquidation + settlement end-to-end | T2/T4/T5 |
| **T+10w** | Phase 3 Canary Pass | Internal test 1 week, BTC/ETH internalization open, reserve sufficient | QA/Risk |

---

## Development Implementation Highlights

### 1. Code Organization & Branch Strategy
- Separate repos per module: `repo-l1-account`, `repo-l2-routing`, etc.
- Main branches: `main` (production), `release/phase-X` (phase), `dev` (development)
- All new code via PR, requires 2 approvals
- Canary deployment: 5% → 20% → 100%

### 2. Interface Standardization & Integration
- All cross-team interfaces use OpenAPI 3.0, auto-generate SDKs
- Weekly Mon/Fri interface alignment meetings
- Use Mock Server for parallel development

### 3. Database Design & Sync
- T3 owns core schema design (unified types/indexes)
- Key tables: users, wallets, orders, positions, liquidations, hedge_orders
- Liquibase for migrations, all changes version-controlled
- Weekly DB sync check

### 4. Test Coverage & QA Schedule
- Unit tests: ≥80% coverage per team
- Integration: All cross-team paths per phase
- Load: Phase 2 1000 orders/sec, Phase 3 5000 liquidations/hour
- See Acceptance Checklist (06-acceptance.md)

### 5. Monitoring & Alerting
- Each team configures Prometheus metrics
- P0 alerts: HL disconnect, HL margin <150%, liquidation failure
- P0 response: 5min, P1 response: 30min
- On-call rotation: 1 week per team, 2-person squads

### 6. Documentation & Knowledge Sharing
- API docs + deploy guides per module
- Fri 30min tech talks (rotating)
- Internal Wiki: architecture, FAQs, troubleshooting
- Phase completion summary reports

---

## Tools & Resources

**Development**:
- Backend: Go 1.22+ / Node.js 20+
- Frontend: React 18 + TypeScript
- Database: PostgreSQL 15 + Redis 7
- Message Queue: RabbitMQ 3.12 / Kafka 3.6

**Collaboration**:
- Code: GitHub/GitLab
- Tasks: Jira / Linear
- Docs: Confluence / Notion
- Chat: Slack / Lark
- Monitoring: Prometheus + Grafana + ELK

**Budget**:
- Infrastructure: ~$20K (3.5-week cost)
- Tools: ~$5K (subscriptions)
- Docs/Training: ~$2K

---

## Risks & Mitigation

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| HL API changes | Medium | High | Coordinate with HL, 2-week buffer |
| Liquidation precision drift >0.1% | High | High | Phase 2 canary, replicate HL calc (1000+ tests) |
| Hedge position tracking failure | Medium | Medium | Real-time reconciliation, 10-sec consistency check |
| Team member departure | Low | Medium | Knowledge documentation, dual-code critical modules |
| Chain confirmation delay | Low | Low | Backup nodes, auto-rescan |

---

## Appendix: Hour Estimation Method

```
Total Hours = (Dev + Integration + Test) × Dependency Factor

Dev Hours: Planning Poker (XS=4h, S=8h, M=16h, L=32h, XL=64h)
Integration Hours: Dev × 20-30% (typically 1/3 to 1/5 of dev)
Test Hours: Dev × 25-50% (functional); +50% for performance (Phase 2-3)
Canary: 20-40h fixed (all teams shared)

Parallel Coefficient:
  - No dependencies: 1.0 (fully parallel)
  - Weak dependencies: 1.1 (few sync points)
  - Strong dependencies: 1.3-1.5 (wait for prereq)

Cycle = max(T1-T6 hours) / (headcount × 40h/week) + sync/buffer (+10%)
```
