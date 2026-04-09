# Unified Architecture Baseline (MVP)

## Design Principles

### Principle 1: Minimum Viable Cost (Survivability First)

- **Reuse over building**: If HL already provides functionality (prices, order management, margin), use their API rather than reimplementing
- **Configuration over hardcoding**: Thresholds, fees, leverage limits should live in config tables for rapid adjustment during canary period
- **Async over blocking**: Hedge instructions, liquidation orchestration, and event notifications must be asynchronous to keep order response <100ms
- **Manual backstop over automation**: In Phase 2, we can manually intervene on deviation corrections, abnormal liquidations, and rollback decisions

### Principle 2: Two-Domain Decoupling

The entire system splits into two independent domains communicating via event bus:

- **Ledger & Trading Domain**
  - Responsibilities: User accounts, order routing, internalization execution, HL proxy, market data, settlement reconciliation
  - Characteristics: Transaction logic-driven, high responsiveness requirement, automatic recovery on failure

- **Risk Control Domain**
  - Responsibilities: Pre-trade risk control, intra-day exposure monitoring, liquidation orchestration, hedge instruction dispatch, risk reserve management
  - Characteristics: Safety first, every decision audited and tracked, active circuit breaker on failure

**Forbidden Rules**:
- No synchronous RPC calls between domains (message queue only)
- No circular dependencies (A→B→A)
- Risk Control Domain cannot directly modify ledger data; must trigger execution via events

### Principle 3: Dual Account Isolation

Platform maintains two separate accounts on HL, physically isolated with independent risk limits:

- **Trading Account**
  - Purpose: Handle large user orders (>$10K), all orders in HL_MODE
  - Minimum capital: ≥$500K
  - Target margin ratio: >500% (never trigger HL liquidation)

- **Hedge Account**
  - Purpose: Execute net exposure hedge instructions
  - Minimum capital: ≥$200K
  - Target margin ratio: >300%
  - Purpose: Absorb internal exposure, isolate trading account risk

**Monitoring Rules**:
- Monitor both account balances independently, sync every 5 minutes
- If either account margin ratio drops below 200%, immediate alert + enter HL_MODE
- After hedging, both accounts should satisfy: |Trading Exposure| < $100K

### Principle 4: Approach 2 Purity

- **Internalization not matching**: Platform is sole counterparty for all ≤$10K orders, no user-to-user matching exists
- **Single risk bearer**: User loss → platform gain; user profit → platform loss
- **Closed risk loop**: User PnL + Hedge Cost + Reserve = Platform PnL
- **Unified terminology**: Never use "matching", "matching engine", or "matching queue"; always use "internalization"

### Principle 5: MVP-First Mentality

- **Close loop before perfecting**: Prioritize minimum viable path, don't chase completeness
- **Launch before optimizing**: Discover and adjust during canary period, avoid over-design
- **Config over extensibility**: Rapid strategy adjustment via config, no code changes needed
- **Alert for missing pieces**: Instrumentation at every critical path step; visibility first

---

## Two-Domain Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                   API Gateway (R0 - API Relay)                  │
│                  HL WebSocket/REST → format conversion          │
└──────────────────────────┬─────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           │            Event Bus          │
           │    (Redis Streams / Kafka)    │
           │               │               │
    ┌──────▼──────────┐   │     ┌─────────▼──────────┐
    │  Ledger & Trade │────┼────▶│ Risk & Safety     │
    │   Domain        │   │     │ Domain            │
    │                 │   │     │                   │
    ├─────────────────┤   │     ├──────────────────┤
    │                 │   │     │                  │
    │ • L1 Account    │───┼─┐   │ • Pre-trade      │
    │   Assets        │   │ │   │   Risk Control   │
    │                 │   │ │   │                  │
    │ • L2 Routing    │───┼─┤   │ • Intra-day      │
    │   Engine        │   │ │   │   Exposure       │
    │                 │   │ │   │   Monitoring     │
    │ • L3 Internal   │◀──┼─┤   │                  │
    │   Execution     │   │ │   │ • Liquidation    │
    │                 │   │ │   │   Orchestration  │
    │ • L4 HL Proxy   │───┼─┤   │                  │
    │   Execution     │   │ │   │ • L6 Hedge      │
    │                 │   │ │   │   Engine         │
    │ • L7 Market     │───┼─┐   │ • Risk Reserve  │
    │   Data          │   │ │   │   Management     │
    │                 │   │ │   │                  │
    │ • L8 Settlement │───┼─┤   │ • Limits &      │
    │   Reconciliation│   │ │   │   Circuit       │
    │                 │   │ │   │   Breaker       │
    │ • L9a Admin     │───┼─┘   │ • L9b Admin     │
    │   (Trading)     │   │     │   (Risk)        │
    │                 │   │     │                 │
    └────────┬────────┘   │     └────────┬────────┘
             │            │             │
             └─────┬──────┴──────┬──────┘
                   │             │
        ┌──────────▼──┐ ┌───────▼───────┐
        │ HL Trading  │ │ HL Hedge      │
        │ Account     │ │ Account       │
        │ ≥$500K      │ │ ≥$200K        │
        └─────────────┘ └───────────────┘
             │                  │
    ┌────────┼──────────────────┼────────┐
    │        │                  │        │
 ┌──▼─┐ ┌──▼──┐  ┌──────┐  ┌──▼─┐ ┌──▼──┐
 │TRON│ │Eth  │  │Solana│  │   │ │     │
 │Wallet│ │Wallet│  │Wallet│  │   │ │     │
 └────┘ └─────┘  └──────┘  └───┘ └─────┘
```

**Legend**:
- Solid arrows (`──▶`): Async event push
- Dotted arrows (`◀──`): Query/feedback
- Bold arrows (`──┼──`): Critical communication

---

## Two-Domain Interface Contracts

### Interface A: Order Request Validation (Ledger Domain → Risk Domain)

**Trigger**: User submits new order, after L2 routing decision completes

**Request Message**

```
Message: ORDER_SUBMITTED
{
  "request_id": "req_20260409_001",      // Idempotency key
  "timestamp": 1712639999000,             // Unix ms
  "user_id": "usr_alice_001",
  "order_id": "ord_20260409_001",
  "symbol": "BTC-USD",
  "side": "LONG",                         // LONG / SHORT
  "size": 0.5,                            // BTC
  "notional": 21500,                      // USD
  "leverage": 2,                          // Platform limit ≤ 10x
  "margin_mode": "ISOLATED",              // ISOLATED / CROSS
  "route": "INTERNAL",                    // INTERNAL / HYPERLIQUID
  "limit_price": 43000.00,                // Non-null for limit orders only
  "order_type": "LIMIT"                   // MARKET / LIMIT
}
```

**Response Message**

Success:
```
Message: ORDER_APPROVED
{
  "request_id": "req_20260409_001",
  "order_id": "ord_20260409_001",
  "approved": true,
  "reasons": []
}
```

Rejection:
```
Message: ORDER_REJECTED
{
  "request_id": "req_20260409_001",
  "order_id": "ord_20260409_001",
  "approved": false,
  "error_code": "RISK_EXPOSURE_EXCEED",  // See table below
  "reason": "Net exposure after order would exceed $1M limit",
  "suggested_action": "RETRY_IN_BETTING_MODE"
}
```

**Error Codes**

| Error Code | Meaning | Recovery |
|-----------|---------|----------|
| `RISK_EXPOSURE_EXCEED` | Net exposure limit exceeded | Reduce position size / Risk manager manual adjustment |
| `DAILY_LIMIT_EXCEED` | Daily trading limit reached | Wait until next day |
| `RISK_RESERVE_LOW` | Insufficient risk reserve | Manual reserve top-up |
| `ROUTING_MODE_HL_ONLY` | Currently HL-only routing active | Use large order channel |

**Idempotency Key**: `request_id` (cache for 24 hours, return same result for duplicates)

**Timeout**: <10ms (after async conversion, still requires early response, converted to event notification)

---

### Interface B: Exposure Change Event (Ledger Domain → Risk Domain)

**Trigger**: Order filled, partially filled, position closed, or liquidation executed

**Event Message**

```
Message: EXPOSURE_CHANGED
{
  "event_id": "evt_20260409_001",
  "event_type": "ORDER_FILLED",          // ORDER_FILLED / PARTIAL_FILLED / POSITION_CLOSED / LIQUIDATED
  "timestamp": 1712640100000,
  "user_id": "usr_alice_001",
  "symbol": "BTC-USD",
  "side": "LONG",
  "delta_size": 0.5,                     // Size delta this event
  "delta_notional": 21500,               // Notional delta
  "execution_price": 43000.00,           // Actual fill price
  "route": "INTERNAL",                   // Which system executed

  // Cumulative exposure snapshot
  "snapshot": {
    "total_internal_long": 2.5,          // BTC
    "total_internal_short": 0.0,
    "total_hl_long": 1.0,
    "total_hl_short": 0.0,
    "net_exposure": {
      "BTC": {
        "internal": 2.5,
        "hyperliquid": 1.0,
        "notional": "128500",            // USD
        "direction": "LONG"
      }
    },
    "user_total_equity": "150000"        // USD
  }
}
```

**Risk Domain Response**

```
Message: EXPOSURE_ACKNOWLEDGED
{
  "event_id": "evt_20260409_001",
  "status": "PROCESSED",
  "hedge_triggered": false,              // Whether hedging was triggered
  "hedge_job_id": null,
  "actions": []
}
```

Or

```
Message: HEDGE_INITIATED
{
  "event_id": "evt_20260409_001",
  "status": "HEDGE_QUEUED",
  "hedge_job_id": "hedge_20260409_001",
  "hedge_instruction": {
    "symbol": "BTC-USD",
    "direction": "SHORT",                // Opposite of internal exposure
    "size": 1.25,                        // 50% hedge ratio
    "leverage": 1,
    "target_account": "HEDGE"
  }
}
```

**Error Codes**

| Error Code | Meaning | Recovery |
|-----------|---------|----------|
| `SNAPSHOT_MISMATCH` | Exposure snapshot mismatch with risk records | Trigger reconciliation, manual review |
| `HEDGE_QUEUE_FULL` | Hedge queue full | Wait for queue processing, reduce inflow rate |
| `STALE_EVENT` | Event timestamp too old | Ledger domain resend |

**Idempotency Key**: `event_id`

**Timeout**: <50ms (async processing, risk domain should persist within this time)

---

### Interface C: Liquidation Command (Risk Domain → Ledger Domain)

**Trigger**: Risk domain detects margin ratio breaching liquidation threshold

**Command Message**

```
Message: LIQUIDATION_COMMAND
{
  "command_id": "liq_20260409_001",
  "timestamp": 1712640200000,
  "user_id": "usr_alice_001",
  "trigger_type": "MARGIN_RATIO_BREACH",  // MARGIN_RATIO / PRICE_LIMIT / MANUAL

  "positions": [
    {
      "position_id": "pos_20260409_001",
      "symbol": "BTC-USD",
      "side": "LONG",
      "size": 0.5,
      "route": "INTERNAL",                // Which system holds position
      "margin_mode": "ISOLATED"           // Isolated liquidation
    }
  ],

  "liquidation_type": "PARTIAL",         // PARTIAL / FULL_ACCOUNT
  "execution_target": "INTERNAL",        // Which system to close in
  "priority": 1,                          // 1(urgent) ~ 5(normal)
  "timeout_ms": 5000                     // Max 5 seconds to liquidate
}
```

**Execution Result**

Success:
```
Message: LIQUIDATION_EXECUTED
{
  "command_id": "liq_20260409_001",
  "status": "COMPLETED",
  "positions_closed": ["pos_20260409_001"],
  "total_loss": 500.00,                 // User loss
  "platform_gain": 500.00               // Platform revenue
}
```

Failure:
```
Message: LIQUIDATION_FAILED
{
  "command_id": "liq_20260409_001",
  "status": "FAILED",
  "error_code": "INTERNAL_ORDER_QUEUE_FULL",
  "error_msg": "Cannot execute immediately, queue congested",
  "fallback_action": "ROUTE_TO_HL",      // Fallback to HL close
  "retry_count": 1,
  "max_retries": 3
}
```

**Error Codes**

| Error Code | Meaning | Auto-Rollback |
|-----------|---------|----------------|
| `INTERNAL_ORDER_QUEUE_FULL` | Internal order queue full | Yes → Route to HL |
| `INSUFFICIENT_HL_BALANCE` | HL account balance insufficient | Yes → Alert + manual backstop |
| `POSITION_ALREADY_CLOSED` | Position already closed | Yes → Mark complete |
| `LIQUIDATION_PRICE_MOVED` | Target price has drifted | Yes → Reprice + retry |

**Idempotency Key**: `command_id` (return same result if resubmitted within 3 hours, prevent duplicate liquidation)

**Timeout**: <1000ms (liquidation must respond quickly, risk domain initiates retries)

---

### Interface D: Hedge Instruction (Risk Domain → Ledger Domain L4)

**Trigger**: After exposure change, risk domain calculates required hedge position

**Command Message**

```
Message: HEDGE_INSTRUCTION
{
  "hedge_job_id": "hedge_20260409_001",
  "created_at": 1712640300000,

  "symbol": "BTC-USD",
  "direction": "SHORT",                  // Opposite of internal exposure
  "size": 1.25,                          // Hedge size (units: coins)
  "leverage": 1,                         // Hedge account typically 1x
  "target_account": "HEDGE",             // Fixed

  "hedge_ratio": 0.5,                    // 50% of current exposure
  "net_exposure_before": 2500,           // Net exposure before hedge (USD)
  "target_exposure_after": 1250,         // Target exposure after hedge

  "max_slippage_bps": 20,                // Max slippage 20 bps = 0.2%
  "timeout_ms": 30000,                   // 30-second hedge timeout
  "retry_policy": {
    "max_attempts": 3,
    "backoff_ms": 1000
  }
}
```

**Execution Result**

Success:
```
Message: HEDGE_COMPLETED
{
  "hedge_job_id": "hedge_20260409_001",
  "status": "COMPLETED",
  "execution_details": {
    "symbol": "BTC-USD",
    "direction": "SHORT",
    "size_executed": 1.25,
    "average_price": 42950.00,
    "total_cost": 1050.00,               // Hedge cost (spread)
    "slippage_bps": 15                   // Actual slippage 15 bps
  },
  "hl_order_id": "hl_ord_20260409_001",
  "new_net_exposure": 1250,
  "hedge_reserve_used": 100.00           // Deducted from reserve
}
```

**Error Codes**

| Error Code | Meaning | Handling |
|-----------|---------|----------|
| `HEDGE_PRICE_TOO_VOLATILE` | Price too volatile | Reduce hedge size → retry |
| `HEDGE_TIMEOUT` | Hedge instruction timeout | Downgrade to partial hedge → alert |
| `HEDGE_ACCOUNT_LOW_MARGIN` | Hedge account margin insufficient | Top-up hedge account |

**Idempotency Key**: `hedge_job_id`

**Timeout**: <30s (async execution, doesn't block order submission)

---

### Interface E: Routing Mode Change (Risk Domain → Ledger Domain L2)

**Trigger**: Risk domain assesses system risk state and needs routing strategy change

**Command Message**

```
Message: ROUTING_MODE_CHANGE
{
  "command_id": "route_change_20260409_001",
  "timestamp": 1712640400000,

  "new_mode": "HL_MODE",                 // HL_MODE / NORMAL_MODE / BETTING_MODE
  "trigger_reason": "HL_ACCOUNT_MARGIN_LOW",
  "trigger_details": {
    "current_margin_ratio": 240,         // %
    "threshold": 300,                    // %
    "deficit": "$60K"
  },

  "operator": "SYSTEM",                  // SYSTEM / admin_user_id
  "approval_required": false,            // MVP can auto-switch

  "effective_immediately": true,
  "fallback_mode": "NORMAL_MODE"         // Fallback mode on failure
}
```

**Execution Result**

```
Message: ROUTING_MODE_CHANGED
{
  "command_id": "route_change_20260409_001",
  "status": "COMPLETED",
  "old_mode": "NORMAL_MODE",
  "new_mode": "HL_MODE",
  "effective_at": 1712640401000,
  "pending_orders": {
    "affected_count": 5,
    "action": "ROUTE_PENDING_TO_HL"      // Route pending orders to HL
  }
}
```

**Error Codes**

| Error Code | Meaning | Fallback |
|-----------|---------|----------|
| `MODE_ALREADY_ACTIVE` | Already in target mode | Ignore |
| `INVALID_MODE_TRANSITION` | Invalid mode transition | Reject, don't execute |

**Idempotency Key**: `command_id`

**Timeout**: <5ms (routing mode change needs extremely fast response)

---

### Interface F: Balance & Risk Query (Risk Domain reads Ledger Domain)

**Trigger**: Risk domain needs latest account state before decision-making

**Query Request**

```
Message: ACCOUNT_STATE_QUERY
{
  "query_id": "query_20260409_001",
  "user_id": "usr_alice_001",
  "include_fields": [
    "available_balance",
    "total_equity",
    "margin_used",
    "margin_ratio",
    "unrealized_pnl",
    "exposure_by_symbol",
    "liquidation_price"
  ]
}
```

**Query Response**

```
Message: ACCOUNT_STATE_SNAPSHOT
{
  "query_id": "query_20260409_001",
  "timestamp": 1712640500000,
  "user_id": "usr_alice_001",

  "available_balance": 95000.00,         // Available funds
  "total_equity": 150000.00,             // Total equity = initial + PnL
  "margin_used": 55000.00,               // Used margin
  "margin_ratio": 272,                   // % (total equity / margin)

  "unrealized_pnl": 5000.00,             // Unrealized PnL
  "total_notional": 150000.00,           // Total notional exposure

  "exposure": {
    "BTC-USD": {
      "internal_long": 0.5,
      "internal_short": 0.0,
      "hl_long": 1.0,
      "hl_short": 0.0,
      "net_long": 1.5,
      "notional_usd": 64500,
      "liquidation_price": 20000.00      // Price at which liquidation occurs
    }
  },

  "cumulative_loss_today": 2000.00,      // Daily cumulative loss
  "cumulative_gain_today": 7000.00       // Daily cumulative gain
}
```

**Error Codes**

| Error Code | Meaning | Handling |
|-----------|---------|----------|
| `USER_NOT_FOUND` | User not found | Return 404 |
| `BALANCE_STALE` | Data older than 5 seconds | Return with stale flag |

**Timeout**: <50ms (queries should be very fast, use local cache + periodic refresh)

---

## Consistency Strategy

### Core Principle: Eventual Consistency + Periodic Reconciliation

**Assumption**: Domains communicate asynchronously via events, temporary inconsistencies possible. Periodic reconciliation and compensating transactions ensure eventual consistency.

### Reconciliation Frequency Table

| Item | Frequency | Tolerance | Overrun Action |
|------|-----------|-----------|----------------|
| **User Balance Real-time** | Real-time | ±$1 | Mark alert, freeze account |
| **HL Position Mapping** | 5 minutes | ±0.01 coins | Send correction instruction |
| **B-book Exposure** | Hourly | ±$10K | Manual review + correction |
| **Risk Reserve** | Daily | ±$100 | Adjust next day |
| **User Equity** | Daily | ±$100 | Mark anomaly, manual review |

### Compensating Transactions

**Scenario 1: Liquidation Command Fails**

1. Risk domain sends `LIQUIDATION_COMMAND` to L3
2. L3 should return `LIQUIDATION_EXECUTED` or `LIQUIDATION_FAILED` within 5 seconds
3. If failed:
   - If L3 queue full → L3 auto-routes to L4 (HL liquidation)
   - If HL balance insufficient → Alert + manual intervention
   - If failed >3 times → Force HL_MODE, prevent further losses

**Scenario 2: Hedge Command Lost**

1. Risk domain sends `HEDGE_INSTRUCTION` to L4
2. L4 should return `HEDGE_COMPLETED` or `HEDGE_FAILED`
3. If no reply within 30 seconds → Risk domain queries HL exposure
4. If exposure unchanged → Resend hedge instruction with retry count
5. If failed 3 times → Alert + manual intervention

**Scenario 3: Exposure Snapshot Mismatch**

1. Ledger domain sends `EXPOSURE_CHANGED` event with snapshot
2. Risk domain recalculates user's cumulative exposure hourly, compares with snapshot
3. If delta > $10K → Send `REBALANCE_INSTRUCTION` to Ledger domain
4. Ledger domain adjusts internal books per instruction

### Invocation Rules

| Rule | Meaning | Violation Consequence |
|------|---------|----------------------|
| **Risk Domain Read-Only** | Risk domain can read all Ledger data, but cannot directly modify | Code review + architecture alert |
| **Modify via Events** | If Risk must modify Ledger (e.g., freeze account), use events | Transaction fails + manual backstop |
| **No RPC Sync** | No synchronous RPC between domains, all async | Timeout → auto-degrade |
| **Idempotency Key Required** | All cross-domain messages must carry idempotency key, receiver caches result | Duplicate processing → alert |
| **Timeout + Auto-Retry** | Message not acknowledged → sender auto-retries, but doesn't block main flow | Configurable retry count |

---

## System Layers Overview

| Layer | Name | Domain | Phase | Core Responsibility |
|-------|------|--------|-------|-------------------|
| **R0** | API Relay | Ledger & Trading | 1 | HL WebSocket/REST proxy; format conversion; transparent migration |
| **L1** | Account & Assets | Ledger & Trading | 1 | User registration; wallet management; multi-chain deposit/withdrawal |
| **L2** | Order Routing | Ledger & Trading | 2 | Three-mode routing decision; pre-order approval (queries Risk) |
| **L3** | Internal Execution | Ledger & Trading | 3 | Internal position management; internalization queue; close execution |
| **L4** | HL Proxy Execution | Ledger & Trading | 2 | HL API wrapper; fill optimization; position mapping |
| **L5** | Margin & Liquidation | Ledger & Trading | 3 | Isolated/cross margin calculation; unified liquidation engine (triggered by Risk) |
| **L6** | Risk & Hedging | Risk Control | 2 | Net exposure monitoring; hedge threshold decision; risk reserve management |
| **L7** | Market Data | Ledger & Trading | 1 | Real-time HL prices, funding rates, order book |
| **L8** | Settlement & Reconciliation | Ledger & Trading | 3 | PnL calculation; funding rate settlement; cross-system reconciliation |
| **L9a** | Admin (Trading) | Ledger & Trading | 2 | User queries; order history; account management |
| **L9b** | Admin (Risk) | Risk Control | 2 | Routing mode switching; hedge account monitoring; risk reserve config |
| **Infra** | Infrastructure | Both | 1-3 | Database; message queue; monitoring alerts; deployment framework |

---

## Implementation Highlights for Developers

### 1. Message Queue is the Only Inter-Domain Communication Channel

- Use Redis Streams or Kafka
- Ledger domain publishes events to `topic:ledger.events`
- Risk domain consumes that topic, publishes commands to `topic:risk.commands`
- **Forbidden**: RPC calls, shared memory, direct database modifications

### 2. Risk Domain is "Observer + Circuit Breaker", Not "Executor"

- Risk domain does **NOT** execute user orders, liquidations, or hedges
- Risk domain **DECIDES** whether to liquidate, how to hedge, and issues commands
- Ledger domain **EXECUTES** these commands and returns results

### 3. Complete Liquidation Orchestration

```
1. Risk Domain: Detect margin ratio < liquidation threshold
2. Risk Domain: Send LIQUIDATION_COMMAND to L3/L5
3. L3/L5: Execute close order
4. L3/L5: Send LIQUIDATION_EXECUTED back to Risk
5. Risk Domain: Receive confirmation, mark liquidation complete
6. L8: Calculate final PnL, write settlement record
```

**Key**: If L3 fails → L5 auto-routes to HL close; if both fail → alert + manual backstop

### 4. HL Dual Account Uses Different Addresses

- **Trading Account**: `0x...HL_TRADING_...`, dedicated to large user orders
- **Hedge Account**: `0x...HL_HEDGE_...`, dedicated to hedging

Monitor both independently, never mix. Hardcode both addresses in system config.

### 5. All Cross-Domain Calls Must Carry Idempotency Key

- Ledger sends: `request_id` or `event_id`
- Risk sends: `command_id` or `hedge_job_id`
- Receiver caches historical messages within 24 hours, returns cached result for duplicates

### 6. Hedge Instructions Execute Async, Don't Block Order Flow

- User submits order → immediately return 200 OK
- Exposure changes → async notify Risk domain
- Risk domain determines hedge needed → send hedge instruction to L4
- L4 async executes hedge, users can continue ordering

### 7. Canary Phase 2: Full HL Routing, Risk Domain in Observation Mode

MVP launch strategy during Phase 2:
- L2 Order Routing: All orders (regardless of size) route to HL (i.e., HL_MODE)
- L3 Internal Execution: Code complete but **disabled**, no orders accepted
- L6 Hedge Engine: **Observation mode**, calculate hedges but don't dispatch
- Risk Reserve: Still accumulating, but not used for hedging

**Purpose**: Verify system reliability via full HL routing while warming up L3 launch. Once stable for 3 weeks + all alerts cleared, enter Phase 3 and activate L3 internal execution.

### 8. Each Domain Independently Deployed and Scaled

- Ledger & Trading Domain: Pod count scales with "order QPS"
- Risk Control Domain: Pod count scales with "exposure change frequency"

Separate deployment, fault isolation. Trading domain failure doesn't affect Risk domain monitoring and alerting.

---

## Performance Targets

| Operation | P99 Latency | Reliability |
|-----------|-----------|------------|
| Routing decision | <5ms | 99.99% |
| HL order forwarding | <50ms | 99.95% |
| Internal internalization execution | <10ms | 99.99% |
| Liquidation detection | <1s | 99.99% |
| API response | <100ms | 99.9% |
| Exposure snapshot consistency | <5 minutes | 100% |
