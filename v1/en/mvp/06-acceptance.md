---
title: Acceptance Checklist
slug: acceptance
category: MVP Planning
phase: Testing
updated: 2026-04-09
---

# Acceptance Checklist

This document standardizes acceptance criteria and checklists for each MVP phase. Covers functional, risk control, performance, and operational dimensions with 100+ verifiable items.

---

## Phase 1 Acceptance Checklist (Foundation & Onboarding)

### 1.1 Authentication (T1)

- [ ] Email registration with verification code
- [ ] Email login with session management
- [ ] MetaMask signature login (no email)
- [ ] Google/Apple OAuth login
- [ ] Profile editing (nickname/avatar/KYC)
- [ ] Password reset (email verify + new password)
- [ ] Web2/Web3 account linking
- [ ] Session security: 2FA (TOTP) support
- [ ] Session security: token expiry <1 hour
- [ ] Session security: device trust list support

### 1.2 Wallet & Assets (T1)

- [ ] Independent wallet address per user
- [ ] Balance display accuracy (USDT/USDC/etc)
- [ ] Balance transaction history complete
- [ ] Transaction history filterable by time/type
- [ ] Total asset calculation correct (cash + positions)
- [ ] Frozen amount display accurate
- [ ] Multi-currency support (USDT/USDC/etc)

### 1.3 TRON Deposit (T1)

- [ ] Generate TRON deposit address (TRC20)
- [ ] Address valid long-term
- [ ] Network selection display (TRON mainnet)
- [ ] Amount validation ($10-$100K)
- [ ] On-chain TRON transfer (scan <30s interval)
- [ ] Auto-book after 19 confirmations
- [ ] Complete deposit history (TxHash/time/amount/status)
- [ ] Clear error messages
- [ ] Support multiple concurrent deposits

### 1.4 ETH Deposit (T1)

- [ ] Generate ETH deposit address (ERC20)
- [ ] Currency selection (USDT/USDC)
- [ ] On-chain scan normal (12 confirmations)
- [ ] Auto-book after 12 confirmations
- [ ] ETH and TRON deposits independent

### 1.5 Withdrawal (T1)

- [ ] User initiate withdrawal (amount/address/chain)
- [ ] <$10K: auto-approve, 5-min on-chain
- [ ] $10K-$100K: 1 ops approval, <1 hour
- [ ] >$100K: risk + ops 2-person approval, <2 hours
- [ ] Address management (add/delete/label)
- [ ] Address whitelist support (optional)
- [ ] Transparent fee display
- [ ] Complete withdrawal history
- [ ] Rejection reason display
- [ ] Duplicate prevention (same address, 10-min interval)

### 1.6 Multi-Chain Aggregation & Hot/Cold Wallets (T1)

- [ ] Hot wallet monitoring (>$1M → auto-transfer cold)
- [ ] Cold wallet auto-replenish (<$100K)
- [ ] Cross-chain transfer process clear
- [ ] Multi-chain balance summary
- [ ] Wallet security (no private key exposure)

### 1.7 HL API Relay (T3)

- [ ] Full REST proxy (GET/POST /v1/*)
- [ ] Auto request/response format conversion
- [ ] HL error transparency
- [ ] Timeout handling (>30s → error)
- [ ] Request dedup (no double-send)
- [ ] Rate limiting management

### 1.8 HL WebSocket (T3)

- [ ] WS connection stable
- [ ] Multi-channel subscribe (prices, orderbook, trades)
- [ ] Real-time price push (<100ms)
- [ ] Real-time orderbook push
- [ ] Real-time K-line push
- [ ] Heartbeat + auto-reconnect
- [ ] Per-user WS isolation

### 1.9 Market Data Display (T3+T6)

- [ ] Price display (BTC/ETH/major coins)
- [ ] Real-time refresh (<500ms)
- [ ] K-line charts (1m/5m/15m/1h/4h/1d)
- [ ] Volume display
- [ ] Funding rate display
- [ ] 24h change display
- [ ] Coin search support

### 1.10 Admin Framework & RBAC (T6)

- [ ] Admin login (Super Admin/Risk Manager/Operations/Data Analyst)
- [ ] Role permission isolation
- [ ] Four role types with complete function division
  - [ ] Super Admin: all + system parameters
  - [ ] Risk Manager: risk config + routing + hedge thresholds
  - [ ] Operations: withdrawal approval + user queries (read-only)
  - [ ] Data Analyst: reports + audit logs (read-only)
- [ ] Operation audit logging

### 1.11 User Query Page (T6)

- [ ] Search users by ID/email
- [ ] Show asset details (deposit/withdrawal/trade history)
- [ ] Show wallet addresses + binding time
- [ ] Export user list (Excel)
- [ ] Time range filtering

### 1.12 Infrastructure (T3)

- [ ] PostgreSQL deployed, backup strategy sound
  - [ ] Daily backups (7-day retention)
  - [ ] Master-slave replication normal
  - [ ] Disaster recovery tested
- [ ] Redis cluster deployed, cache warming
- [ ] Message queue (RabbitMQ/Kafka)
  - [ ] Message persistence
  - [ ] Dead letter queue configured
- [ ] Docker containerization
- [ ] CI/CD complete (git push → test → deploy)
  - [ ] Pre-commit hooks (format check)
  - [ ] Unit tests >80% pass required
  - [ ] Auto-deploy to test env
- [ ] Prometheus monitoring
  - [ ] Key metrics (latency/error/QPS)
  - [ ] Alert rules (>5s latency, >1% errors)
- [ ] Grafana dashboard
  - [ ] Real-time system status
  - [ ] Historical trends

### 1.13 Phase 1 Pass Criteria

- [ ] All features pass ✓
- [ ] Availability ≥99.5% (during testing)
- [ ] No P0 failures
- [ ] No critical user issues
- [ ] Asset reconciliation pass (error <0.01%)

---

## Phase 2 Acceptance Checklist (Routing + Hedging)

### 2.1 Order Routing (T2)

- [ ] Mode switching (HL_MODE ↔ NORMAL_MODE ↔ BETTING_MODE)
- [ ] NORMAL_MODE: ≤$10K → INTERNAL
- [ ] NORMAL_MODE: >$10K → HL
- [ ] BETTING_MODE: ≤$50K → INTERNAL
- [ ] BETTING_MODE: >$50K → HL
- [ ] Force HL conditions:
  - [ ] Volatility >5%/hour → HL_MODE auto
  - [ ] HL latency >500ms → HL_MODE auto
  - [ ] Auto-recovery to previous mode
- [ ] Routing latency <5ms P99
- [ ] Complete routing logs

### 2.2 Order Lifecycle (T2)

- [ ] Create: status PENDING
- [ ] Submit: status CREATED, assign ID
- [ ] Receipt: fill price/quantity received
- [ ] Partial fill: cumulative tracking
- [ ] Full fill: status FILLED
- [ ] Cancel: status CANCELLED
- [ ] Error: status ERROR, error message
- [ ] Retry mechanism (3 attempts auto)
- [ ] Query by ID/symbol/time range

### 2.3 HL Proxy Execution (T3)

- [ ] Forward to HL immediately
- [ ] HL order ID mapping (1:1)
- [ ] Receipt handling (price/quantity/time)
- [ ] Price correction (<1bp deviation)
- [ ] Position sync every 5s
- [ ] Position consistency (error <0.01%)
- [ ] Forwarding latency <50ms P99
- [ ] Connection stability (recovery <5min)

### 2.4 Pre-Trade Risk Control (T4)

- [ ] Balance check (insufficient → reject)
- [ ] Symbol availability check
- [ ] Leverage limit check
- [ ] Invalid order check (0/negative)
- [ ] Blacklist check
- [ ] Check latency <10ms

### 2.5 Real-Time Net Exposure (T4)

- [ ] Formula: Net = Σ(Long_INTERNAL) - Σ(Short_INTERNAL)
- [ ] Per-symbol separation
- [ ] Real-time update on new order/close
- [ ] Calculation precision ≥99.5%
- [ ] Calculation latency <1s
- [ ] Backend queryable

### 2.6 Hedge Engine (T4)

- [ ] Threshold tiers correct:
  - [ ] <$100K: no hedge
  - [ ] $100K-$500K: 50% hedge
  - [ ] $500K-$1M: 80% hedge
  - [ ] >$1M: 80% hedge + pause new internalization
- [ ] Instruction generation
- [ ] Instruction forwarding to HL
- [ ] Position tracking match
- [ ] Auto reduce-position as exposure decreases
- [ ] Stop-loss at 5% loss
- [ ] Latency <10ms (generation)

### 2.7 Risk Reserve (T4)

- [ ] Initialize: $500K (manual transfer)
- [ ] Inflow: 20% of user loss
- [ ] Outflow: asset abnormality backstop
- [ ] Real-time balance display (admin)
- [ ] Alerts: <$500K (yellow), <$200K (red + pause internalization)
- [ ] Audit logs for all inflows/outflows

### 2.8 Daily Net Loss Circuit Breaker (T4)

- [ ] Definition: cumulative daily INTERNAL user loss
- [ ] Threshold: $500K
- [ ] Trigger: pause all new internalization orders
- [ ] Clear message to users
- [ ] Auto-reset next day

### 2.9 Deviation Monitoring (T5)

- [ ] Definition: platform calc vs HL actual
- [ ] Check interval: every 10s
- [ ] Alert thresholds: >0.1% (yellow), >1% (red + pause)
- [ ] Event logging (time/reason/adjustment)
- [ ] Backstop: use reserve to adjust if unexplained

### 2.10 Admin Control Pages (T6)

- [ ] Routing mode switch page (current + history)
- [ ] Threshold adjustment (normal/betting)
- [ ] HL account status (margin ratio/positions/balance)
- [ ] Refresh rate <5s

### 2.11 Phase 2 Canary Pass Criteria

- [ ] All features pass ✓
- [ ] 7 days zero failures
- [ ] Routing accuracy ≥99.9%
- [ ] HL proxy precision ≥99.9%
- [ ] API availability ≥99.95%
- [ ] Risk calc precision ≥99.5%
- [ ] No user asset loss

---

## Phase 3 Acceptance Checklist (Internalization Launch)

### 3.1 Internalization Execution (T2)

- [ ] Market order: fill <10ms
- [ ] Fill price accuracy (±0.5bp)
- [ ] Fill quantity matches request
- [ ] Limit order queue management
- [ ] Auto-trigger when HL price reaches limit
- [ ] Auto-cancel after 7 days no-fill
- [ ] Platform counterparty auto-creation
- [ ] Position query support

### 3.2 Internal Limit Order Queue (T2)

- [ ] FIFO or priority queue management
- [ ] Price trigger detection every second
- [ ] Trigger latency <1s
- [ ] Partial fill handling
- [ ] Queue visualization (admin)

### 3.3 Margin Calculation (T4)

- [ ] Isolated margin:
  - [ ] Formula: margin_ratio = position_value / margin
  - [ ] Maintenance rates match HL
  - [ ] Calculation precision ≥95% vs HL
- [ ] Cross margin:
  - [ ] Formula: margin_ratio = account_equity / total_maint
  - [ ] Calculation precision ≥95% vs HL
- [ ] Real-time unrealized PnL
- [ ] Per-second updates

### 3.4 Unified Liquidation (T4)

- [ ] Trigger detection:
  - [ ] Isolated: margin_ratio ≤ maint_rate
  - [ ] Cross: account_margin_ratio ≤ account_maint_rate
  - [ ] Detection latency <1s
- [ ] Isolated close: market price ±0.5bp
- [ ] Cross close: by loss ranking until recovery
- [ ] Partial close support
- [ ] Complete liquidation if needed
- [ ] Liquidation logging
- [ ] Failure escalation to manual

### 3.5 HL Liquidation Sync (T4)

- [ ] Detect HL liquidation events
- [ ] Send close orders to HL
- [ ] Reconcile after close

### 3.6 HL Precision Replication (T4)

- [ ] Maintenance rate parity with HL
- [ ] Mark price calculation replication
- [ ] Liquidation precision verification (>100 liquidations)

### 3.7 Complete PnL Settlement (T5)

- [ ] PnL = fill proceeds - open cost ± funding - fees
- [ ] Calculate on close
- [ ] Precision ≥99.99% vs HL
- [ ] Show floating PnL (real-time)
- [ ] Show realized PnL (closed positions)
- [ ] User query by position/symbol/time

### 3.8 Funding Rate Settlement (T5)

- [ ] 8-hour settlement cycle
- [ ] Rate source: HL WebSocket
- [ ] Settlement times: UTC 0:00/8:00/16:00
- [ ] INTERNAL: platform pays user
- [ ] HL: HL account bears cost
- [ ] Funding logging
- [ ] User query support

### 3.9 Three-Party Reconciliation (T5)

- [ ] Three parties: user / platform B-book / HL
- [ ] Position reconciliation
- [ ] Balance reconciliation
- [ ] Daily auto-check
- [ ] >0.1% variance → alert
- [ ] >1% variance → pause trading

### 3.10 Risk Dashboard (T6)

- [ ] Net exposure display (real-time)
- [ ] PnL by symbol/user
- [ ] Liquidation statistics
- [ ] Risk reserve balance
- [ ] HL account status
- [ ] Deviation log

### 3.11 Phase 3 Canary Pass Criteria

- [ ] Liquidation latency <1s (100%)
- [ ] PnL error <0.1% vs HL
- [ ] Reserve sufficient (>$500K)
- [ ] Availability >99.95%
- [ ] No user asset loss
- [ ] <5-min P0 response
- [ ] >$1M daily volume target
- [ ] >50 liquidations/day verify

---

## General Performance Targets

| Operation | Target | Pass |
|-----------|--------|------|
| Route decision | <5ms P99 | ✓ |
| HL forward | <50ms P99 | ✓ |
| Internalization fill | <10ms | ✓ |
| Liquidation detection | <1s | ✓ |
| API response | <100ms P95 | ✓ |
| Snapshot consistency | <5min | ✓ |
| Availability | >99.9% | ✓ |

---

## Test Coverage Requirements

- Unit tests: ≥80% coverage per team
- Integration: all cross-team paths per phase
- Load: Phase 2 (1000 ord/s), Phase 3 (5000 liquidations/hr)
- Canary: 7 days zero failures

---

## Sign-Off

This checklist must be completed and approved by:
- Product Lead: ___________
- Tech Lead: ___________
- Risk Manager: ___________
- QA Lead: ___________

Date: ________________
