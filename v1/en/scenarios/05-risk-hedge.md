# Risk Control & Hedging Scenarios

**Document Version**: v1.0
**Last Updated**: 2026-04-09
**Author**: Claude Code
**Status**: Specification Phase

---

## Scenario Overview

This document details the core business scenarios for platform risk control and hedge engine (L6). Covers net exposure monitoring, automatic hedge triggers, capital management, routing mode switching, risk reserve management, circuit breaker mechanisms, multi-asset capital allocation, and hedge buffer issues—8 key scenarios total.

Each scenario is based on the "方案二 (Internalization + Threshold Hedging)" dual-account architecture:
- **Trading Account (HL Trading)**: Executes large user orders (>$10K)
- **Hedge Account (HL Hedge)**: Executes internal exposure hedges

---

## SC-RH-001: Net Exposure Grows from Zero to Trigger Hedge

**Scenario Background**:
Throughout a day, users continuously go long BTC. Platform accepts these small orders via L3 (INTERNAL internalization), and net exposure grows from $0. When accumulated net exposure crosses $100K threshold, L6 hedge engine triggers automatic hedge, opening hedge position on HL hedge account.

**Input**:
- Series of small orders: Order1($3K), Order2($5K), Order3($4K), Order4($8K)
- Pre-existing net exposure: $95K (all INTERNAL longs)
- New order: $8K BTC long (INTERNAL)
- Current HL hedge account leverage: 3x, available net capital: $100K

**Decision Rule**:
```
net_exposure = cumulative_INTERNAL_long - cumulative_INTERNAL_short

if (net_exposure > $100K):
    hedge_ratio = 50%
    hedge_amount = net_exposure * hedge_ratio
    execute_hedge(amount=hedge_amount, direction="long", symbol="BTC")
else:
    skip_hedge()

current_net_exposure = $95K + $8K = $103K > $100K
→ hedge_amount = $103K * 50% = $51.5K
```

**Output**:
- L6 generates hedge instruction: `HEDGE_ORDER{symbol: BTC, side: long, notional: $51.5K, leverage: 3x}`
- L4 proxy executes to HL hedge account, records execution price
- Hedge account position updated: BTC long $51.5K (3x = $17.17K margin occupied)
- L6 internal hedge mapping updated: `{symbol: BTC, internal_exposure: $103K, hedged: $51.5K, ratio: 50%}`
- Real-time monitoring dashboard updates net exposure and hedge coverage ratio

**Monitoring Points**:
- Real-time net exposure value and growth rate
- Hedge execution latency (target: <50ms P99)
- Hedge account margin ratio (target: >500%)
- Spread between execution price and real-time price
- Time interval between user order execution and hedge execution

**Fallback on Failure**:
- HL rejects hedge order (rate limiting) → Exponential backoff retry 3 times (100ms, 200ms, 400ms)
- All 3 retries fail → Send P1 alert → Stop accepting new INTERNAL orders → Auto-switch to HL_MODE (all orders→HL)
- Hedge account margin insufficient → Trigger SC-RH-002 (capital supplement flow)

---

## SC-RH-002: Hedge Account Capital Insufficient

**Scenario Background**:
Platform needs to hedge $400K internal exposure, but hedge account only has $100K available capital. At 3x leverage, can only cover $300K, creating $100K gap. System intelligently raises leverage or triggers capital top-up, ensuring hedge capacity isn't constrained by capital.

**Input**:
- Net exposure needing hedge: $400K BTC
- Target hedge ratio: 50% → Hedge amount $200K
- Hedge account current capital: $100K
- Current leverage: 3x (capacity $300K)
- Max allowed leverage: 5x (HL permitting, platform risk policy)

**Decision Rule**:
```
required_hedge = net_exposure * hedge_ratio = $400K * 50% = $200K

available_capacity = account_capital * current_leverage = $100K * 3x = $300K

if (required_hedge <= available_capacity):
    execute_hedge(amount=required_hedge)
else:
    required_leverage = required_hedge / account_capital = $200K / $100K = 2x
    if (required_leverage <= max_leverage):
        increase_leverage_to(required_leverage)
        execute_hedge(amount=required_hedge)
        trigger_fund_request()
    else:
        partial_hedge = account_capital * max_leverage
        execute_hedge(amount=partial_hedge)
        alert("Capital insufficient", severity=P1)
```

In this scenario:
- `required_leverage = $200K / $100K = 2x < max_leverage(5x)` → Raise leverage to 2x (or 2.5x with buffer)
- Execute full $200K hedge
- Simultaneously trigger capital supplement request (on-chain arrival: 10–30 min)

**Output**:
- Hedge account leverage adjusted from 3x to 2.5x (retain safety margin)
- Hedge instruction executed: $200K BTC hedge
- Capital supplement request generated: `FUND_REQUEST{amount: $100K, chain: TRON, priority: HIGH}`
- Admin receives Slack notification: "Hedge account capital gap $100K, supplement requested, expected 30 min arrival"
- Internal risk control log records leverage adjustment event

**Monitoring Points**:
- Hedge account leverage change history
- Capital occupation ratio (current exposure / account capital)
- Capital supplement request status (pending→on-chain→arrival→confirmed)
- Timestamp and reason for each leverage adjustment

**Fallback on Failure**:
- Leverage raise fails (HL API error) → Fallback to partial hedge: `partial_amount = $100K * 3x = $300K`, hedge $150K (75% of $200K)
- Try raising to 4x, re-execute → Hedge $200K (4x = $50K cost)
- Still fails or leverage maxed → Immediately switch HL_MODE, route all new orders to HL to avoid exposure
- Capital supplement delayed >1 hour → Consider manual top-up or pause internalization trading

---

## SC-RH-003: BTC Drops 10%, Hedge Position Floating Loss

**Scenario Background**:
Platform hedge account holds BTC long $500K (3x leverage), base capital $166.67K. Suddenly BTC market price drops 10%, hedge position incurs $50K floating loss, margin ratio drops from 500% to 350%. System must emergency-supplement from risk reserve to maintain hedge account health.

**Input**:
- Hedge account position: BTC long $500K @ 3x leverage (occupies $166.67K margin)
- Account total capital: $200K
- BTC price change: -10%
- Floating loss: -$50K
- Current margin ratio: `(Account equity / Occupied margin) = ($200K - $50K) / $166.67K = 150% / 166.67K ≈ 300%` (HL definition)
- Target margin ratio floor: 500% (platform risk policy)

**Decision Rule**:
```
account_equity = account_capital + unrealized_pnl = $200K - $50K = $150K
maintenance_margin_requirement = hedge_position / max_leverage_allowed
                               = $500K / 5x = $100K (conservative estimate)

margin_ratio = account_equity / maintenance_margin_requirement = $150K / $100K = 150%

if (margin_ratio < 500%):
    shortage = (target_ratio * maintenance - account_equity)
                = (5 * $100K) - $150K = $350K

    # Top-up minimum safe amount
    supplement = min(shortage, available_reserve)
    transfer_from_reserve(supplement)

if (shortage > available_reserve):
    alert("Reserve insufficient to cover floating loss", severity=P0)
    reduce_hedge_position()
```

In this scenario:
- Account equity $150K, margin ratio below safety threshold
- Transfer $50K from risk reserve → Account equity recovers to $200K → Margin ratio recovers to 500%
- Simultaneously issue P1 alert to Risk Manager for monitoring

**Output**:
- Risk reserve transfer: $50K → Hedge account (on-chain or internal transfer, <1 min)
- Hedge account equity updated: $200K (including floating loss)
- Margin ratio real-time monitoring: 500% ✓ (restored to safe)
- Alert log: `MARGIN_SHORTFALL_ALERT{account: hedge, shortage: $50K, supplement: $50K, reserve_balance_after: $XXK}`
- Slack notification: "Hedge account floating loss $50K, margin supplement completed"

**Monitoring Points**:
- Hedge account equity (capital + floating loss) real-time
- Margin ratio curve (refresh frequency: 1s)
- Frequency and amount of floating loss supplement triggers
- Risk reserve balance changes (logged per transfer)
- BTC price volatility correlation with hedge floating loss

**Fallback on Failure**:
- Reserve insufficient (balance<$50K) → Trigger SC-RH-005 (reserve supplement mechanism)
  - Pause new INTERNAL orders, all→HL
  - Issue P0 alert to admin for reserve top-up
  - Reduce hedge position in batches to free margin
- On-chain transfer delay (>5 min) → Record internally first, async verify chain state
- Position margin ratio continues deteriorating (drops below 150%) → Emergency reduce 50% of hedge to release margin

---

## SC-RH-004: Automatic Routing Mode Switch (NORMAL → HL_MODE)

**Scenario Background**:
Platform normally runs in NORMAL_MODE ($10K threshold), internal exposure stable at $300K–$400K. Suddenly large user/institution orders flood in, INTERNAL net exposure rapidly spikes to $800K, far exceeding safe capacity. System must detect and auto-switch to HL_MODE, forcing all new orders to Hyperliquid, preventing exposure runaway.

**Input**:
- Current routing mode: NORMAL_MODE (threshold $10K)
- Current net exposure: $550K (50% already hedged = $275K)
- New large user order series: $100K + $150K + $120K = $370K (total)
- Post-execution net exposure: $550K + $370K = $920K
- HL_MODE switch threshold: $800K (platform risk config)

**Decision Rule**:
```
def check_routing_mode():
    if (net_exposure > HL_MODE_threshold):
        if (current_mode != HL_MODE):
            switch_mode(NORMAL → HL_MODE)
            halt_internal_orders()
            notify_risk_manager()

check_point_A: net_exposure = $550K → OK, stay NORMAL_MODE
check_point_B: net_exposure = $550K + $220K (first two orders) = $770K → Approaching threshold, alert
check_point_C: net_exposure = $770K + $150K = $920K > $800K → Trigger auto-switch

# All subsequent orders route to HL
Order_4($50K) → HL (not INTERNAL)
Order_5($80K) → HL
```

**Output**:
- Routing mode switch: NORMAL_MODE → HL_MODE (event logged with timestamp)
- System broadcast: L2 Order Router receives mode switch signal
- All pending and subsequent order routing: `amount > 0 → HL_MODE → forward to L4 (HL Proxy Execution)`
- INTERNAL betting paused: New orders not routed to L3, existing INTERNAL positions continue settling
- Risk Manager receives Slack notification:
  ```
  🚨 ROUTING MODE AUTO-SWITCH
  Previous: NORMAL_MODE | Current: HL_MODE
  Trigger: Net Exposure $920K > Threshold $800K
  Action: All new orders → Hyperliquid
  Time: 2026-04-09 14:32:15 UTC
  ```
- Internal alert dashboard updated: Prominent display of current mode and trigger reason

**Monitoring Points**:
- Net exposure real-time value and growth rate (1s refresh)
- Mode switch event history (time, reason, trigger value)
- Post-switch order flow distribution (all new→HL ✓)
- INTERNAL position liquidation progress (if any)
- User experience during switch (latency, success rate)

**Fallback on Failure**:
- Auto-switch directive send fails → Retry 3 times → Fail then send P0 alert
- P0 alert unresponded → System auto-downgrades:
  1. Hard reject all new INTERNAL orders (return HTTP 429)
  2. All orders auto-route to HL
  3. Wait for Risk Manager manual confirmation and intervention
- Switch directive issued but some orders still routed INTERNAL → Data consistency check → Post-trade reconciliation

---

## SC-RH-005: Risk Reserve Falls Below $200K

**Scenario Background**:
Platform has accumulated $450K risk reserve (from net user losses in INTERNAL trading). Due to favorable market, users profit continuously, reserve gets consumed for hedge float supplements and operation costs. Reserve balance declines daily, finally drops below $200K safety floor. System must trigger reserve supplement flow, pause or restrict internalization trading.

**Input**:
- Prior reserve balance: $450K
- Daily consumption: $50K–$80K (hedge supplements, float coverage)
- Current balance: $180K (< $200K floor)
- Normal operation reserve target: ≥$500K
- Alert thresholds: $300K / $200K (yellow/red)

**Decision Rule**:
```
def monitor_reserve():
    if (reserve < $500K):
        severity = "YELLOW"  # Alert: recommend top-up
    if (reserve < $300K):
        severity = "ORANGE"  # Warning: start restricting internalization
    if (reserve < $200K):
        severity = "RED"     # Emergency: halt internalization
        action = "halt_internal_trading"

Current reserve = $180K < $200K → RED
→ halt_internal_trading()
→ switch_mode(NORMAL → HL_MODE / or FULL_HL)
→ notify_admin(urgency=P0)
```

**Output**:
- Risk reserve status change: HEALTHY → CRITICAL
- Trading restrictions activated: Pause all new INTERNAL orders
  - Route all new orders to HL (or reject directly)
  - Existing INTERNAL positions can continue closing
- Multi-channel admin notification:
  ```
  【EMERGENCY NOTICE】 Risk Reserve Balance $180K < Safety Floor $200K
  - INTERNAL internalization betting HALTED
  - All new orders routed to Hyperliquid
  - Immediate reserve supplement required
  - Target: Restore to $500K+
  ```
- Finance/Operations receive supplement request: `RESERVE_REPLENISHMENT_REQ{target: $500K, current: $180K, gap: $320K}`
- System enters "restricted operation" mode: Retain basic services, prohibit high-risk operations

**Monitoring Points**:
- Risk reserve balance (real-time)
- Daily change trend and rate prediction
- Supplement process status (request→approval→transfer→arrival)
- Traffic during restriction period (INTERNAL vs HL split)
- User complaints/churn rate (due to betting pause)

**Fallback on Failure**:
- Supplement delay (not completed within 1 day)
  - Maintain HL_MODE until arrival
  - Issue recurring P1 alert (every 4 hours)
- Supplement amount insufficient (only to $250K, target $500K)
  - Shift to phased targets: First restore $300K (lift ORANGE), resume restricted betting
  - Continue requesting supplement to $500K
- Admin unable to complete supplement (fund liquidity issue)
  - Maintain HL_MODE long-term
  - Aggressive risk control: Limit leverage, caps, gradually shrink INTERNAL operations

---

## SC-RH-006: Daily Loss Circuit Breaker Triggers

**Scenario Background**:
Platform's INTERNAL internalization has daily PnL swings. Normal daily variance ±$20K, but extreme days may see concentrated user profit, platform net loss. System implements daily loss circuit breaker: when daily INTERNAL net loss reaches $500K, auto-halt betting, preventing extreme risk.

**Input**:
- Trading day: 2026-04-09 (UTC 00:00 – 23:59)
- INTERNAL cumulative user PnL: +$520K (user profit = platform loss)
- Alert threshold: $100K
- Trigger threshold: $500K
- Reset time: Next day 00:00 UTC

**Decision Rule**:
```
# After each INTERNAL execution, accumulate daily loss
daily_loss = sum(all_internal_trades_pnl_today)

if (daily_loss < -$100K):
    log_alert("Daily loss alert", severity=P2)
    notify_risk_manager()

if (daily_loss < -$500K):
    log_alert("Daily loss circuit breaker", severity=P0)
    halt_internal_trading()
    switch_mode(HL_MODE)
    update_circuit_breaker_status(TRIGGERED)

# Next day 00:00 UTC auto-reset
if (utc_hour == 0 and utc_minute == 0):
    reset_daily_loss_counter()
    circuit_breaker_status = RESET
    if (mode == HL_MODE_due_to_loss):
        # Auto-try to restore NORMAL_MODE (if other metrics allow)
        try_restore_normal_mode()
```

Timeline example:
```
09:30 - Daily loss -$20K (OK)
12:15 - Daily loss -$80K (OK)
14:45 - Daily loss -$120K → Alert P2
16:20 - Daily loss -$350K → Alert upgraded P1
18:05 - Daily loss -$510K > -$500K → Breaker triggers P0
        → Halt INTERNAL betting
        → Switch HL_MODE
        → circuit_breaker = TRIGGERED
        → Notify Risk Manager + CTO

Next day 2026-04-10 00:00:00 UTC:
        → daily_loss_counter = 0
        → circuit_breaker = RESET
        → Try restore NORMAL_MODE (if exposure permits)
```

**Output**:
- Daily net loss real-time cumulative: $510K
- Breaker status change: ARMED → TRIGGERED
- Trading halt directive: halt_internal_trading()
- Routing mode forced switch: NORMAL_MODE → HL_MODE (reason marked "LOSS_CIRCUIT_BREAKER")
- P0 alert notification (multi-channel):
  ```
  🔴 CIRCUIT BREAKER TRIGGERED
  Type: Daily Loss Limit
  Current Loss: -$510K
  Threshold: -$500K
  Action: All internal trading HALTED
  Mode: HL_MODE (all orders → Hyperliquid)
  Auto-Reset: 2026-04-10 00:00:00 UTC
  ```
- Internal log: `{timestamp, daily_loss, trigger_value, action, user_impact}`
- User API notification: Subsequent betting requests return error code (e.g., 429), message "Platform maintenance"

**Monitoring Points**:
- Daily net loss real-time (aggregate every minute)
- Daily loss trend chart (hourly)
- Alert and breaker trigger event history
- Order flow distribution during halt (100% → HL)
- INTERNAL recovery post-breaker reset

**Fallback on Failure**:
- Breaker directive fails (some orders still route INTERNAL) → Post-trade reconciliation + force close
- Auto-reset fails (still not reset at next day 00:00) → Manual reset + alert
- User disputes circuit breaker decision (claims loss unfair) → Event replay, possible manual review

---

## SC-RH-007: Multi-Asset Hedge, Capital Allocation

**Scenario Background**:
Platform supports multi-asset trading (BTC, ETH, SOL, etc.). On high-volatility day, multiple assets accumulate large net exposure, all need hedging. But hedge account's available capital is limited, can't fully satisfy all assets. System must implement intelligent capital allocation: rank by exposure size, prioritize covering largest, smaller may switch to HL_MODE.

**Input**:
- Hedge account total capital: $200K, current leverage: 3x, available capacity: $600K
- Net exposure distribution:
  - BTC long: $600K (56% of total)
  - ETH long: $300K (28% of total)
  - SOL long: $150K (14% of total)
  - Total: $1,050K
- Hedge ratio: all 50% (unified standard)
- Hedge requirements:
  - BTC: $600K * 50% = $300K → Scale up to $480K (cover majority)
  - ETH: $300K * 50% = $150K
  - SOL: $150K * 50% = $75K
  - Total needed: $480K + $150K + $75K = $705K > available $600K

**Decision Rule**:
```
def allocate_hedge_across_symbols():
    total_demand = sum(symbol.exposure * hedge_ratio for symbol in symbols)
    total_capacity = account_capital * current_leverage

    if (total_demand <= total_capacity):
        # Sufficient, satisfy all
        for symbol in symbols:
            execute_hedge(symbol, symbol.exposure * hedge_ratio)
    else:
        # Insufficient, allocate by size priority
        symbols_by_size = sort_by_exposure(symbols, desc=True)
        remaining_capacity = total_capacity

        for symbol in symbols_by_size:
            required = symbol.exposure * hedge_ratio
            if (remaining_capacity >= required):
                execute_hedge(symbol, required)
                remaining_capacity -= required
            else:
                # Can't satisfy, split handling
                if (remaining_capacity > 0):
                    execute_hedge(symbol, remaining_capacity)
                    remaining_capacity = 0
                # Remaining exposure switch to HL_MODE
                alert(f"{symbol} exposure switch to HL_MODE")
                switch_symbol_to_hl(symbol)

Execution order in this scenario:
1. BTC (largest) → Hedge $480K (consumes $160K capital @ 3x)
   remaining = $600K - $160K = $440K

2. ETH (medium) → Hedge $150K (consumes $50K capital @ 3x)
   remaining = $440K - $50K = $390K

3. SOL (smallest) → Need $75K
   Can actually hedge: $390K / 3x = $130K (covers $75K)
   Hedge $75K (consumes $25K capital @ 3x)
   remaining = $390K - $25K = $365K

Result: All assets satisfiable ✓
```

**Output**:
- Hedge instruction sequence (by priority):
  ```
  HEDGE_ORDER_1: {symbol: BTC, amount: $480K, leverage: 3x}
  HEDGE_ORDER_2: {symbol: ETH, amount: $150K, leverage: 3x}
  HEDGE_ORDER_3: {symbol: SOL, amount: $75K, leverage: 3x}
  ```
- Hedge account position updated:
  - BTC long $480K
  - ETH long $150K
  - SOL long $75K
  - Total margin occupied: ($480K + $150K + $75K) / 3x = $235K (exceeds capacity $35K)

  **Correction**: If total occupied exceeds account capital, must trim. Example with true capacity $200K:
  - BTC: $400K (67%)
  - ETH: $120K (20%)
  - SOL: $60K (10%)
  - Unhedged portions (BTC +$80K, ETH +$30K) → Switch to HL_MODE for those assets

- Multi-asset hedge mapping updated: `{BTC: {internal: $600K, hedged: $480K}, ETH: {...}, SOL: {...}}`
- Monitoring dashboard: Coverage ratios per asset (BTC 80%, ETH 50%, SOL 50%)

**Monitoring Points**:
- Real-time net exposure by asset
- Hedge coverage ratio per asset (target 50%)
- Total hedge account margin ratio
- Proportion of unhedged exposure switched to HL
- Cross-asset margin interaction (if any)

**Fallback on Failure**:
- One asset's hedge order fails (HL low liquidity) → Partial hedge for that asset + switch remainder to HL
- Hedge account sudden liquidation (extreme) → Force reduce all assets 50% → Switch corresponding INTERNAL to HL
- Allocation algorithm error → Conservative fallback: evaluate each asset independently, unhedgeable switch to HL

---

## SC-RH-008: Concentrated User Close Causes Over-hedge

**Scenario Background**:
Platform hedged large user's BTC long $800K (INTERNAL) via HL hedge account long $640K (80% hedge + 10% buffer). That user suddenly closes all positions within 1 hour. INTERNAL exposure drops $800K → $0. Platform hedge account must de-hedge from $640K → $0, but can't de-hedge too fast (slippage, market impact). Must execute in batches, gradually.

**Input**:
- Initial state:
  - INTERNAL user long: $800K BTC
  - HL hedge account long: $640K BTC (3x leverage, occupies $213.3K capital)
  - Hedge ratio: 80% (= $800K * 80%)

- User close operations:
  - Time: 2026-04-09 14:00:00 UTC
  - Close speed: Complete within 1 hour (or 30 min continuous)
  - Execution:
    - 14:10 -$150K
    - 14:20 -$200K
    - 14:35 -$250K
    - 14:50 -$200K
    - Total -$800K complete

- Post-close state:
  - INTERNAL exposure: $0
  - Hedge need: $0 (below $100K threshold, should fully remove hedge)
  - Hedge account should hold: $0

**Decision Rule**:
```
def adjust_hedge_on_position_change():
    new_exposure = get_current_net_exposure()
    current_hedge = get_current_hedge_position()

    # Calculate target based on hedge threshold and ratio
    if (new_exposure < HEDGE_THRESHOLD):
        target_hedge = 0
    else:
        target_hedge = new_exposure * hedge_ratio

    hedge_adjustment = current_hedge - target_hedge

    if (hedge_adjustment > 0):
        # Need de-hedge
        reduce_hedge(hedge_adjustment)
    elif (hedge_adjustment < 0):
        # Need add hedge
        increase_hedge(-hedge_adjustment)

# In this scenario:
trigger_point_1: exposure = $800K - $150K = $650K
                target_hedge = $650K * 80% = $520K
                adjust = $640K - $520K = $120K (need reduce $120K)
                execute_reduce($120K, max_rate=$200K/min)

trigger_point_2: exposure = $450K
                target_hedge = $450K * 80% = $360K
                adjust = $520K - $360K = $160K (need reduce $160K)
                execute_reduce($160K, max_rate=$200K/min)

...finally...

trigger_point_N: exposure = $0 < $100K_threshold
                target_hedge = 0
                adjust = remaining_hedge (full de-hedge)
                execute_reduce_all(remaining_hedge, max_rate=$200K/min)
```

Each reduction limited to max $200K/min (prevent market impact)

**Output**:
- De-hedge plan generated:
  ```
  HEDGE_REDUCTION_PLAN:
  - Batch 1 (14:00-14:05): Reduce $200K
  - Batch 2 (14:05-14:10): Reduce $200K
  - Batch 3 (14:10-14:15): Reduce $240K (final)
  ```

- Hedge account position step changes:
  ```
  Time        INTERNAL Exp   Hedge Acc   Hedge Ratio
  14:00       $800K          $640K       80%
  14:05       $650K          $440K       67.7%  (in progress)
  14:10       $450K          $240K       53.3%  (in progress)
  14:15       $0K            $0K         0%     (complete)
  ```

- De-hedge execution records: Each batch submitted to HL, logs execution price, slippage, fees
- Hedge mapping updated: BTC hedge $640K → $0

**Monitoring Points**:
- INTERNAL exposure change rate (detect unusual close)
- HL hedge account position vs. expected exposure gap
- Each de-hedge batch timing, execution price, slippage
- Hedge account margin ratio changes (de-hedge releases margin)
- Total de-hedge completion time

**Fallback on Failure**:
- De-hedge slippage too large (>1%) → Lower de-hedge rate, extend timeline
  ```
  Original: $200K/min → Reduce to $100K/min
  Replan, extend to 8–10 min completion
  ```

- De-hedge order rejected (HL low liquidity) → Exponential backoff retry + mark delay state

- Mid-de-hedge, INTERNAL exposure reverses (user adds position) → Dynamically recalculate target, pause or reverse add

- De-hedge exceeds 30 min → Issue alert + human review + consider market order acceleration

---

## SC-RH-009: (Reserved) Internalization Execution vs. Hedge Directive Race Condition

This scenario detailed in "Edge Cases & Extreme Conditions" (06-edge-cases.md) SC-EC-009.

---

## SC-RH-010: (Reserved) HL API Rate Limiting Causes Hedge Delay

This scenario detailed in "Edge Cases & Extreme Conditions" (06-edge-cases.md) SC-EC-010.

---

## Appendix: Key Metrics & Alert Thresholds Summary

| Metric | Alert Threshold (Yellow) | Trigger Threshold (Red) | Action |
|--------|-------------------------|----------------------|--------|
| Net Exposure | $500K | $800K | Switch HL_MODE |
| Hedge Account Margin Ratio | 300% | 150% | Top-up capital |
| Risk Reserve | $300K | $200K | Halt internalization |
| Daily Net Loss | -$100K | -$500K | Circuit breaker |
| HL API Latency | 200ms | 500ms | Fallback/Switch HL |
| Data Sync Latency | 1s | 5s | Alert + manual |

---

**End of Document**
