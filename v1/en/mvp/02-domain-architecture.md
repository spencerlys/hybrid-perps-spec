---
doc_id: "domain-architecture-002"
title: "Ledger & Trading Domain and Risk Control Domain — Decoupled Architecture, Interface Contracts, Invocation Rules"
tags: ["architecture design", "domain design", "decoupling", "interface contracts", "MVP"]
version: "1.0"
lang: "en"
updated: "2026-04-09"
phase: "Phase 2"
---

# Ledger & Trading Domain and Risk Control Domain — Decoupled Architecture, Interface Contracts, Invocation Rules

## Overview

Platform's core system splits into two independent business domains, decoupled by strict interface contracts and unidirectional dependencies:

1. **Ledger & Trading Domain** — Manages user assets, order flow, position records, PnL calculation
2. **Risk Control Domain** — Manages pre-trade approval, real-time exposure monitoring, liquidation execution, risk reserve management

Both domains strictly forbid circular dependencies and communicate via async message queue and command dispatch.

---

## Domain 1: Ledger & Trading Domain

### Responsibility Scope

- **User Asset Management** (L1): Balance, frozen margin, used cross-margin, unrealized PnL
- **Order Reception & Validation** (L2): Order format validation, signature verification, basic rule checks (not risk control)
- **Order Routing Decision** (L2): Based on size, mode, and risk reserve status, decide INTERNAL or HL routing
- **Internalization Execution** (L3): INTERNAL fill accounting, position initialization, fill fee into balance
- **HL Proxy Execution** (L4): HL API relay, receipt verification, price correction, HL position mapping to user account
- **Market Data Maintenance** (L7): HL market data caching, mark prices, funding rates, on-chain data sync
- **Settlement & Reconciliation** (L8): PnL calculation, funding rate settlement, reconciliation correction, trade record maintenance

### Core Modules

| Module | Function | Key Output |
|--------|----------|-----------|
| L1 Account & Assets | User registration, multi-chain deposit/withdrawal, balance management | Account ID, wallet address, balance snapshot |
| L2 Order Routing | Order classification, three-mode routing, threshold decision | Routing instruction (INTERNAL/HL), Order ID |
| L3 Platform B-book Execution | INTERNAL fill, position accounting, fee settlement | Fill confirmation, Position ID, fill price |
| L4 HL Proxy Execution | HL order submission, receipt mapping, price correction | HL Order ID, fill price, error receipt |
| L7 Market Data | Market subscription, caching, sync | Mark price, funding rate, depth data |
| L8 Settlement & Reconciliation | PnL calculation, funding fee settlement, reconciliation | Settlement record, profit/loss voucher, reconciliation report |

### Forbidden Operations

❌ Direct balance modification (except fills/liquidation/withdrawal)
❌ Self-triggered liquidation (must be commanded by Risk domain)
❌ Hedge decision-making (entirely L6 responsibility)
❌ Risk rule checking (outside order format validation scope)

---

## Domain 2: Risk Control Domain

### Responsibility Scope

- **Pre-Trade Risk Approval**: Check sufficient margin, position limits, user risk level, symbol restrictions before order submission
- **Intra-Day Risk Monitoring**: Real-time exposure calculation, margin ratio monitoring, large order alerts, anomaly detection
- **Liquidation Detection & Orchestration**: Margin ratio trigger monitoring, liquidation priority sorting, execution orchestration (INTERNAL first, HL second)
- **Hedge Engine**: Net exposure aggregation, hedge threshold judgment, hedge ratio calculation, hedge instruction dispatch
- **Risk Reserve Management**: Reserve net value calculation, inflow/outflow, threshold alerting, freeze new INTERNAL orders when reserve insufficient
- **Limit System**: User daily trade amount limit, risk value limit, position limit, cross-symbol association limits
- **Circuit Breaker**: Extreme volatility breaker (>5%/hour), HL connection failure breaker, liquidation path failure breaker

### Core Modules

| Module | Function | Key Output |
|--------|----------|-----------|
| Pre-Trade Risk Check | Pre-order approval | Approved/Rejected, reason, warnings |
| Exposure Monitor | Real-time exposure calculation, aggregation | Net exposure, direction, real-time update |
| Margin & Liquidation Engine | Margin ratio calculation, liquidation trigger | Margin ratio, liquidation signal, execution ID |
| Hedge Engine (L6) | Hedge threshold, ratio, instruction generation | Hedge instruction, priority, target account |
| Risk Reserve Management | Reserve net value, liquidity management | Reserve balance, available limit, inflow record |
| Circuit Breaker | Multi-dimensional breaker monitoring | Breaker status, trigger reason, recovery condition |

### Unidirectional Dependency Rules

**Risk Domain → Ledger Domain**: Read-only + Send commands
- ✅ Read: User balance, positions, net exposure, trade records
- ✅ Send commands: Liquidation execution, hedge instructions, routing mode switch, freeze operations

**Ledger Domain → Risk Domain**: Send events only
- ✅ Send events: Exposure changes (fills, closes), position changes, fill notifications, liquidation execution results

**Forbidden Reverse Calls**
- ❌ Risk domain cannot directly modify ledger data
- ❌ Ledger domain cannot self-trigger liquidation, hedging, circuit breaker
- ❌ No synchronous RPC calls between domains

---

## 6 Core Interface Definitions

### 1️⃣ PreTradeCheck (Pre-Trade Risk Approval)

**Purpose**: Verify risk control conditions are met before order submission

**Request Parameters**

```json
{
  "request_id": "order_req_20260409_001",
  "user_id": "user_12345",
  "symbol": "BTC",
  "side": "LONG",
  "size": 0.5,
  "notional": 20000,
  "leverage": 10,
  "margin_mode": "ISOLATED",
  "order_type": "MARKET",
  "timestamp": 1712688000000
}
```

**Response Parameters**

```json
{
  "approved": true,
  "order_id": "order_20260409_001",
  "reject_reason": null,
  "risk_warnings": [
    "account_equity_low: Used margin is 92% of total equity, consider funding account"
  ],
  "metadata": {
    "margin_ratio": 0.92,
    "max_leverage_allowed": 15,
    "user_risk_level": "MEDIUM"
  }
}
```

**Error Codes**

| Error Code | Meaning | HTTP Code |
|-----------|---------|-----------|
| INSUFFICIENT_MARGIN | Insufficient margin | 400 |
| SYMBOL_SUSPENDED | Symbol trading suspended | 400 |
| EXPOSURE_LIMIT_EXCEEDED | Exposure limit exceeded | 400 |
| USER_RESTRICTED | User restricted | 403 |
| RESERVE_INSUFFICIENT | Risk reserve insufficient, cannot open INTERNAL order | 400 |
| LEVERAGE_EXCEED | Leverage exceeds user limit | 400 |

**Idempotency Strategy**

- Idempotency key: `request_id`
- Cache duration: 100ms (prevent duplicate checks)
- Timeout retry: No retry (handled by order layer)

**Timeout Handling**

- Target latency: <10ms P99
- Timeout duration: 100ms
- Timeout behavior: Return 500 error, don't allow order

**Caller**: L2 Order Routing module
**Frequency**: Per order

---

### 2️⃣ ExposureChangeEvent (Exposure Change Event)

**Purpose**: Ledger domain notifies Risk domain of INTERNAL exposure change (async, non-blocking)

**Message Format**

```json
{
  "event_id": "exposure_evt_20260409_001",
  "event_type": "OPEN",
  "user_id": "user_12345",
  "symbol": "BTC",
  "direction": "LONG",
  "position_size": 0.5,
  "entry_price": 40000,
  "delta_notional": 20000,
  "current_exposure": {
    "net_exposure": 45000,
    "long_notional": 65000,
    "short_notional": 20000
  },
  "timestamp": 1712688000000,
  "source": "L3_EXECUTION"
}
```

**Producer**: L3 (internalization execution), L4 (HL receipt)
**Consumer**: L6 (hedge engine, exposure monitoring)
**Transport**: Redis Streams, at-least-once
**Ensure Monotonicity**: Consumer uses message ID sorting, idempotent dedup

**Timeout Handling**

- Queue congestion alert: >10,000 unconsumed messages
- Consumption delay alert: >5 seconds
- Failure recovery: Manual full reconciliation trigger

---

### 3️⃣ LiquidationTrigger (Liquidation Command)

**Purpose**: Risk domain commands Ledger domain to execute user position liquidation

**Request Parameters**

```json
{
  "trigger_id": "liq_trg_20260409_001",
  "position_id": "pos_20260408_123",
  "user_id": "user_12345",
  "symbol": "BTC",
  "direction": "LONG",
  "position_size": 0.5,
  "trigger_price": 38000,
  "margin_ratio": 0.095,
  "liquidation_type": "ISOLATED",
  "execution_target": "INTERNAL",
  "urgency": "HIGH",
  "timestamp": 1712688000000
}
```

**Response Parameters**

```json
{
  "execution_id": "liq_exe_20260409_001",
  "status": "EXECUTING",
  "position_id": "pos_20260408_123",
  "close_price": null,
  "filled_size": 0,
  "remaining_size": 0.5,
  "estimated_loss": -2000,
  "liquidation_fee": 2000,
  "timestamp": 1712688000000
}
```

**Final Response (After Fill)**

```json
{
  "execution_id": "liq_exe_20260409_001",
  "status": "COMPLETED",
  "close_price": 38500,
  "filled_size": 0.5,
  "actual_loss": -2250,
  "liquidation_fee": 2000,
  "platform_profit": 1600,
  "reserve_fund_contribution": 400,
  "settlement_time_ms": 2500
}
```

**Error Codes**

| Error Code | Meaning |
|-----------|---------|
| POSITION_ALREADY_CLOSED | Position already closed |
| POSITION_NOT_FOUND | Position not found |
| HL_UNAVAILABLE | HL liquidation failed (HL-side position liquidated) |
| EXECUTION_TIMEOUT | Liquidation execution timeout (>5s) |
| INSUFFICIENT_LIQUIDITY | Insufficient liquidation liquidity |
| UNAUTHORIZED | Unauthorized command source |

**Idempotency Strategy**

- Idempotency key: `trigger_id`
- Executed check: Query fill records, return completed if exists
- Retry strategy: 3 retries, 1s interval, alert on failure

**Timeout Handling**

- Target latency: <2000ms (liquidation timeliness requirement)
- Timeout duration: 5000ms
- After timeout: Mark as TIMEOUT, Risk alert, consider entering HL_MODE

**Caller**: L6 Risk domain
**Frequency**: Event-driven (margin ratio trigger)

---

### 4️⃣ HedgeInstruction (Hedge Command)

**Purpose**: Risk domain L6 commands Ledger domain L4 to hedge platform net exposure on HL

**Request Parameters**

```json
{
  "instruction_id": "hedge_instr_20260409_001",
  "symbol": "BTC",
  "direction": "SHORT",
  "size": 0.4,
  "leverage": 10,
  "max_slippage_bps": 100,
  "urgency": "NORMAL",
  "reason": "net_exposure_500k_exceeded",
  "timestamp": 1712688000000
}
```

**Response Parameters**

```json
{
  "execution_id": "hedge_exe_20260409_001",
  "status": "EXECUTING",
  "symbol": "BTC",
  "direction": "SHORT",
  "order_size": 0.4,
  "hl_order_id": "hl_order_99999",
  "timestamp": 1712688000000
}
```

**Final Response (After Fill)**

```json
{
  "execution_id": "hedge_exe_20260409_001",
  "status": "COMPLETED",
  "symbol": "BTC",
  "fill_price": 40100,
  "filled_size": 0.4,
  "notional_filled": 16040,
  "taker_fee": 5.61,
  "net_cost": 5.61,
  "remaining_exposure": 100000,
  "settlement_time_ms": 3200
}
```

**Error Codes**

| Error Code | Meaning |
|-----------|---------|
| HEDGE_ACCOUNT_INSUFFICIENT | Hedge account margin insufficient |
| SLIPPAGE_EXCEEDED | Slippage exceeds limit |
| HL_UNAVAILABLE | HL connection interrupted |
| EXECUTION_TIMEOUT | Hedge instruction timeout |
| INVALID_SIZE | Hedge size exceeds limit |

**Idempotency Strategy**

- Idempotency key: `instruction_id`
- Query HL orders, check filled portions
- Executed check: HL order status = FILLED

**Timeout Handling**

- Target latency: <500ms (hedge execution)
- Timeout duration: 30000ms
- After timeout: Auto-retry OR cancel order, alert

**Caller**: L6 Hedge engine
**Frequency**: Threshold-triggered (on exposure change)

---

### 5️⃣ RoutingModeChange (Routing Mode Change)

**Purpose**: Switch L2's three routing modes (NORMAL_MODE / HL_MODE / BETTING_MODE)

**Request Parameters**

```json
{
  "change_id": "mode_chg_20260409_001",
  "new_mode": "HL_MODE",
  "trigger_type": "AUTO",
  "operator_id": null,
  "reason": "net_exposure_exceeded_1m_threshold",
  "enforced": false,
  "timestamp": 1712688000000
}
```

**Response Parameters**

```json
{
  "applied": true,
  "change_id": "mode_chg_20260409_001",
  "previous_mode": "NORMAL_MODE",
  "new_mode": "HL_MODE",
  "effective_at": 1712688000100,
  "effective_for_ms": 0,
  "auto_fallback_threshold": {
    "trigger": "net_exposure_below_500k",
    "fallback_mode": "NORMAL_MODE"
  }
}
```

**Error Codes**

| Error Code | Meaning |
|-----------|---------|
| ALREADY_IN_MODE | Already in target mode |
| UNAUTHORIZED | Operator unauthorized |
| INVALID_MODE | Invalid mode |

**Idempotency Strategy**

- Idempotency key: `change_id`
- Executed check: Current mode = new_mode

**Timeout Handling**

- Target latency: <100ms
- Timeout duration: 1000ms
- After timeout: Error, don't modify mode

**Caller**: L6 Risk domain (auto), L9 Admin (manual)
**Frequency**: Low (minute-level)

---

### 6️⃣ BalanceQuery (Balance & Position Query - Read-Only)

**Purpose**: Risk domain queries user account state for decision-making

**Request Parameters**

```json
{
  "user_id": "user_12345",
  "include_details": false
}
```

**Response Parameters**

```json
{
  "user_id": "user_12345",
  "total_equity": 100000,
  "available_balance": 5000,
  "frozen_margin": 60000,
  "cross_margin_used": 35000,
  "unrealized_pnl": 2500,
  "positions": {
    "internal": [
      {
        "position_id": "pos_001",
        "symbol": "BTC",
        "direction": "LONG",
        "size": 0.5,
        "entry_price": 40000,
        "current_price": 41000,
        "margin_used": 20000,
        "margin_mode": "ISOLATED",
        "unrealized_pnl": 500
      }
    ],
    "hyperliquid": [
      {
        "symbol": "ETH",
        "direction": "SHORT",
        "size": 10,
        "entry_price": 2000,
        "current_price": 1950,
        "unrealized_pnl": 500
      }
    ]
  },
  "margin_ratios": {
    "account_level": 0.92,
    "isolated_positions": [0.85, 0.78]
  }
}
```

**Error Codes**

| Error Code | Meaning |
|-----------|---------|
| USER_NOT_FOUND | User not found |
| ACCOUNT_LOCKED | Account locked |

**Timeout Handling**

- Target latency: <50ms
- Timeout duration: 200ms
- After timeout: Return cached value (TTL 5s)

**Caller**: L6 Hedge engine, liquidation engine, pre-trade risk check
**Frequency**: High (multiple times per second)

---

## Cross-Domain Consistency Strategy

### Principle: Sync as Boundary, Async to Supplement

| Category | Communication | Consistency | Path |
|----------|---------------|-------------|------|
| Core path: order → fill → accounting | Synchronous transaction | ACID | Database transaction + distributed lock |
| Cross-domain notify: exposure/position change | Async message | At-least-once | Redis Streams + consumer idempotent |
| Command dispatch: liquidation/hedge/mode | Async message | At-least-once | Command queue + retry + idempotency key |
| Read operation: query balance/position | Sync query | Eventual | Cache + TTL + version number |

### Reconciliation Correction Process

**Periodic Full Reconciliation** (every hour)

```
1. Ledger domain export: user total equity, total positions, total exposure, total risk value
2. Risk domain export: latest risk monitoring snapshot, liquidation blacklist, freeze list
3. Reconcile dimensions:
   - Is user total equity consistent?
   - Is net exposure value consistent?
   - Are liquidated positions synchronized?
4. Discover discrepancy → Generate audit report → Alert → Manual review → Compensating transaction
```

**Compensating Transaction Examples**

| Discrepancy | Root Cause | Compensation |
|----------|----------|---------------|
| Liquidation failed, position not closed | HL connection lost or timeout | Retry 3 times (1s interval) → Alert → Manual confirm → Force HL_MODE |
| Exposure value mismatch | Fill event lost | Scan fill records → Resend exposure event → Recalculate hedge |
| Risk reserve balance incorrect | Inflow record lost | Check user loss vouchers → Supplement entry → Alert |
| User balance tampered | Abnormal operation log | Rollback to last correct version + manual audit |

---

## Message Queue Design

### Topic 1: exposure_changes (Exposure Changes)

- **Producer**: L3 (INTERNAL fills), L4 (HL receipts)
- **Consumer**: L6 (hedge engine, real-time exposure calculation)
- **Message Format**: See ExposureChangeEvent
- **Frequency**: ~50-200 messages/second (depends on trade volume)
- **Sharding**: By user ID (ensure same-user message ordering)
- **Retry**: Consumption failure → dead letter queue → manual handling

### Topic 2: liquidation_triggers (Liquidation Commands)

- **Producer**: L5 (margin ratio monitoring)
- **Consumer**: L3 (INTERNAL liquidation), L4 (HL liquidation)
- **Message Format**: See LiquidationTrigger
- **Frequency**: Low (event-driven)
- **Priority**: INTERNAL liquidation first, HL second
- **Retry**: Max 3 times, alert + manual takeover on failure

### Topic 3: hedge_instructions (Hedge Commands)

- **Producer**: L6 (hedge engine)
- **Consumer**: L4 (HL hedge account execution)
- **Message Format**: See HedgeInstruction
- **Frequency**: Medium (threshold-triggered, typically 5-10/hour)
- **Confirmation mechanism**: L4 returns execution_id after consuming, L6 tracks fill
- **Failure handling**: Alert + manual confirmation → rehedge OR enter HL_MODE

### Topic 4: mode_changes (Routing Mode Changes)

- **Producer**: L6 (auto-trigger), L9 (manual operation)
- **Consumer**: L2 (order routing logic), L9 (dashboard update)
- **Message Format**: See RoutingModeChange
- **Frequency**: Low (minute-level)
- **Ordering**: Strict single-thread consumption, avoid mode conflict
- **Idempotency**: change_id dedup, duplicate messages harmless

---

## Implementation Highlights for Developers

### 1. Sync Transaction vs Async Message Boundary

**Synchronous (Strong Consistency)**
- Order creation → Format validation → Routing decision → L3/L4 fill → Balance update
- Entire chain completes within database transaction
- All-or-nothing rollback, client gets success/failure synchronously

**Asynchronous (Eventual Consistency)**
- Post-fill, immediately publish to exposure_changes queue
- L6 async consumes, calculates net exposure, decides hedge
- Delay acceptable (<100ms)

**Code Example**
```python
# Sync path: order processing
with db.transaction():
    order = Order.create(user_id, symbol, side, size)
    if route == 'INTERNAL':
        position = Position.create(order)
        balance.update(fee=-fee_amount)
    else:
        hl_order = hl_api.create_order(...)
    db.commit()

# Async notify: exposure change
message_queue.publish('exposure_changes', {
    'event_id': uuid4(),
    'user_id': order.user_id,
    'delta_notional': order.notional,
    ...
})
```

### 2. Idempotency Key Generation & Storage

Each cross-domain interface has an idempotency key:
- PreTradeCheck: `request_id` (client-generated + timestamp)
- LiquidationTrigger: `trigger_id` (risk system-generated)
- HedgeInstruction: `instruction_id` (L6-generated)
- RoutingModeChange: `change_id` (control system-generated)

**Storage Schema**
```sql
CREATE TABLE idempotency_keys (
    key_type VARCHAR(50),
    key_value VARCHAR(256),
    result JSON,
    created_at TIMESTAMP,
    PRIMARY KEY (key_type, key_value),
    INDEX (created_at)
);
```

Check: `SELECT result FROM idempotency_keys WHERE key_value = ?`

### 3. Message Queue Ordering & Dedup

**Partition by User**: Ensure same-user exposure events ordered
```python
partition_key = f"user_{event.user_id}"
message_queue.publish('exposure_changes', event, partition=partition_key)
```

**Consumer Dedup**: Idempotent consumer
```python
def consume_exposure_event(event):
    last_event_id = redis.get(f"last_event_{event.user_id}")
    if event.event_id <= last_event_id:
        return  # Already processed, skip

    # Process event
    update_exposure(event)
    redis.set(f"last_event_{event.user_id}", event.event_id)
```

### 4. Risk Interface Timeout Degradation

PreTradeCheck must be <10ms, else impacts trading UX

**Caching Strategy**
```python
def pre_trade_check(order):
    cache_key = f"ptc_{order.user_id}_{order.symbol}"
    cached = cache.get(cache_key)
    if cached and cached['timestamp'] > now - 100ms:
        return cached['result']

    try:
        result = risk_service.check(order, timeout=10ms)
        cache.set(cache_key, result, ttl=100ms)
        return result
    except TimeoutError:
        # Use last result or default reject
        return cached or {'approved': False}
```

### 5. Liquidation Execution Retry & Alert

Liquidation is high-priority, must have retry mechanism

```python
def execute_liquidation(trigger):
    for retry in range(3):
        try:
            result = liquidation_service.execute(trigger, timeout=5000ms)
            if result.status == 'COMPLETED':
                return result
        except Exception as e:
            if retry < 2:
                sleep(1)
                continue
            else:
                alert(f"Liquidation {trigger.trigger_id} failed after 3 retries", severity=CRITICAL)
                # Force HL_MODE, manual handling
                routing_mode.set('HL_MODE')
                raise
```

### 6. Reconciliation Correction Automation & Manual Intervention

**Auto-Correct Small Discrepancies**
- Discrepancy <$100: Auto-correct
- Generate audit log after correction

**Manual Review Large Discrepancies**
- Discrepancy ≥$100: Pause, alert
- Generate reconciliation report, await manual approval
- Execute compensating transaction after approval

```python
def reconcile():
    discrepancies = ledger_domain.export() ^ risk_domain.export()

    for disc in discrepancies:
        if abs(disc.amount) < 100:
            auto_correct(disc)
        else:
            create_audit_ticket(disc, status='PENDING_REVIEW')
            alert(f"Large discrepancy detected: {disc.amount}")
```

---

## Summary

Via clear domain boundaries, unidirectional dependencies, and sync+async hybrid communication, we achieve:

✅ **Decoupling**: Both domains evolve independently, stable interfaces
✅ **Observability**: Async messages fully traceable across chain
✅ **Fault Tolerance**: Liquidation/hedge has retry + manual backstop
✅ **Consistency**: Sync transactions + async reconciliation double guarantee
✅ **Efficiency**: Core path sync <10ms, non-critical async non-blocking
