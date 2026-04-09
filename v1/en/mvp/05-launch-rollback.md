---
title: Launch & Rollback Manual (MVP)
slug: launch-rollback
category: MVP Planning
phase: Operations
updated: 2026-04-09
---

# Launch & Rollback Manual (MVP)

This document standardizes the canary strategy, emergency response, control matrix, and rollback procedures for the perpetual futures platform (Approach 2). Ensures smooth Phase 2 and Phase 3 launches with fault recovery.

---

## Canary Strategy

### Phase 2 Canary (Routing + Hedging)

**Objective**: 7 days full HL routing with zero failures, risk calc precision verified

**Cycle**: 7 days (4 stages)

#### Stage 1: Full HL Routing (Days 1-3)
- **Routing**: All orders → HL regardless of size (NORMAL_MODE override)
- **Hedge Engine**: Observe net exposure only, no execution
- **Liquidation**: Observe only, no actual liquidation (test users only)
- **Risk**: Minimal (no internalization)
- **Acceptance**:
  - HL proxy zero deviation (actual vs receipt ≤1bp)
  - API latency <5ms P99
  - Order → receipt <50ms P99
  - Zero failures, no data loss

#### Stage 2: Risk Monitoring (Days 4-5)
- **Routing**: Continue full HL
- **Hedge Engine**: Calculate net exposure, log instructions (don't execute)
- **Reconciliation**: Compare observed vs calculated net exposure
- **Acceptance**:
  - Precision >99%
  - Hedge instruction logs complete
  - Zero failures

#### Stage 3: Simulated Hedging (Days 6-7)
- **Routing**: Continue full HL
- **Hedge Engine**: Generate instructions (don't execute), log for audit
- **Liquidation**: Compute but don't trigger actual closes
- **Acceptance**:
  - Hedge instruction generation matches threshold logic
  - Prices reasonable (not >5% from market)
  - No missing logs

#### Pass Criteria
- ✓ 7 days HL proxy zero deviation
- ✓ Routing latency <5ms P99, <10ms 99.5% of time
- ✓ API response <100ms P95
- ✓ Availability >99.9%
- ✓ No data loss
- ✓ Risk calc precision >99.5%

**Failure**: >1% error rate → rollback to Phase 1, fix, recanary

---

### Phase 3 Canary (Internalization Launch)

**Objective**: 1-week internal test, BTC/ETH internalization open, gradual ramp to production

**Cycle**: 10 days (4 stages)

#### Stage 1: Ultra-Conservative (Days 1-3)
- **Symbols**: BTC only (ETH/others standby)
- **Threshold**: $1,000 (99% → HL)
- **Max net exposure**: $50K (auto pause)
- **Max single order**: $1,000
- **Reserve trigger**: Observe only
- **Objective**: Verify internalization stability, liquidation correctness
- **Acceptance**:
  - Internalization <10ms fill
  - Limit order success >99.5%
  - Liquidation delay <1s
  - PnL error <0.01%

#### Stage 2: Low Risk (Days 4-5)
- **Symbols**: BTC, ETH
- **BTC threshold**: $5,000
- **ETH threshold**: $1,000
- **Max net exposure**: $100K
- **Reserve trigger**: <$400K restrict orders
- **Objective**: Multi-symbol coordination, hedge stability
- **Acceptance**:
  - No symbol conflicts
  - Hedge execution >99% on-time
  - Liquidation precision >99%

#### Stage 3: Target Config (Days 6-7)
- **Symbols**: BTC, ETH (target values)
- **Thresholds**: $10,000 each
- **Max net exposure**: $500K
- **Reserve trigger**: Observe only (<$200K pause, logs only)
- **Objective**: Near-production config, stress test
- **Acceptance**:
  - High concurrency <10ms latency (1000 req/s)
  - Reconciliation >99.9%
  - Reserve sufficient (>$500K)

#### Stage 4: Full Open (Days 8-10)
- **Symbols**: BTC, ETH (production)
- **Config**: Final design (NORMAL $10K, BETTING $50K)
- **Risk controls**: All enabled
- **User signup**: Open for small orders
- **Objective**: Production stability, user feedback
- **Acceptance**:
  - >$1M volume/day
  - >50 liquidations/day
  - No critical user issues

#### Pass Criteria
- ✓ Liquidation detection <1s
- ✓ PnL error <0.1% vs HL
- ✓ Reserve >$500K
- ✓ Availability >99.95%
- ✓ No user asset loss

**Failure**:
- Liquidation failure → close internalization, full HL
- Wrong liquidation → manual review, no auto liquidate
- Low reserve → pause new orders, pause withdrawals

---

## Control Matrix (Risk Manager Controlled)

All switches in config center (Consul/etcd), effect <10s globally (no restart).

| Switch | Type | Default | Scope | Permission | Rollback | Latency |
|--------|------|---------|-------|-----------|----------|---------|
| **global_betting_enabled** | bool | false | Global internalization toggle | Super Admin | Switch → HL_MODE | <1s |
| **routing_mode** | enum | HL_MODE | Routing mode (HL_MODE/NORMAL/BETTING) | Risk Manager | Back to HL_MODE | <1s |
| **normal_threshold** | money | $10,000 | NORMAL_MODE size limit | Risk Manager | Increase/set to $0 | <5s |
| **betting_threshold** | money | $50,000 | BETTING_MODE size limit | Risk Manager | Lower/disable | <5s |
| **auto_mode_switch** | bool | false | Auto mode switching | Risk Manager | Disable | <5s |
| **hedge_enabled** | bool | true | Hedge engine toggle | Risk Manager | Disable → manual | <1s |
| **hedge_max_leverage** | float | 3.0x | Max hedge leverage | Risk Manager | Lower to 1.5x | <5s |
| **per_symbol_betting** | list | BTC,ETH | Symbols allowed for internalization | Risk Manager | Clear (ban all) | <5s |
| **liquidation_enabled** | bool | true | Liquidation engine | Super Admin | Disable (manual) | <1s |
| **liquidation_check_interval_ms** | int | 500 | Liquidation detection frequency | Risk Manager | Increase to 2000 | <5s |
| **withdrawal_enabled** | bool | true | Withdrawal toggle (run-on defense) | Super Admin | Disable | <1s |
| **max_internal_exposure_usd** | money | $500,000 | Max internal net exposure/symbol | Risk Manager | Lower | <5s |
| **hedge_threshold_tier1** | money | $100,000 | Tier 1 hedge trigger (50%) | Risk Manager | Raise to $500K | <5s |
| **hedge_threshold_tier2** | money | $500,000 | Tier 2 hedge trigger (80%) | Risk Manager | Raise to $1M | <5s |
| **risk_reserve_min_usd** | money | $200,000 | Reserve minimum (pause if below) | Risk Manager | Raise to $300K | <5s |
| **daily_net_loss_limit** | money | $500,000 | Daily net loss circuit breaker | Risk Manager | Raise/disable | <5s |
| **max_order_size_internal** | money | $10,000 | Max single internal order | Risk Manager | Lower to $1K | <5s |
| **force_hl_volatility_pct** | float | 5.0% | Force HL if volatility >X | Risk Manager | Raise to 10% | <5s |
| **force_hl_latency_ms** | int | 500 | Force HL if HL latency >X | Risk Manager | Raise to 1000 | <5s |

**Permission Levels**:
- **Super Admin**: Global internalization, liquidation, withdrawal
- **Risk Manager**: Routing, hedge, reserve, circuit breaker
- **Operations**: Read-only, withdrawal approval

**Change Audit**:
- Immutable audit log for all switch changes
- Format: `timestamp | user | switch | old_value | new_value | reason`
- Rollback support: `rollback_switch(name, timestamp)`

---

## Alert System & Emergency Response

### P0 Alerts (5-min response, on-call must respond)

| Alert | Trigger | Immediate Action | Further |
|-------|---------|----------|---------|
| **HL disconnection** | WS/REST down >1min | Slack/PagerDuty, pause new orders | Reconnect; if >5min → manual |
| **HL account margin <150%** | margin_ratio < 150% | Pause all HL new opens | Top up to >200%, reduce leverage |
| **Hedge account margin <200%** | margin_ratio < 200% | Same, reduce to 2x leverage | Emergency top-up/closeout |
| **Liquidation engine failure** | Detection fails 5+ times or >10% execution failure | liquidation_enabled=false, manual liquidate | Debug, fix, restart |
| **Hot wallet <$100K** | Available <$100K | Pause large withdrawals (>$10K) | Transfer from cold wallet |
| **Single-order deviation >5%** | Price discrepancy >5% | Pause symbol HL routing → internal | Debug cause |
| **User asset mismatch >0.1%** | Discrepancy >0.1% | Pause trading, start manual reconciliation | Manual adjustment |

**SLA**: 5-min confirm, 10-min action. No response after 5min → escalate

---

## Rollback Decision Tree

```
Incident
    ↓
[Severity Assessment]
    ├─ Critical Path Failure → Immediate Rollback
    ├─ Data Inconsistency → 5-min Rollback
    └─ Non-Critical → Monitor/Fix
    ↓
[Rollback Level]
    ├─ Single Module → Disable that switch + Fix
    ├─ Cross-Module → HL_MODE + disable internalization
    └─ Full System → Degradation Level 3
    ↓
[Execute Rollback]
    └─ Modify config + observe 5min + confirm
    ↓
[Fix & Restart]
    └─ Fix + unit tests + 5% canary → full
```

---

## Degradation Levels

### Level 1: Partial Function (Controllable Risk)

**Trigger**: Internalization/Liquidation failure (HL OK)

**Available**:
- Query ✓
- Withdraw ✓
- New open (HL only) ✓
- Close (INTERNAL needs manual) ⚠
- Internalization ✗

**Switch**: `global_betting_enabled=false, hedge_enabled=false, routing_mode=NORMAL_MODE`

---

### Level 2: Read-Only + Limited (Asset Protection)

**Trigger**: Liquidation completely broken / Multi-module failures

**Available**:
- Query ✓
- Withdraw ✓
- Close (market) ✓
- New open ✗
- Internalization ✗

**Switch**: `global_betting_enabled=false, routing_mode=HL_MODE, liquidation_enabled=false`

---

### Level 3: Maintenance Mode (Emergency Stop)

**Trigger**: Full outage / Database unavailable / Asset at risk

**Available**: None (static maintenance page)

**Switch**: `global_api_enabled=false` → 503 Service Unavailable

**Permission**: Super Admin only

---

## Rollback Checklist

Before any rollback:

- [ ] Confirm problem from logs/monitoring (not rumor)
- [ ] Assess impact (users/assets affected)
- [ ] Choose rollback level
- [ ] Notify management: Slack to CEO/CTO + support
- [ ] Modify config switch (log user + time)
- [ ] Wait <10s for global push
- [ ] Observe 5 minutes (alerts, latency, errors)
- [ ] Confirm recovery
- [ ] Collect logs for post-incident analysis
- [ ] Communicate with users: announce fix + progress
- [ ] Document fix plan + ETA

---

## Implementation Highlights

### 1. Centralized Switch Management
- All switches in Consul/etcd (not hardcoded)
- HTTP API: `GET /config/switch/{name}`
- CLI tool: `set_switch routing_mode HL_MODE --reason "incident rollback"`
- Audit log (Elasticsearch): immutable, no deletion

### 2. Canary Infrastructure
- Traffic splitting gateway: 5% → 20% → 50% → 100%
- Auto-rollback if: P99 latency +20% or error rate >1%
- Zero-downtime: run old and new versions in parallel

### 3. Failure Detection & Auto-Recovery
- Liquidation health check every 30s
- HL reconnect with exponential backoff (1s → 2s → 4s → 8s, max 5min)
- Reconciliation check every 5min (>0.1% mismatch = alert)

### 4. Comprehensive Testing
- Unit tests for each switch
- Automated rollback validation scripts
- Stress tests: 5000 orders/s rollback verification

### 5. Monitoring & Notifications
- Prometheus: All critical metrics
- Grafana: Real-time system dashboard
- PagerDuty/Slack: Auto P0 notifications
- Monthly: Failure drill (simulate incident, validate rollback)

### 6. Documentation & Training
- Runbook: Each switch purpose, usage, modification method
- Troubleshooting guide: P0 incident diagnosis
- Monthly training: New staff on canary/rollback
- Incident database: Historical failures for prevention
