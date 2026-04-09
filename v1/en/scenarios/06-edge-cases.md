# Edge Cases & Extreme Conditions

**Document Version**: v1.0
**Last Updated**: 2026-04-09
**Author**: Claude Code
**Status**: Specification Phase

---

## Scenario Overview

This document details platform behavior under extreme conditions, failure states, and abnormal inputs. Covers network disconnect, HL account risk, extreme market volatility, cascade liquidation, deposit confirmation delay, withdrawal rush, funding rate deviation, admin errors, concurrent race conditions, API rate limiting, multi-account mixing, and system restart recovery—12 key edge cases total.

These scenarios ensure platform maintains data consistency, risk control, and recoverability under non-ideal conditions.

---

## SC-EC-001: HL WebSocket Disconnect Exceeds 1 Minute

**Scenario Background**:
Platform receives real-time HL market data via WebSocket (prices, funding rates, order books). WS connection suddenly breaks (network fault, HL service glitch), stays disconnected >1 min. Platform must detect disconnect, pause data-dependent operations, attempt reconnect, then full state sync to recover consistency.

**Input**:
- WS connection: CONNECTED → DISCONNECTED (T=0ms)
- Disconnect duration: 60s+ (continuous growth)
- Reconnect strategy: Exponential backoff (100ms, 200ms, 500ms, 1s, 2s, ...)
- Platform state:
  - INTERNAL net exposure: $350K (cached)
  - HL hedge account BTC long: $280K (cached)
  - Pending limit orders: 3 orders, cache status not updated

**Decision Rule**:
```
def monitor_ws_connection():
    while True:
        if (ws.is_connected()):
            last_msg_time = current_time()
        else:
            disconnect_duration = current_time() - last_msg_time

            if (disconnect_duration > 10s):
                log_alert("WS disconnect", severity=P2)

            if (disconnect_duration > 60s):
                log_alert("WS prolonged disconnect", severity=P1)
                handle_long_disconnect()

def handle_long_disconnect():
    # Step 1: Pause risky operations
    halt_new_orders()  # Reject new orders (error)
    pause_liquidation_checks()  # Pause checks (lower frequency)
    halt_hedge_orders()  # Don't send hedge directives

    # Step 2: Reconnect attempt (exponential backoff)
    for attempt in range(max_retries=10):
        wait_time = min(100ms * (2 ^ attempt), 30s)
        if (ws.reconnect()):
            log_info(f"WS reconnected, elapsed {wait_time}")
            break
        sleep(wait_time)
    else:
        log_alert("WS reconnect failed", severity=P0)
        trigger_fallback_mode()
        return

    # Step 3: Full state sync
    sync_all_state():
        # Fetch latest snapshots
        hl_account_state = fetch_account_snapshot()
        hl_positions = fetch_positions()
        hl_orders = fetch_open_orders()
        market_prices = fetch_latest_prices()

        # Reconcile local state
        reconcile_positions(local_cache, hl_account_state)
        reconcile_orders(local_cache, hl_orders)

        # Resume normal operations
        resume_operations()

def trigger_fallback_mode():
    # WS unavailable → Degrade to REST polling
    log_alert("WS unavailable, enable REST polling mode", severity=P0)
    ws_available = False
    poll_interval = 5s  # Poll every 5s
    poll_endpoints = [
        "/account",
        "/user/positions",
        "/user/orders"
    ]
```

**Output**:
- T=0ms: WS disconnect event logged
- T=10s: P2 alert: "WS disconnect 10s, monitoring"
- T=60s: P1 alert: "WS disconnect 60s, emergency handling started"
- T=60s–120s: Reconnect attempt sequence (4–5 exponential backoff tries)
- T=120s: WS recovers → Full sync initiated
- T=125s: Sync complete → Data consistency verified
- T=130s: Resume new order acceptance + liquidation checks + hedge directives
- System status marked: `ws_status: CONNECTED, last_sync: 2026-04-09 14:32:15Z, sync_gap: 2m15s`

**Monitoring Points**:
- WS connection status real-time (CONNECTED / DISCONNECTED)
- Disconnect duration accumulated
- Reconnect attempt count and last timestamp
- Full sync latency (target <10s)
- Data completeness check (positions, orders, prices)
- Fallback mode activation (REST polling)

**Fallback on Failure**:
- Reconnect fails 5+ times, still unavailable → Enter MAINTENANCE mode
  ```
  System state: MAINTENANCE
  User API: Return 503 Service Unavailable
  New orders: Reject (HTTP 503)
  Existing positions: Only allow close (sell market orders)
  Liquidation: Pause auto, manual review
  Hedge: Pause directives
  Notification: P0 alert + manual intervention begins
  ```

- Full sync fails (data inconsistent) → Validation failure → Stay MAINTENANCE → Manual repair

- Worst case (WS + REST both unavailable) → System read-only, trading prohibited

---

## SC-EC-002: HL Trading Account Margin Ratio Drops Below 150%

**Scenario Background**:
Platform executes user large orders on HL trading account, holds user-proxy positions. Initial margin ratio 500%, suddenly extreme market volatility, floating losses expand, account equity drops, margin ratio falls to 150% (approaching HL liquidation line 200%). System must immediately take emergency action to prevent HL force liquidation.

**Input**:
- HL trading account:
  - Account equity: $300K (was $500K, floating loss $200K)
  - Occupied maintenance margin: $200K
  - Margin ratio: 300K / 200K = 150% (danger level!)
  - Platform policy floor: 500% (target)
  - HL force liquidation: 200% (absolute bottom)

- Position mix:
  - BTC long $1M (floating loss $150K)
  - ETH short $500K (floating loss $50K)
  - Total floating loss $200K

- Capital pool available: $500K
- Emergency top-up need: $200K (restore to $400K equity, margin ratio 400%)

**Decision Rule**:
```
def monitor_trading_account_margin():
    margin_ratio = account_equity / maintenance_margin_required

    if (margin_ratio < 200%):
        # Absolute emergency: Liquidation imminent
        severity = CRITICAL
        trigger_emergency_liquidation_prevention()

    elif (margin_ratio < 300%):
        # High risk: Over threshold
        severity = P0_ALERT
        take_immediate_action()

    elif (margin_ratio < 500%):
        # Moderate risk: Monitoring
        severity = P1_ALERT
        notify_risk_manager()

def take_immediate_action():
    # Step 1: Halt all new HL orders
    halt_hl_new_orders()

    # Step 2: Emergency capital transfer (on-chain)
    emergency_fund_transfer(amount=$200K, source=fund_pool, dest=hl_trading_account)
    # Note: On-chain needs 10–30 min, account at risk during wait

    # Step 3: Consider forced partial liquidation (conservative)
    # If capital not arriving timely, force close some positions
    if (margin_ratio < 250%):
        consider_partial_liquidation(amount=$100K, priority="largest_floating_loss")

    # Step 4: Notify Risk Manager + CTO
    notify_critical_incident()

# Timeline in this scenario:
T=0:   margin_ratio = 150% < 300% → P0 alert
       → halt_hl_new_orders()
       → emergency_fund_transfer($200K)
       Notify: "HL trading account margin at 150%, halted orders, emergency top-up in progress"

T+1m:  margin_ratio still 150% (capital on-chain processing)
       → Assess partial close
       → Evaluate: Close BTC long $50K to release ~$20K margin
       → Await capital or execute close (Risk Manager decides)

T+15m: Capital appears on-chain → Account equity restores to $300K + $200K = $500K
       margin_ratio = $500K / $200K = 250% (improving, still monitor)

T+30m: Capital final settlement → Account equity $500K+
       margin_ratio = 500%+ → Safe restored
       → Resume HL order acceptance
       → Continue monitoring
```

**Output**:
- P0 alert: "HL trading account margin at 150%, near liquidation threshold"
- Actions:
  - Halt all new HL orders (immediate)
  - New user large orders rejected, error: `{code: SERVICE_UNAVAILABLE, msg: "HL account margin insufficient"}`

- Emergency top-up request: $200K → HL trading account
  ```
  EMERGENCY_FUND_REQUEST {
    source: platform_fund_pool
    destination: hl_trading_account
    amount: $200K
    priority: CRITICAL
    reason: margin_ratio_critical
    expected_arrival: 30min
  }
  ```

- User notification (if applicable): Subsequent large orders return 503 (HL account under maintenance)

- Internal control panel:
  - HL trading account margin ratio display: 150% (red warning)
  - Capital top-up tracking (pending→on-chain→arrival→confirmed)
  - Recommended actions list (await top-up / forced close / emergency override)

**Monitoring Points**:
- HL trading account margin ratio (real-time, 1s refresh)
- Account equity curve
- Position floating loss distribution
- Capital top-up process status (on-chain confirms)
- Order rejection rate during emergency

**Fallback on Failure**:
- Top-up delayed >15 min, margin ratio continues falling <200%
  → Execute forced partial close:
    - Close largest floating loss (BTC long $50K)
    - Expect to free $20K margin → Ratio may recover to 180%–200%
    - Continue awaiting top-up or further close

- Close itself constrained (HL slippage huge, low liquidity)
  → Use limit orders to close, may not execute fast
  → Last resort: Accept HL liquidation (cover margin loss from reserve)

- Top-up source exhausted (capital pool empty)
  → Trigger emergency financing (partner) or accept liquidation

---

## SC-EC-003: BTC Flashes Down 15% in 1 Minute

**Scenario Background**:
Extreme market event (flash crash, cascade liquidation, black swan), BTC price collapses 15% in 1 minute. Exceeds "volatility >5%/hour" force-HL-route rule, also massive hedge position floating loss risk. Platform must switch to defense mode, pause INTERNAL betting, closely monitor hedge account, high-frequency liquidation checks.

**Input**:
- BTC price change: $60,000 → $51,000 (1 min, -15%)
- Volatility: 15% / 1min = 15%/min = 900%/hour (far >5%)
- Platform current state:
  - Mode: NORMAL_MODE
  - INTERNAL BTC net exposure: $1.2M (long)
  - HL hedge BTC long: $600K (3x leverage, occupies $200K)
  - Hedge account equity: $300K

- Volatility detection: Real-time 1-min price % change monitoring

**Decision Rule**:
```
def monitor_market_volatility():
    price_1m_ago = get_price_at(now - 1min)
    price_now = get_price_now()
    volatility_1min = abs((price_now - price_1m_ago) / price_1m_ago)

    # Convert to hourly (approx)
    volatility_per_hour = volatility_1min * 60

    if (volatility_per_hour > 5%):
        log_alert("Market volatility excessive", severity=P1)
        force_route_to_hl()
        halt_internal_trading()
        trigger_liquidation_check()
        monitor_hedge_margin()

def force_route_to_hl():
    # Pause INTERNAL bet, all orders→HL
    routing_mode = HL_MODE
    log_info("Force switch to HL_MODE: volatility excessive")

def trigger_liquidation_check():
    # High-frequency liquidation check (lower latency)
    check_interval = 1s  # vs normal 5s
    liquidation_check_frequency = HIGH

def monitor_hedge_margin():
    # Hedge position floating loss risk
    hl_hedge_account.margin_ratio = (account_equity) / (maintenance_margin)

    # Hedge floating loss calculation
    hedge_notional = $600K
    price_loss = 15% * $600K = $90K (floating loss)
    account_equity = $300K - $90K = $210K
    maintenance_margin ≈ $600K / 5x = $120K (conservative)
    margin_ratio = $210K / $120K = 175% (danger!)

    if (margin_ratio < 200%):
        alert("Hedge account margin danger", severity=P0)
        consider_reduce_hedge_position()

    if (margin_ratio < 150%):
        alert("Hedge account near liquidation", severity=CRITICAL)
        emergency_reduce_hedge()
```

**Output**:
- Volatility detected: 15% / 1min triggers >5%/hour rule
- Immediate actions:
  1. Switch routing: NORMAL_MODE → HL_MODE
  2. Halt INTERNAL betting: Reject new or switch to HL
  3. Upgrade liquidation check: High frequency (1s scan)
  4. Cascade liquidation event: Many users may trigger → L5 liquidation engine activates

- Hedge account alerts:
  ```
  MARGIN_ALERT {
    account: hl_hedge
    current_ratio: 175%
    threshold: 200%
    status: CRITICAL
    action: Consider reduce hedge
  }
  ```

- User notification (API error):
  ```
  New order request returns:
  {
    code: MARKET_VOLATILITY_HALT,
    message: "Market volatility excessive, betting paused. Order routed to Hyperliquid.",
    order_routed_to: "HYPERLIQUID"
  }
  ```

- Internal notifications:
  - P0 alert to Risk Manager + CTO
  - Hedge account real-time monitoring highlighted
  - Liquidation event stream live log

**Monitoring Points**:
- Real-time volatility (1-min, 5-min, 1-hour)
- BTC price K-line trend
- Hedge account floating loss and margin ratio
- Liquidation trigger count and user count
- INTERNAL→HL order flow distribution
- System latency during extreme volatility

**Fallback on Failure**:
- Hedge floating loss continues (BTC drops another 5%) → Triggers SC-EC-002 (margin <150% flow)
  → Emergency top-up + partial de-hedge

- Liquidation check latency exceeds (>5s) → Some users' liquidation delayed
  → Post-trade reconciliation, makeup liquidation

- HL market depth insufficient, orders can't execute → User orders rejected → Poor experience
  → May need retry incentives or manual handling

- Extreme crash (30%+) → System may unable to respond → Temp trading halt, maintenance mode

---

## SC-EC-004: User Cascade Liquidation (Cross-Margin)

**Scenario Background**:
User in cross-margin (Cross-Margin) mode holds 5 positions (BTC, ETH, SOL, etc.). Market drops, account equity falls, triggers liquidation line (Account Equity ≤ Total Maintenance Margin). Liquidation engine closes positions in order, post-close recalculates equity, may continue triggering liquidation—cascade effect. System must sequence closes correctly, reconcile accurately.

**Input**:
- User account mode: Cross-Margin
- Account total equity: $50K
- Total maintenance margin required: $40K
- Initial margin ratio: 125% (= $50K / $40K)
- Position list (by floating loss order):
  ```
  Pos1: BTC 1x        Initial $10K → Floating loss -$8K → Equity $2K
  Pos2: ETH 3x        Initial $5K  → Floating loss -$4K → Equity $1K
  Pos3: SOL 2x        Initial $8K  → Floating loss -$3K → Equity $5K
  Pos4: DOGE 1x short Initial $2K  → Profit +$1K → Equity $3K
  Pos5: LINK 1x       Initial $25K → Floating loss -$2K → Equity $23K
  Total equity: $2K + $1K + $5K + $3K + $23K = $34K
  ```

- BTC drops 20% more → Pos1, Pos5 lose further → Total equity drops to $20K < $40K maintenance → Liquidation triggers

**Decision Rule**:
```
def cascade_liquidation(user_account):
    while True:
        account_equity = sum(position.equity for position in account)
        total_maintenance = sum(position.maintenance_margin for position in account)

        if (account_equity <= total_maintenance):
            # Liquidation triggered
            log_alert("Liquidation triggered", user=user_account.id)

            # Strategy: Close from largest loss first
            positions_by_loss = sort_by_floating_loss(account.positions, desc=True)

            for position in positions_by_loss:
                if (account_equity > total_maintenance):
                    # Restored to safe → Stop
                    break

                # Close position (market order)
                execute_market_liquidation(position)

                # Recalculate
                account_equity = recalculate_equity()
                total_maintenance = recalculate_total_maintenance()

                # Log each step
                log_cascade_step(position.id, account_equity, total_maintenance)
        else:
            break  # No liquidation needed

# Timeline:
T=0:   Account equity $20K < Maintenance $40K → Liquidation starts

       Close order (by loss):
       1. Pos1 (BTC long) floating loss -$8K (largest)

T+1s:  Close Pos1 (BTC long 1x) → Free $10K capital
       Account equity = $20K + $10K (capital) - slippage = $28K
       Total maintenance = $30K (Pos1 removed) → Still need close

T+2s:  Close next: Pos2 (ETH long 3x) floating loss -$4K
       Close Pos2 → Free $5K capital
       Account equity ≈ $28K + $5K - slippage = $32K
       Total maintenance = $24K (Pos1, Pos2 removed) → Equity > Maintenance ✓

T+3s:  Liquidation stops (restored to safe)
       Final: Pos3, Pos4, Pos5 retained, Pos1/Pos2 closed
       Final equity ~$32K, margin ratio 32K / 24K = 133%
```

**Output**:
- Liquidation event sequence:
  ```
  LIQUIDATION_CASCADE {
    user_id: user_ABC
    trigger_time: 2026-04-09 14:32:15Z
    initial_equity: $20K
    initial_maintenance: $40K

    cascade_steps: [
      {
        step: 1
        closed_position: {id: Pos1, symbol: BTC, side: long, notional: $10K}
        execution: market_order, price: $48,000 (slippage -1%)
        capital_released: $10K
        post_close_equity: $28K
        post_close_maintenance: $30K
        ratio_after: 93% (unsafe)
      },
      {
        step: 2
        closed_position: {id: Pos2, symbol: ETH, side: long, notional: $5K}
        execution: market_order, price: $2,400 (slippage -1.5%)
        capital_released: $5K
        post_close_equity: $32K
        post_close_maintenance: $24K
        ratio_after: 133% (safe ✓)
      }
    ]

    final_equity: $32K
    final_maintenance: $24K
    final_margin_ratio: 133%
    liquidation_complete: true
  }
  ```

- User notification:
  ```
  Liquidation started: Account margin insufficient, positions force-closed.
  Closed: BTC long 1x ($10K), ETH long 3x ($5K)
  Current margin ratio: 133% (restored to safe)
  Remaining: SOL long, DOGE short, LINK long
  ```

- Internal liquidation log: Every step, execution price, slippage, intermediate equity

**Monitoring Points**:
- Cascade steps and total duration (target <10s complete)
- Post-close equity and margin ratio per step
- Close execution price vs real-time (slippage)
- Liquidated user count and position count
- INTERNAL vs HL position liquidation mix

**Fallback on Failure**:
- Certain step close fails (order rejected) → Retry
  → If continuous failure, degrade to limit order (may not execute timely)
  → Last resort: Manual override, Risk Manager direct close

- Mid-liquidation price fetch delay → Use cached/last-known price
  → Post-trade reconciliation

- INTERNAL and HL positions mix liquidate → Separate paths
  - INTERNAL: Internal settlement (L5)
  - HL: Send market close to HL (L4)
  → Timing may differ, post-trade reconcile

- Cascade unable to fully restore safe state → Continue liquidating until safe

---

## SC-EC-005: Deposit Confirmation Delay Causes Balance Dispute

**Scenario Background**:
User deposits $10K USDT on TRON chain to platform wallet. Transaction sent. Platform monitors TRON for confirmations, requires 19 blocks. Mid-confirmation, network glitch interrupts, reconnects, confirmation stalls. System must ensure no duplicate credit, use idempotency, achieve eventual consistency.

**Input**:
- Deposit info:
  - User ID: user_XYZ
  - Chain: TRON
  - Asset: USDT (TRC20)
  - Amount: $10K
  - TxHash: 0xabc123...
  - Initiated: 2026-04-09 14:00:00Z

- Target confirms: 19 blocks (≈57s, TRON 3s/block)
- Expected complete: 14:01:00Z

- Listen progress (interrupted mid-way):
  ```
  14:00:05 - Block 50000 confirmed (1/19)
  14:00:08 - Block 50001 confirmed (2/19)
  ...
  14:00:35 - Block 50011 confirmed (12/19)
  14:00:36 - [Network interrupt] → Listen pauses
  14:00:56 - [Reconnect] → Continue from Block 50012
  14:01:00 - Block 50018 confirmed (19/19) ✓ Complete
  ```

**Decision Rule**:
```
def monitor_deposit_confirmation(tx_hash, chain, required_confirmations=19):
    deposit_record = db.get_deposit_by_txhash(tx_hash)
    confirmation_count = deposit_record.confirmation_count or 0
    last_check_block = deposit_record.last_confirmed_block or 0

    while (confirmation_count < required_confirmations):
        try:
            current_block = fetch_current_block(chain)
            target_block = last_check_block + 1

            # Check if target block exists
            if (verify_block_exists(chain, target_block)):
                confirmation_count += 1
                db.update_deposit(tx_hash, {
                    confirmation_count: confirmation_count,
                    last_confirmed_block: target_block,
                    last_check_time: now()
                })
                last_check_block = target_block

            # Check timeout (should complete ~60s)
            if (now() - deposit_record.created_time > 300s):
                # 1. Verify tx really on chain
                tx_on_chain = verify_tx_on_chain(chain, tx_hash)
                if (not tx_on_chain):
                    # Tx failed or replaced → Notify user
                    mark_deposit_failed(tx_hash, reason="tx_not_on_chain")
                    break

                # 2. Tx on chain but slow confirm → Alert, continue wait
                alert(f"Deposit slow confirm: {tx_hash}, {confirmation_count}/19", severity=P2)

        except ConnectionError:
            # Network glitch → Record progress → Retry after backoff
            log_info(f"Listen interrupted, {confirmation_count}/19 confirmed, last block: {last_check_block}")
            sleep(5s)  # Backoff retry
            continue

        sleep(0.5s)  # Poll interval

    # Confirm complete
    db.update_deposit(tx_hash, {
        status: CONFIRMED,
        confirmation_time: now(),
        user_balance: add_balance(user_id, amount)
    })
    notify_user(user_id, f"Deposit ${amount} arrived")

# Key: Idempotency
# All DB updates use tx_hash as unique key
# Even if listener rechecks same block, won't double-credit
```

**Output**:
- Deposit status flow:
  ```
  PENDING (14:00:00)
  → CONFIRMING [2/19] (14:00:08)
  → CONFIRMING [12/19] (14:00:35)
  → [NETWORK_ERROR] (14:00:36)
  → CONFIRMING [12/19] (14:00:56, reconnect resume)
  → CONFIRMING [19/19] (14:01:00)
  → CONFIRMED (14:01:00)
  ```

- User balance update:
  ```
  Balance Before: $50,000
  + Deposit: $10,000
  = Balance After: $60,000 (14:01:05)
  ```

- Internal audit log:
  ```
  DEPOSIT_LOG {
    tx_hash: 0xabc123
    chain: TRON
    amount: $10K
    created: 2026-04-09 14:00:00Z
    confirmed: 2026-04-09 14:01:05Z
    confirm_time: 65s
    network_interrupts: 1
    final_confirmation_count: 19
    status: SUCCESS
  }
  ```

**Monitoring Points**:
- Unconfirmed deposit queue (by timestamp)
- Each deposit progress (X/19)
- Average confirm time
- Network interrupt events and recovery
- Timed-out deposits (>5 min unconfirmed)

**Fallback on Failure**:
- Tx not on chain → Mark failed, ask user to verify send
- Tx replaced (RBF, accelerate) → Track new tx_hash → Re-listen
- Network interrupt >10 min → P1 alert + manual check
- Deposit amount mismatch → Chain audit + manual investigation

---

## SC-EC-006: Withdrawal Rush (Concurrent Mass Withdrawals)

**Scenario Background**:
Panic event (market fear, competitor promo, platform glitch rumor) triggers massive user withdrawal. 1 hour withdrawal requests total $2M, but platform hot wallet only $500K. System must triage, trigger emergency consolidation (cold→hot wallet), prevent withdrawal stoppage.

**Input**:
- Hot wallet balance: $500K
- Withdrawal request queue (1 hour):
  ```
  Request 1: user_A, $50K, tier_normal
  Request 2: user_B, $30K, tier_normal
  Request 3: user_C, $200K, tier_large
  Request 4: user_D, $150K, tier_large
  Request 5: user_E, $15K, tier_normal
  Request 6: user_F, $300K, tier_xtra_large
  Request 7: user_G, $25K, tier_normal
  Request 8: user_H, $1.2M, tier_xtra_large ← Exceeds entire hot wallet
  Total need: $1.97M
  ```

- Cold wallet balance (consolidation source): $5M
- Hot→cold delay (on-chain): 15–30 min

**Decision Rule**:
```
def handle_withdrawal_surge():
    total_requested = sum(all_pending_requests)
    available = hot_wallet_balance()

    if (total_requested > available):
        trigger_triage()

def trigger_triage():
    # Tier assignments
    for request in withdrawal_queue:
        if (request.amount < $10K):
            tier = "NORMAL"
            priority = HIGH
            max_wait = 5min
        elif (request.amount < $100K):
            tier = "LARGE"
            priority = MEDIUM
            max_wait = 30min
        else:
            tier = "XTRA_LARGE"
            priority = LOW
            max_wait = 2hours

    # Step 1: Process by priority with available funds
    normal_requests = [r for r in queue if r.tier == NORMAL]
    large_requests = [r for r in queue if r.tier == LARGE]
    xtra_large_requests = [r for r in queue if r.tier == XTRA_LARGE]

    remaining = hot_wallet_balance()

    # NORMAL priority first
    for req in normal_requests[:]:
        if (remaining >= req.amount):
            execute_withdrawal(req)
            remaining -= req.amount
            normal_requests.remove(req)
        else:
            break

    # LARGE next
    for req in large_requests[:]:
        if (remaining >= req.amount):
            execute_withdrawal(req)
            remaining -= req.amount
            large_requests.remove(req)
        else:
            break

    # Step 2: Trigger emergency consolidation
    shortage = total_requested - hot_wallet_balance()
    if (shortage > 0):
        trigger_emergency_consolidation(shortage)
        # Expected arrival: 15–30 min

    # Step 3: Wait for consolidation, continue with remaining
    # As hot wallet receives new funds, process LARGE/XTRA_LARGE
```

Timeline example:
```
14:00:00 - Withdrawal rush detected, triage starts ($1.97M > $500K available)
           Tiers: NORMAL $120K, LARGE $350K, XTRA_LARGE $1.5M

14:00:15 - Batch 1: All NORMAL approved ($120K)
           Hot wallet: $380K remaining

14:00:30 - Batch 2: LARGE partial approved ($350K)
           Hot wallet: $30K (depleted)

14:00:35 - Emergency consolidation triggered: Cold→Hot $1.5M+
           Expected arrival: 14:15–14:30

14:15:00 - Hot wallet receives partial funds ($800K)
           Continue: LARGE remainder + XTRA_LARGE partial

14:30:00 - Hot wallet receives full ($1.5M+)
           All XTRA_LARGE completed
```

**Output**:
- Triage result and status:
  ```
  TRIAGE_RESULT {
    total_requested: $1.97M
    available: $500K
    shortage: $1.47M

    tiers: {
      NORMAL: {
        count: 4,
        total: $120K,
        status: APPROVED,
        expected_complete: 14:01:00Z
      },
      LARGE: {
        count: 2,
        total: $350K,
        status: PARTIAL_APPROVED,
        approved: $350K,
        expected_complete: 14:05:00Z
      },
      XTRA_LARGE: {
        count: 2,
        total: $1.5M,
        status: QUEUED,
        expected_start: 14:20:00Z (post-consolidation)
      }
    }
  }
  ```

- Emergency consolidation directive:
  ```
  EMERGENCY_CONSOLIDATION {
    from: cold_wallet
    to: hot_wallet
    amount: $1.5M
    chains: TRON, ETH (multi-chain)
    initiated: 14:00:35Z
    expected_arrival: 14:15–14:30Z
    status: IN_PROGRESS
  }
  ```

- User notifications (tiered):
  ```
  [NORMAL] Your withdrawal approved, ~5 min arrival
  [LARGE] Withdrawal queued, ~30 min arrival
  [XTRA_LARGE] Your large withdrawal processing. Platform refilling liquidity.
                Expected ~2 hours. Thank you for your patience.
  ```

**Monitoring Points**:
- Withdrawal queue length and total
- Tiers pending and completed
- Hot wallet balance and depletion rate
- Consolidation progress (on-chain confirm)
- User complaints/disputes

**Fallback on Failure**:
- Consolidation >30 min delay → Alternative liquidity (other channels)
- Cold wallet insufficient → External financing or pause new (honor existing)
- Hot wallet empty, consolidation pending → Pause new, prioritize existing
- User escalation handling → Compensate (if reasonable)

---

## SC-EC-007: Funding Rate Deviation at Settlement Time

**Scenario Background**:
HL funding rate settles every 8 hours. Platform caches HL's rate but latency causes platform cache (0.010%) to differ from actual HL rate (0.012%), delta 0.002%. At settlement, must reconcile and handle deviation.

**Input**:
- HL actual funding: 0.012% / 8h
- Platform cache: 0.010% / 8h
- Deviation: 0.002% (cache too low)

- INTERNAL position total: $10M BTC long
- Settlement: 2026-04-09 16:00:00 UTC (8h cycle)

- Fee math:
  - HL actual due: $10M * 0.012% = $1,200
  - Platform calculated (cache): $10M * 0.010% = $1,000
  - Gap: $200 (platform short)

**Decision Rule**:
```
def settle_funding_rate():
    # Step 1: Settle using cache (normal)
    for position in internal_positions:
        cached_rate = get_cached_funding_rate(position.symbol)
        fee_collected = position.notional * cached_rate
        settlement_record = record_settlement(position, cached_rate, fee_collected)

    # Step 2: Post-settlement reconcile
    actual_rate = fetch_latest_funding_rate_from_hl(symbol)

    if (abs(actual_rate - cached_rate) > tolerance):  # tolerance = 0.001%
        deviation = actual_rate - cached_rate

        # Calculate impact
        total_notional = sum(pos.notional for pos in internal_positions)
        deviation_amount = total_notional * deviation

        log_warning(f"Funding rate deviation: {deviation}, amount: ${deviation_amount}")

        # Handle
        if (deviation_amount > 0):  # HL higher, platform under-charged
            # Platform eats the difference
            log_action(f"Platform absorbs gap ${deviation_amount}")
            transfer_from_reserve(deviation_amount)
            log_deviation(symbol, cached_rate, actual_rate, deviation_amount, "platform_absorb")

        elif (deviation_amount < 0):  # HL lower, platform over-charged
            # Refund users
            refund_amount = -deviation_amount
            distribute_refund_to_users(refund_amount)
            log_deviation(symbol, cached_rate, actual_rate, deviation_amount, "refund_users")

        # Update cache
        update_funding_rate_cache(symbol, actual_rate)

        # Alert
        alert(f"Funding rate deviation {deviation}, handled", severity=P2)
```

**Output**:
- Settlement record (per position):
  ```
  FUNDING_SETTLEMENT {
    period: 2026-04-09 08:00–16:00 UTC
    user: user_XYZ
    symbol: BTC
    notional: $100K
    side: long
    rate_applied: 0.010% (cached)
    fee_charged: $10
    status: SETTLED
  }
  ```

- Post-settlement deviation detected:
  ```
  FUNDING_RATE_DEVIATION_LOG {
    symbol: BTC
    cached_rate: 0.010%
    actual_rate: 0.012%
    deviation: +0.002%

    affected_notional: $10M
    deviation_amount: $200

    action: PLATFORM_ABSORB
    cost: $200 (from reserve)

    timestamp: 2026-04-09 16:05:00Z
    severity: P2
  }
  ```

- Reserve adjustment:
  - Before: $450K
  - Absorb: -$200
  - After: $449,800

- Cache update: BTC 0.010% → 0.012%

**Monitoring Points**:
- Cache vs actual deviation (every sync)
- Deviation >threshold events
- Platform absorbed cost cumulative
- Rate update latency

**Fallback on Failure**:
- Deviation discovered late (multiple cycles) → Multi-cycle reconcile → Batch adjustment
- Huge deviation (>$1K) → P1 review + manual verification
- Reserve insufficient → Top-up from other source or delay adjustment
- HL rate data unavailable → Use cached, reconcile when recovered

---

## SC-EC-008: Admin Misconfiguration (Threshold Set to $0)

**Scenario Background**:
Risk Manager adjusts routing threshold via admin UI. Should change $10K→$15K but fat-fingers $0 (zero). All orders, even $1, route to INTERNAL, exposure surges. System must detect via audit log, validate threshold reasonableness, alert, auto-correct.

**Input**:
- Original config: routing_threshold = $10K
- Error config: routing_threshold = $0
- Admin: risk_manager_01
- Time: 2026-04-09 14:30:00 UTC
- Intent: Should be $15K

- Post-change orders:
  ```
  14:30:30 Order 1: $2K → Check: $2K <= $0? (false) → Undefined behavior
  Should route HL, but threshold=0 causes issue
  ```

**Decision Rule**:
```
def apply_routing_threshold_change(new_threshold):
    # Step 1: Validate
    if (new_threshold < 0):
        reject_change("Threshold cannot be negative")
        return

    if (new_threshold == 0 or new_threshold < $100):  # Min-bound check
        log_warning(f"Anomalous threshold: {new_threshold}, below min $100")
        alert("Threshold anomalously low", severity=P1)
        # Can reject or warn

    # Step 2: Audit log
    audit_log({
        change_id: uuid(),
        timestamp: now(),
        changed_by: current_user,
        old_value: routing_threshold,
        new_value: new_threshold,
        status: PENDING_APPROVAL  # Sensitive changes need approval
    })

    # Step 3: Apply (only low-volume periods)
    current_hour = now().hour
    if (current_hour >= 2 and current_hour <= 8):  # Low-volume window
        apply_threshold(new_threshold)
        log_info(f"Threshold updated: {new_threshold}")
    else:
        queue_threshold_change(new_threshold, apply_at=next_low_volume_hour)
        alert("Sensitive change queued for low-volume window")

def monitor_threshold_anomaly():
    # Post-change, watch exposure
    if (routing_threshold == $0):
        # All orders→INTERNAL (or undefined)
        # Exposure rapidly grows
        net_exposure = get_net_exposure()
        if (net_exposure > $400K):  # Expect $0 threshold triggers fast growth
            alert("Threshold anomaly $0, exposure surge ${net_exposure}", severity=P0)

            # Auto-correct if change recent
            if (abs(now() - threshold_change_time) < 1hour):
                # Likely misconfig → auto-revert
                revert_threshold_to_previous(reason="anomaly_detected")
                alert("Auto-reverted anomalous threshold", severity=P1)
```

**Output**:
- Admin change audit log:
  ```
  AUDIT_LOG {
    event_id: audit_20260409_1430_001
    timestamp: 2026-04-09 14:30:00Z
    action: THRESHOLD_UPDATE
    actor: risk_manager_01
    old_value: $10,000
    new_value: $0  ← ANOMALY!
    status: APPLIED
  }
  ```

- Alert sequence:
  ```
  14:30:05 - P2 alert: Anomalous threshold $0 detected
  14:32:00 - P1 alert: Net exposure $350K (expect $100–200K)
  14:32:30 - P0 alert: Threshold $0 uncontrolled exposure, auto-revert
  ```

- Auto-recovery:
  ```
  AUTO_RECOVERY {
    anomaly: routing_threshold = $0
    trigger_time: 2026-04-09 14:32:30Z

    action: REVERT_THRESHOLD
    reverted_to: $10,000
    reason: anomaly_detected_and_exposure_surge

    status: SUCCESS
    new_net_exposure: $420K (stable)
  }
  ```

- Notifications:
  - Admin: "Your threshold change ($0) detected anomalous, auto-reverted to $10K. Please confirm."
  - CTO: P0 alert + audit link
  - Dashboard: Revert mark on change history

**Monitoring Points**:
- Config change audit log
- Post-change exposure growth rate
- Auto-revert trigger frequency
- Admin error rate

**Fallback on Failure**:
- Auto-revert fails → Manual revert by Risk Manager
- Missing threshold validation → Implement bounds immediately
- Exposure already surged → Post-event loss assessment + compensation

---

## SC-EC-009: Internalization vs. Hedge Directive Race Condition

**Scenario Background**:
L3 (INTERNAL) and L6 (Hedge) run in parallel. User $9K order executed L3 while L6 assesses exposure, about to send hedge. Timing mismatch: L6 may use stale exposure snapshot. System tolerates short-term gap, uses eventual consistency reconciliation.

**Input**:
- Current net exposure (L6 cache): $95K
- Hedge threshold: $100K
- L6 scan interval: 100ms
- L3 execution latency: <10ms

- Timeline:
  ```
  T=0ms:    L6 scan: Exposure $95K, < $100K → No hedge
  T=5ms:    L3 execute: User $9K order → Exposure becomes $104K
  T=10ms:   Exposure change event queued (async)
  T=100ms:  L6 next scan: Exposure updated to $104K > $100K → Trigger 50% hedge = $52K
  T=150ms:  Hedge order executed
  ```

- Race: If L6 doesn't update in T=5–100ms window, hedges only $47.5K (insufficient)

**Decision Rule**:
```
# L3 (sync)
def execute_internal_order(user_order):
    order.status = MATCHED
    user.position += order.notional
    net_exposure = recalculate_net_exposure()

    # Async event publish
    publish_event({
        type: INTERNAL_ORDER_MATCHED,
        exposure_changed: order.notional,
        new_exposure: net_exposure,
        timestamp: now()
    })

    return order.status

# L6 (periodic + event-driven)
last_cached_exposure = $95K
last_cache_time = T=0

def hedge_engine_loop():
    while True:
        # Periodic scan
        current_exposure = get_net_exposure()  # Direct DB read

        if (current_exposure != last_cached_exposure):
            log_debug(f"Exposure updated: {last_cached_exposure} → {current_exposure}")
            last_cached_exposure = current_exposure
            last_cache_time = now()

        # Hedge trigger
        if (current_exposure > HEDGE_THRESHOLD):
            execute_hedge(amount=current_exposure * hedge_ratio)

        sleep(100ms)

# Event-driven supplement
def on_exposure_changed_event(event):
    # If new exposure triggers hedge, execute immediately
    if (event.new_exposure > HEDGE_THRESHOLD):
        execute_hedge_immediately(event.new_exposure * hedge_ratio)
```

**Output**:
- Timeline (eventual consistency):
  ```
  T=0ms:    L3 execute $9K → Exposure $95K→$104K ✓
  T=50ms:   L6 scan (may use cache $95K) → No hedge
  T=100ms:  L6 scan → Read latest $104K → Trigger 50% hedge $52K
  T=150ms:  Hedge execute complete

  Final:
    - INTERNAL: $104K ✓ Correct
    - HL hedge: $52K ✓ Correct (50% of $104K)
    - Gap: 0 (L6 final read correct exposure)
  ```

- Monitoring:
  ```
  HEDGE_TIMING_LOG {
    order_id: order_abc
    internal_exec_time: 5ms
    exposure_change: +$9K
    exposure_before: $95K
    exposure_after: $104K

    hedge_trigger_latency: 100ms (L6 next scan)
    hedge_exec_time: 50ms
    total_order_to_hedge: 150ms

    expected_hedge: $52K
    actual_hedge: $52K
    discrepancy: 0% ✓
  }
  ```

**Monitoring Points**:
- Order-to-hedge latency
- Hedge coverage ratio accuracy
- Exposure gap/overshoot %
- High-frequency period distribution

**Fallback on Failure**:
- Insufficient hedge (L6 missed update) → Next scan detects, supplements → Max 100ms gap → Acceptable

- Over-hedge → Next exposure change triggers de-hedge → Auto-correct

- Extreme inconsistency (both failed) → Hourly full reconciliation + manual fix

---

## SC-EC-010: HL API Rate Limiting Delays Hedge

**Scenario Background**:
Platform sends hedge order to HL, receives 429 Too Many Requests. Order rejected, needs retry. Meanwhile INTERNAL exposure naked. Exponential backoff retry, monitor exposure risk rise.

**Input**:
- Hedge order: BTC long $500K (3x, $166.67K margin)
- HL rate limit: 100 req/min, response 429
- Retry: Exponential backoff

- Timeline:
  ```
  Attempt 1: T=0ms    (fail: 429)
  Attempt 2: T=100ms  (fail: 429)
  Attempt 3: T=300ms  (fail: 429)
  Attempt 4: T=700ms  (fail: 429)
  Attempt 5: T=1500ms (success: 200)
  ```

- Exposure naked: 1.5 seconds

**Decision Rule**:
```
def submit_hedge_order_with_retry(order):
    max_retries = 10
    base_wait = 100ms

    for attempt in range(max_retries):
        try:
            response = call_hl_api(order)
            if (response.status == 200):
                log_info(f"Hedge success, elapsed {attempt * base_wait}ms")
                return SUCCESS

            elif (response.status == 429):
                # Rate limit
                wait_time = base_wait * (2 ^ attempt)

                if (attempt < 5):  # Quick retries
                    log_warning(f"Rate limit, retry in {wait_time}ms")
                else:  # >5 retries, escalate
                    log_alert(f"Rate limit persist, hedge delayed {attempt * wait_time}ms", severity=P1)

                if (attempt == 5):  # >5s delay
                    # Exposure naked >5s, P0
                    publish_alert({
                        type: "HEDGE_DELAYED",
                        reason: "HL_RATE_LIMIT",
                        exposure: order.notional,
                        delay: wait_time,
                        severity: P0
                    })

                sleep(wait_time)
                continue

            else:
                # Other error
                log_error(f"HL API error {response.status}")
                return FAILURE

        except Exception as e:
            log_error(f"Network: {e}")
            sleep(base_wait * (2 ^ attempt))

    # 10 retries fail
    log_alert("Hedge retry 10x fail, switch HL_MODE", severity=P0)
    switch_routing_mode(HL_MODE)
    return FAILURE
```

**Output**:
- Retry log:
  ```
  HEDGE_RETRY_LOG {
    order_id: hedge_order_xyz
    initial_exposure: $500K

    attempts: [
      {attempt: 1, time: 0ms, response: 429, next_wait: 100ms},
      {attempt: 2, time: 100ms, response: 429, next_wait: 200ms},
      {attempt: 3, time: 300ms, response: 429, next_wait: 400ms},
      {attempt: 4, time: 700ms, response: 429, next_wait: 800ms},
      {attempt: 5, time: 1500ms, response: 200, status: SUCCESS}
    ]

    total_delay: 1500ms
    exposure_duration: 1.5s
    final: SUCCESS
  }
  ```

- Alert sequence:
  ```
  T=300ms: P2 alert - Rate limit, hedge delayed
  T=1500ms: P1 alert - Rate limit persist, exposure naked >1s
  ```

- Exposure risk assessment:
  ```
  Duration: 1.5s
  Potential price move: ±0.5% → Loss $2.5K (manageable)
  Hedge completed: Exposure normal covered ✓
  ```

**Monitoring Points**:
- Rate limit event frequency
- Hedge retry success rate
- Avg retry latency
- Exposure naked time cumulative

**Fallback on Failure**:
- 10 retries all fail → Switch HL_MODE + P0 alert
  ```
  Action:
    - Route all new→HL (no new INTERNAL)
    - Halt current INTERNAL
    - Prep manual intervention
  ```

- Rate limit too frequent → Negotiate quota with HL
- Partial success/failure → Post-trade reconcile remainder

---

## SC-EC-011: User Simultaneously INTERNAL Long + HL Short (Opposite Positions)

**Scenario Background**:
User legitimate hedging strategy: INTERNAL $5K BTC long, HL account $20K BTC short. Separate, independent, both monitored separately. System must handle dual liquidation, risk calcs per system.

**Input**:
- User: user_strategy_trader_001

- INTERNAL: BTC long $5K, 1x, equity $5K
- HL (trading): BTC short $20K, 2x, equity $10K

- BTC price: $60K → $55K (-8.3%)

**Decision Rule**:
```
def calculate_user_risk(user_id):
    # Fetch all positions (cross-system)
    internal_positions = get_positions(user_id, system=INTERNAL)
    hl_positions = get_positions(user_id, system=HYPERLIQUID)

    # Calculate each system separately
    for pos in internal_positions:
        pos.floating_pnl = calculate_pnl(pos, system=INTERNAL)
        pos.margin_ratio = calculate_margin_ratio(pos, system=INTERNAL)

    for pos in hl_positions:
        pos.floating_pnl = calculate_pnl(pos, system=HYPERLIQUID)
        # HL cross-margin aggregation

    # Aggregate without merging
    total_pnl = sum_all_floating_pnl(internal + hl)

    return {
        internal: {positions: [...], total_pnl: X},
        hl: {positions: [...], total_pnl: Y},
        overall_pnl: X + Y
    }

def handle_liquidation_for_user(user_id, price_change):
    # Separate handling

    # 1. INTERNAL check
    internal_pos = get_position(user_id, "BTC", system=INTERNAL)
    if (internal_pos.equity <= internal_pos.maintenance_margin):
        liquidate_position(internal_pos, system=INTERNAL)

    # 2. HL check (independent)
    hl_positions_user = get_all_positions(user_id, system=HYPERLIQUID)
    hl_account_equity = sum(p.equity for p in hl_positions_user)
    hl_total_maintenance = sum(p.maintenance_margin for p in hl_positions_user)

    if (hl_account_equity <= hl_total_maintenance):
        cascade_liquidation(user_id, system=HYPERLIQUID)

# Timeline:
Initial:
  INTERNAL BTC long $5K: Floating $0, Equity $5K, Maintenance $2.5K (200% ratio) ✓
  HL BTC short $20K (2x): Floating +$1.67K (short profit), Equity $11.67K ✓

BTC drops 8.3%:
  INTERNAL BTC long $5K: Floating -$416, Equity $4.58K, Maintenance $2.5K (183% ratio) ✓
  HL BTC short $20K: Floating more +$1.67K (short gains more), Equity stable+ ✓

Conclusion: Both safe, hedge strategy works ✓
```

**Output**:
- User aggregate view:
  ```
  USER_POSITIONS_AGGREGATE {
    user_id: user_strategy_trader_001

    internal: {
      positions: [BTC long $5K, floating -$416, SAFE],
      total_equity: $4.58K,
      total_maintenance: $2.5K,
      margin_ratio: 183%
    },

    hl: {
      positions: [BTC short $20K, floating +$1.67K, SAFE],
      account_equity: $11.67K,
      total_maintenance: $10K,
      margin_ratio: 117%
    },

    overall: {
      total_floating_pnl: -$416 + $1.67K = $1.25K (net profit, hedge works!)
      margin_ratio: N/A (separate)
    }
  }
  ```

- Risk monitoring (per-system):
  ```
  INTERNAL liquidation: Equity > Maintenance (183% > 100% ✓)
  HL liquidation: Account Equity > Aggregate Maintenance (117% > 100% ✓)
  ```

**Monitoring Points**:
- Multi-system position consistency per user
- Each system's margin ratio
- Hedge strategy net PnL contribution
- Cross-system position correlation

**Fallback on Failure**:
- State inconsistency (INTERNAL has, HL doesn't) → Reconcile + repair
- One system liquidates, other doesn't → Verify each independently → Manual remedy if needed
- User disputes mixed liquidation → Replay liquidation logic, correct if error

---

## SC-EC-012: System Restart State Recovery

**Scenario Background**:
System needs restart (bug fix, upgrade). All in-memory lost. Must rebuild from DB: user positions, INTERNAL limit queue, net exposure, HL hedge mapping. Restart latency expected 3–5 min, HL sync <10s post-recovery.

**Input**:
- Pre-restart state:
  ```
  Users: 450
  INTERNAL positions: 1,200
  HL hedge positions: 145
  Pending limit orders: 85
  Total INTERNAL net exposure: $1.23M
  Hedge coverage: 8 assets, $800K total
  System uptime: 30 days stable
  ```

- Restart latency: 3–5 min
- HL sync: <10s

**Decision Rule**:
```
def system_startup_recovery():
    log_info("System starting, recovery begins...")

    # Step 1: Load all state from DB
    log_info("Step 1: Load from database...")
    users = db.load_all_users()
    internal_positions = db.load_internal_positions()
    hl_hedge_positions = db.load_hl_hedge_positions()
    pending_limit_orders = db.load_pending_orders()

    log_info(f"  Loaded {len(users)} users")
    log_info(f"  Loaded {len(internal_positions)} INTERNAL positions")
    log_info(f"  Loaded {len(pending_limit_orders)} pending orders")

    # Step 2: Rebuild in-memory
    log_info("Step 2: Rebuild snapshots...")
    net_exposure = calculate_net_exposure(internal_positions)
    hedge_mapping = build_hedge_mapping(hl_hedge_positions)
    order_queue = rebuild_order_queue(pending_limit_orders)

    log_info(f"  Net exposure: ${net_exposure}")
    log_info(f"  Hedge mapping: {len(hedge_mapping)} assets")
    log_info(f"  Order queue: {len(order_queue)} orders")

    # Step 3: Reconnect HL
    log_info("Step 3: Reconnect Hyperliquid...")
    try:
        ws.connect(hl_endpoint)
        ws_connected = True
        log_info("  WS reconnected")
    except Exception as e:
        log_error(f"  WS fail: {e}")
        ws_connected = False
        trigger_fallback_mode()

    # Step 4: Full HL sync
    if (ws_connected):
        log_info("Step 4: Full HL sync...")
        try:
            hl_account_state = fetch_account_snapshot()
            hl_open_orders = fetch_open_orders()

            # Reconcile
            discrepancies = reconcile_hl_state(hl_account_state, hedge_mapping)

            if (discrepancies):
                log_warning(f"  Found {len(discrepancies)} mismatches")
                for disc in discrepancies:
                    log_warning(f"    {disc}")
                    update_local_state(disc)

            log_info("  Sync complete")
        except Exception as e:
            log_error(f"  Sync fail: {e}")
            trigger_alert("HL sync failed, entering limited mode", severity=P1)

    # Step 5: Data consistency check
    log_info("Step 5: Consistency checks...")
    check_results = run_consistency_checks(
        internal_positions,
        hl_hedge_positions,
        order_queue
    )

    if (check_results.passed):
        log_info("  ✓ All checks passed")
    else:
        log_error(f"  ✗ {len(check_results.failures)} checks failed")
        for failure in check_results.failures:
            log_error(f"    {failure}")
        trigger_alert("Consistency check failed, trading paused", severity=P0)
        skip_resumption = True

    # Step 6: Resume operations
    if (not skip_resumption):
        log_info("Step 6: Resume operations...")

        enable_new_orders()
        start_liquidation_checks()
        start_hedge_engine()
        start_api_service()

        log_info("✓ System recovered")
        publish_notification({
            type: "SYSTEM_RECOVERED",
            timestamp: now(),
            downtime: calculate_downtime()
        })
    else:
        log_info("× Consistency issues, system paused")
        publish_notification({
            type: "SYSTEM_RECOVERY_FAILED",
            reason: "data_consistency_failed",
            manual_required: True
        })

def consistency_checks():
    checks = [
        ("position_count", lambda: len(internal_positions) == db.count_positions()),
        ("net_exposure", lambda: abs(calc_exposure() - db.cached_exposure()) < tolerance),
        ("hedge_mapping", lambda: all(check_hedge_coverage())),
        ("order_unique", lambda: len(set(o.id for o in order_queue)) == len(order_queue)),
        ("balances_positive", lambda: all(u.balance >= 0 for u in users)),
    ]

    return run_checks(checks)
```

**Output**:
- Restart log (chronological):
  ```
  [2026-04-09 14:00:00] INFO: System startup...
  [2026-04-09 14:00:02] INFO: Step 1: DB load complete
                             450 users, 1,200 positions, 85 pending
  [2026-04-09 14:00:05] INFO: Step 2: Snapshots rebuilt
                             Net exposure $1.23M, 8 assets, 85 orders
  [2026-04-09 14:00:10] INFO: Step 3: WS reconnect to HL... success
  [2026-04-09 14:00:15] INFO: Step 4: Full sync... complete (no mismatches)
  [2026-04-09 14:00:18] INFO: Step 5: Consistency checks... ✓ Passed
  [2026-04-09 14:00:20] INFO: Step 6: Resume...
  [2026-04-09 14:00:22] INFO: ✓ System recovered (22s)
  ```

- Recovery report:
  ```
  SYSTEM_RECOVERY_REPORT {
    start: 2026-04-09 14:00:00Z
    complete: 2026-04-09 14:00:22Z
    total_time: 22s

    state_loaded: {
      users: 450,
      internal_positions: 1,200,
      hl_hedge_positions: 145,
      pending_orders: 85
    },

    ws_reconnect: 8s
    hl_sync: 5s
    consistency: 5/5 passed

    final_status: OPERATIONAL
  }
  ```

- User notification:
  ```
  System recovered (2026-04-09 14:00:22Z)
  Downtime: 22 seconds

  Your positions:
  - BTC long $50K ✓
  - Pending orders ✓
  - Account balance ✓

  Ready to trade. Contact support if any issues.
  ```

**Monitoring Points**:
- Recovery time breakdown (each step)
- DB load speed (data volume)
- WS reconnect success and latency
- Consistency check failure rate and types
- First order latency post-recovery

**Fallback on Failure**:
- DB unavailable → Try backup DB → Fail then wait for DBA
- WS reconnect fails → Degrade REST polling (latency higher but functional)
- Consistency check fails → Pause trading + P0 + manual review
- Partial position load failure → Continue with others + flag issues + post-recovery repair
- Recovery timeout (>5 min) → Likely deeper issue, restart again, check logs

---

## Appendix: Edge Case Trigger Conditions Quick Reference

| Scenario ID | Trigger | Severity | Recovery Time |
|-----------|---------|----------|---------------|
| SC-EC-001 | WS disconnect >60s | P1 | 1–2 min (reconnect) |
| SC-EC-002 | HL trading margin <300% | P0 | 10–30 min (capital) |
| SC-EC-003 | 1-hour volatility >5% | P1 | <5 min (mode switch) |
| SC-EC-004 | Account equity ≤ maintenance | P0 | <10s (liquidation) |
| SC-EC-005 | Deposit confirm >5 min timeout | P2 | Variable (chain confirm) |
| SC-EC-006 | 1-hour withdrawal >hot wallet | P1 | 30 min (consolidation) |
| SC-EC-007 | Funding rate deviation >0.001% | P2 | <1s (detect+correct) |
| SC-EC-008 | Admin config anomaly | P1 | <1 min (auto-revert) |
| SC-EC-009 | L3/L6 parallel execution | P3 | 100ms (eventual consistent) |
| SC-EC-010 | HL API 429 rate limit | P1 | 1.5s (backoff+retry) |
| SC-EC-011 | Multi-system opposite positions | P3 | No risk (separate liquidate) |
| SC-EC-012 | System restart | P0 | 20–30s (recovery) |

---

**End of Document**
