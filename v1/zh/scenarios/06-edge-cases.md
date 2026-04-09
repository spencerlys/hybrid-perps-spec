# 边缘与异常场景

**文档版本**: v1.0
**最后更新**: 2026-04-09
**作者**: Claude Code
**状态**: 规范阶段 (Specification Phase)

---

## 场景概述

本文档详细描述了平台在极端条件、故障状态和异常输入下的行为。涵盖网络断连、外汇账户风险、市场极端波动、级联清算、资产确认延迟、挤兑风险、费率偏差、管理员误操作、并发竞态、API限流、多账户混合、以及系统重启恢复等12个关键边缘场景。

这些场景确保平台在非理想条件下仍能保持数据一致性、风险控制，以及可恢复性。

---

## SC-EC-001：HL WebSocket断连超过1分钟

**场景背景**：
平台通过WebSocket实时接收Hyperliquid市场数据（价格、资金费率、订单簿等）。WS连接突然断开（网络故障、HL服务波动等），超过1分钟未恢复。平台需检测断连、暂停依赖该数据的操作、尝试重连，最后全量同步以恢复一致性。

**输入**：
- WS连接状态：CONNECTED → DISCONNECTED (T=0ms)
- 断连时长：60s+（持续增长）
- 重连策略：指数退避 (100ms, 200ms, 500ms, 1s, 2s, ...)
- 平台内状态：
  - INTERNAL净敞口：$350K（已缓存）
  - 对冲账户HL持仓：BTC多$280K（已缓存）
  - 待成交限价单：3笔，缓存状态未更新

**决策规则**：
```
def monitor_ws_connection():
    while True:
        if (ws.is_connected()):
            last_msg_time = current_time()
        else:
            disconnect_duration = current_time() - last_msg_time

            if (disconnect_duration > 10s):
                log_alert("WS断连", severity=P2)

            if (disconnect_duration > 60s):
                log_alert("WS长时断连", severity=P1)
                handle_long_disconnect()

def handle_long_disconnect():
    # 第一步：暂停风险操作
    halt_new_orders()  # 新订单不接收（返回错误）
    pause_liquidation_checks()  # 清算检查暂停（改为降频）
    halt_hedge_orders()  # 对冲指令不发送

    # 第二步：尝试重连（指数退避）
    for attempt in range(max_retries=10):
        wait_time = min(100ms * (2 ^ attempt), 30s)
        if (ws.reconnect()):
            log_info(f"WS重连成功，耗时{wait_time}")
            break
        sleep(wait_time)
    else:
        log_alert("WS重连失败", severity=P0)
        trigger_fallback_mode()
        return

    # 第三步：全量同步
    sync_all_state():
        # 拉取最新快照
        hl_account_state = fetch_account_snapshot()
        hl_positions = fetch_positions()
        hl_orders = fetch_open_orders()
        market_prices = fetch_latest_prices()

        # 本地状态比对与修正
        reconcile_positions(local_cache, hl_account_state)
        reconcile_orders(local_cache, hl_orders)

        # 恢复正常运营
        resume_operations()

def trigger_fallback_mode():
    # WS无法恢复 → 降级到REST轮询
    log_alert("WS不可用，启用REST轮询模式", severity=P0)
    ws_available = False
    poll_interval = 5s  # 每5秒轮询一次
    poll_endpoints = [
        "/account",
        "/user/positions",
        "/user/orders"
    ]
```

**输出**：
- T=0ms: WS断开事件日志
- T=10s: P2告警：「WS断连已10秒，监控中」
- T=60s: P1告警：「WS断连已60秒，开始应急处理」
- T=60s-120s: 重连尝试序列（4-5次指数退避尝试）
- T=120s: WS恢复连接 → 全量同步启动
- T=125s: 同步完成 → 验证数据一致性
- T=130s: 恢复新订单接收 + 清算检查 + 对冲指令
- 系统状态标记：`ws_status: CONNECTED, last_sync: 2026-04-09 14:32:15Z, sync_gap: 2m15s`

**监控点**：
- WS连接状态实时标记（CONNECTED / DISCONNECTED）
- 断连时长累计
- 重连次数与最后重连时间
- 全量同步耗时（目标 <10s）
- 数据完整性检查（位置、订单数、价格鲜度）
- 降级模式激活标记（REST轮询）

**失败兜底**：
- 重连失败5次仍不可用 → 进入维护模式（MAINTENANCE）
  ```
  系统状态: MAINTENANCE
  用户API: 返回503 Service Unavailable
  新订单: 拒绝（HTTP 503）
  现有仓位: 只允许平仓（卖出市价单）
  清算: 暂停自动清算，人工审核
  对冲: 暂停对冲指令
  通知: P0告警 + 人工干预开始
  ```

- 全量同步失败（数据不一致） → 数据校验失败 → 继续MAINTENANCE → 人工修复

- 最坏情况（WS + REST都不可用） → 系统进入只读模式，禁止交易

---

## SC-EC-002：HL交易账户保证金率跌破150%

**场景背景**：
平台在HL交易账户执行用户的大额订单（>$10K），持有用户代理持仓。该账户初始保证金率500%，突然因市场剧烈波动，浮亏扩大，账户权益下降，保证金率快速跌至150%（接近HL清算线200%）。系统需立即采取紧急行动防止HL强制清算该账户。

**输入**：
- HL交易账户：
  - 账户权益：$300K（原$500K，浮亏$200K）
  - 占用保证金：$200K（维持保证金）
  - 保证金率：300K / 200K = 150%（危险水位！）
  - 平台风险政策下限：500%（目标）
  - HL强制清算线：200%（绝对底线）

- 持仓分布：
  - BTC多$1M（浮亏$150K）
  - ETH短$500K（浮亏$50K）
  - 总浮亏$200K

- 资金池可用：$500K
- 应急补资需求：$200K（恢复到400K权益，保证金率400%）

**决策规则**：
```
def monitor_trading_account_margin():
    margin_ratio = account_equity / maintenance_margin_required

    if (margin_ratio < 200%):
        # 绝对紧急：即将清算
        severity = CRITICAL
        trigger_emergency_liquidation_prevention()

    elif (margin_ratio < 300%):
        # 高风险：超过阈值
        severity = P0_ALERT
        take_immediate_action()

    elif (margin_ratio < 500%):
        # 中等风险：监控中
        severity = P1_ALERT
        notify_risk_manager()

def take_immediate_action():
    # 第一步：暂停所有新HL订单
    halt_hl_new_orders()

    # 第二步：立即紧急补资（链上操作）
    emergency_fund_transfer(amount=$200K, source=fund_pool, dest=hl_trading_account)
    # 备注：链上操作需10-30分钟确认，期间账户风险持续

    # 第三步：同步考虑平仓止损（保守）
    # 如补资未及时到账，考虑平仓小部分头寸释放保证金
    if (margin_ratio < 250%):
        consider_partial_liquidation(amount=$100K, priority="floating_loss_largest")

    # 第四步：通知Risk Manager + CTO
    notify_critical_incident()

# 本场景时间线：
T=0:   margin_ratio = 150% < 300% → P0告警
       → halt_hl_new_orders()
       → emergency_fund_transfer($200K)
       通知: "HL交易账户保证金率跌至150%，已暂停新订单，正紧急补资"

T+1m:  margin_ratio仍150% （补资链上处理中）
       → 考虑部分平仓
       → 评估：平仓BTC多$50K释放~$20K保证金
       → 等待补资或执行平仓（Risk Manager决策）

T+15m: 补资上链确认 → 账户权益恢复 $300K + $200K = $500K
       margin_ratio = $500K / $200K = 250% （好转，但仍需监控）

T+30m: 补资最终到账 → 账户权益 $500K+（假设市场无更坏变化）
       margin_ratio = 500%+ → 恢复安全
       → 恢复HL新订单接收
       → 继续监控
```

**输出**：
- P0告警：「HL交易账户保证金率跌至150%，即将触发清算」
- 操作指令：
  - 暂停所有新HL订单（立即生效）
  - 非对冲性大额用户订单被拒，返回错误：`{code: SERVICE_UNAVAILABLE, msg: "HL account margin insufficient"}`

- 紧急补资请求：$200K → HL交易账户
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

- 用户通知（如适用）：后续大额订单返回503（HL账户维护中）

- 内部管理面板：
  - HL交易账户保证金率实时显示：150% (红色警告）
  - 补资状态跟踪（待发起→链上处理→入账确认）
  - 推荐行动列表（等待补资/平仓止损/强平个别头寸）

**监控点**：
- HL交易账户保证金率（实时，刷新频率1s）
- 账户权益曲线
- 持仓浮亏分布
- 补资流程状态（链上确认数）
- 期间用户订单拒绝率

**失败兜底**：
- 补资延迟超过15分钟，保证金率继续下跌 <200%
  → 执行部分紧急平仓：
    - 平仓浮亏最大的仓位（BTC多$50K）
    - 预期释放$20K保证金 → margin_ratio 可能恢复到180%-200%
    - 继续等待补资或进一步平仓

- 平仓本身受限（HL滑点过大、流动性不足）
  → 改用限价单平仓，但可能无法及时成交
  → 最后兜底：接受HL强制清算（覆盖保证金亏损从风险准备金）

- 补资来源不足（资金池已耗尽）
  → 启动应急融资流程（如有合作方）或接受清算

---

## SC-EC-003：BTC价格1分钟闪崩15%

**场景背景**：
市场突发极端事件（闪崩、连环清算、黑天鹅事件），BTC价格在1分钟内暴跌15%。这是"波动度超过5%/小时"的强制路由规则的触发条件，也是对冲账户浮亏的巨大风险。平台需切换到防御模式、暂停INTERNAL对赌、密切监控对冲账户、高频清算检查。

**输入**：
- BTC价格变化：$60,000 → $51,000 (1分钟内，-15%)
- 波动度：15% / 1min = 15%/min = 900%/hour （远>5%）
- 平台当前状态：
  - 模式：NORMAL_MODE
  - INTERNAL BTC净敞口：$1.2M（多头）
  - HL对冲账户BTC多头：$600K（3x杠杆，占用$200K）
  - 对冲账户权益：$300K

- 波动检测：实时监控最近1分钟的价格变化百分比

**决策规则**：
```
def monitor_market_volatility():
    price_1m_ago = get_price_at(now - 1min)
    price_now = get_price_now()
    volatility_1min = abs((price_now - price_1m_ago) / price_1m_ago)

    # 转换为小时波动率（近似）
    volatility_per_hour = volatility_1min * 60

    if (volatility_per_hour > 5%):
        log_alert("市场波动过大", severity=P1)
        force_route_to_hl()
        halt_internal_trading()
        trigger_liquidation_check()
        monitor_hedge_margin()

def force_route_to_hl():
    # 暂停INTERNAL对赌，所有订单转HL
    routing_mode = HL_MODE
    log_info("强制切换到HL_MODE: 波动度过高")

def trigger_liquidation_check():
    # 高频清算检查（降低延迟）
    check_interval = 1s  # 而非正常的5s
    liquidation_check_frequency = HIGH

def monitor_hedge_margin():
    # 对冲仓位浮亏风险
    hl_hedge_account.margin_ratio = (account_equity) / (maintenance_margin)

    # 对冲账户浮亏计算
    hedge_notional = $600K
    price_loss = 15% * $600K = $90K (浮亏)
    account_equity = $300K - $90K = $210K
    maintenance_margin ≈ $600K / 5x = $120K（保守）
    margin_ratio = $210K / $120K = 175% （危险！)

    if (margin_ratio < 200%):
        alert("对冲账户保证金率危险", severity=P0)
        consider_reduce_hedge_position()

    if (margin_ratio < 150%):
        alert("对冲账户即将清算", severity=CRITICAL)
        emergency_reduce_hedge()
```

**输出**：
- 波动度检测：15% / 1min 触发 > 5%/hour 规则
- 立即操作：
  1. 切换路由模式：NORMAL_MODE → HL_MODE
  2. 暂停INTERNAL对赌：新订单全部拒绝或转HL
  3. 清算检查升级为高频（1s扫描）
  4. 清算事件：可能大量用户触发清算条件 → L5清算引擎激活

- 对冲账户警报：
  ```
  MARGIN_ALERT {
    account: hl_hedge
    current_ratio: 175%
    threshold: 200%
    status: CRITICAL
    action: 考虑减仓对冲头寸
  }
  ```

- 用户通知（API错误码）：
  ```
  新订单请求返回:
  {
    code: MARKET_VOLATILITY_HALT,
    message: "市场波动过大，暂停对赌服务。订单已转Hyperliquid。",
    order_routed_to: "HYPERLIQUID"
  }
  ```

- 内部通知：
  - P0告警给Risk Manager + CTO
  - 对冲账户实时监控面板高亮显示
  - 清算事件流实时日志

**监控点**：
- 实时波动度（1分钟、5分钟、1小时）
- BTC价格K线走势
- 对冲账户浮亏与保证金率
- 清算触发次数与用户数
- INTERNAL转HL的订单分流比例
- 市场极端波动期间的系统延迟

**失败兜底**：
- 对冲账户浮亏继续恶化（BTC再跌5%） → 触发SC-EC-002（保证金率跌破150%流程）
  → 紧急补资 + 部分减仓对冲

- 清算检查计算延迟（>5s） → 某些用户清算被延迟发送
  → 事后对账，补偿性清算

- HL市场深度不足，转HL的订单成交不了 → 用户订单被拒 → 用户体验下降
  → 可能需要激励重试或人工处理

- 若波动达到极限（30%+闪崩），系统可能无法应对 → 考虑临时禁止交易，进入维护模式

---

## SC-EC-004：用户爆仓链式反应 (全仓模式)

**场景背景**：
用户在全仓模式（Cross-Margin）下持有5个BTC、ETH、SOL等仓位。由于市场暴跌，账户权益快速下降，触发清算线（Account Equity ≤ Total Maintenance Margin）。清算引擎需按规则逐个平仓，但各仓位平仓后权益重算，可能继续触发清算，导致级联清算。系统需确保清算顺序正确、对账准确。

**输入**：
- 用户账户模式：全仓（Cross-Margin）
- 账户总权益：$50K
- 总维持保证金需求：$40K
- 初始保证金率：125% (= $50K / $40K)
- 仓位列表（按浮亏排序）：
  ```
  Pos1: BTC 多1x    初始资本$10K → 当前浮亏-$8K → 权益$2K
  Pos2: ETH 多3x    初始资本$5K  → 当前浮亏-$4K → 权益$1K
  Pos3: SOL 多2x    初始资本$8K  → 当前浮亏-$3K → 权益$5K
  Pos4: DOGE 空1x   初始资本$2K  → 当前盈利+$1K → 权益$3K
  Pos5: LINK 多1x   初始资本$25K → 当前浮亏-$2K → 权益$23K
  总权益: $2K + $1K + $5K + $3K + $23K = $34K
  ```

- BTC暴跌触发：价格-20% → Pos1和Pos5进一步亏损 → 总权益降到$20K < $40K维保 → 清算触发

**决策规则**：
```
def cascade_liquidation(user_account):
    while True:
        account_equity = sum(position.equity for position in account)
        total_maintenance = sum(position.maintenance_margin for position in account)

        if (account_equity <= total_maintenance):
            # 触发清算
            log_alert("清算触发", user=user_account.id)

            # 清算策略：按浮亏从大到小，优先平仓
            positions_by_loss = sort_by_floating_loss(account.positions, desc=True)

            for position in positions_by_loss:
                if (account_equity > total_maintenance):
                    # 已恢复安全 → 停止清算
                    break

                # 平仓该仓位（市价单）
                execute_market_liquidation(position)

                # 重新计算账户权益和维保
                account_equity = recalculate_equity()
                total_maintenance = recalculate_total_maintenance()

                # 记录每次清算后的权益变化
                log_cascade_step(position.id, account_equity, total_maintenance)
        else:
            break  # 不需清算

# 时间线：
T=0:   账户权益$20K < 维保$40K → 清算启动

       清算顺序（按浮亏）：
       1. Pos1 (BTC多) 浮亏-$8K （最大亏损）

T+1s:  平仓Pos1 (BTC多1x) → 释放$10K资本
       账户权益 = $20K + $10K（资本回收） - 滑点 = $28K
       总维保 = $30K（移除Pos1后） → 仍需清算

T+2s:  清算继续: Pos2 (ETH多3x) 浮亏-$4K
       平仓Pos2 → 释放$5K资本
       账户权益 ≈ $28K + $5K - 滑点 = $32K
       总维保 = $24K（移除Pos1, Pos2后） → 权益 > 维保 ✓

T+3s:  清算停止（账户已恢复安全）
       最终状态：Pos3, Pos4, Pos5保留，Pos1, Pos2已平仓
       最终权益: ~$32K，保证金率: 32K / 24K = 133%
```

**输出**：
- 清算事件序列：
  ```
  LIQUIDATION_CASCADE {
    user_id: user_ABC
    trigger_time: 2026-04-09 14:32:15Z
    initial_equity: $20K
    initial_maintenance: $40K

    cascade_steps: [
      {
        step: 1
        liquidated_position: {id: Pos1, symbol: BTC, side: long, notional: $10K}
        execution: market_order, price: $48,000 (滑点-1%)
        capital_released: $10K
        post_liquidation_equity: $28K
        post_liquidation_maintenance: $30K
        ratio_after: 93% (unsafe)
      },
      {
        step: 2
        liquidated_position: {id: Pos2, symbol: ETH, side: long, notional: $5K}
        execution: market_order, price: $2,400 (滑点-1.5%)
        capital_released: $5K
        post_liquidation_equity: $32K
        post_liquidation_maintenance: $24K
        ratio_after: 133% (safe ✓)
      }
    ]

    final_equity: $32K
    final_maintenance: $24K
    final_margin_ratio: 133%
    liquidation_complete: true
  }
  ```

- 用户通知：
  ```
  账户清算已启动：您的仓位由于保证金不足已被部分平仓。
  已平仓: BTC多1x ($10K), ETH多3x ($5K)
  当前保证金率: 133%（已恢复安全）
  剩余仓位: SOL多, DOGE空, LINK多
  ```

- 内部清算日志：记录每一步，包括成交价、滑点、中间权益

**监控点**：
- 级联清算步数与总耗时（目标<10s完成）
- 每步清算后的权益与保证金率变化
- 清算成交价与实时价格的偏差（滑点）
- 被清算用户数与仓位数
- INTERNAL vs HL仓位的清算混合情况

**失败兜底**：
- 某步清算失败（订单被拒或无法成交） → 尝试重新提交清算单
  → 如连续失败，降级为限价单，但可能无法及时完成
  → 最后兜底：人工干预，Risk Manager直接平仓

- 清算中途获取价格延迟 → 使用缓存价格或上一个已知价格进行清算
  → 事后偏差对账

- INTERNAL仓位与HL仓位混合清算 → 各自路径执行
  - INTERNAL: 内部结算 (L5 Liquidation)
  - HL: 发市价平仓指令到HL (L4 HL Proxy)
  → 可能时间不同步，事后对账

- 级联清算未能完全恢复安全 (权益仍<维保) → 继续进行追加清算，直至恢复

---

## SC-EC-005：充值确认延迟导致余额争议

**场景背景**：
用户在TRON链充值$10K USDT到平台钱包，交易已发出。平台监听TRON链确认，要求19个区块确认。但在确认过程中，网络条件波动导致确认中断、重连，或确认进度卡滞。系统需确保不重复入账、使用幂等性、最终一致性确认。

**输入**：
- 充值信息：
  - 用户ID: user_XYZ
  - 链: TRON
  - 资产: USDT (TRC20)
  - 金额: $10K
  - 交易哈希: 0xabc123...
  - 发起时间: 2026-04-09 14:00:00Z

- 确认目标: 19个区块确认（约57秒，TRON 3秒/块）
- 预期完成: 14:01:00Z

- 监听进度（中途中断）：
  ```
  14:00:05 - Block 50000确认 (1/19)
  14:00:08 - Block 50001确认 (2/19)
  ...
  14:00:35 - Block 50011确认 (12/19)
  14:00:36 - [网络中断] → 监听暂停
  14:00:56 - [重新连接] → 从Block 50012继续确认
  14:01:00 - Block 50018确认 (19/19) ✓ 完成
  ```

**决策规则**：
```
def monitor_deposit_confirmation(tx_hash, chain, required_confirmations=19):
    deposit_record = db.get_deposit_by_txhash(tx_hash)
    confirmation_count = deposit_record.confirmation_count or 0
    last_check_block = deposit_record.last_confirmed_block or 0

    while (confirmation_count < required_confirmations):
        try:
            current_block = fetch_current_block(chain)
            target_block = last_check_block + 1

            # 检查目标块是否存在
            if (verify_block_exists(chain, target_block)):
                confirmation_count += 1
                db.update_deposit(tx_hash, {
                    confirmation_count: confirmation_count,
                    last_confirmed_block: target_block,
                    last_check_time: now()
                })
                last_check_block = target_block

            # 检查超时（充值完成应在 ~60s 内）
            if (now() - deposit_record.created_time > 300s):
                # 1. 检查交易是否真的在链上
                tx_on_chain = verify_tx_on_chain(chain, tx_hash)
                if (not tx_on_chain):
                    # 交易失败或被替换 → 通知用户
                    mark_deposit_failed(tx_hash, reason="tx_not_on_chain")
                    break

                # 2. 交易在链上但确认缓慢 → 继续等待，但发告警
                alert(f"充值确认缓慢: {tx_hash}, 已{confirmation_count}/19", severity=P2)

        except ConnectionError:
            # 网络中断 → 记录进度 → 重连后继续
            log_info(f"监听中断, 已确认{confirmation_count}/19, 最后块: {last_check_block}")
            sleep(5s)  # 退避重试
            continue

        sleep(0.5s)  # 轮询间隔

    # 确认完成
    db.update_deposit(tx_hash, {
        status: CONFIRMED,
        confirmation_time: now(),
        user_balance: add_balance(user_id, amount)
    })
    notify_user(user_id, f"充值${amount}已到账")

# 关键: 幂等性
# 所有DB更新使用 tx_hash 作为唯一键
# 即使监听程序重复检查同一个块，也不会重复入账
```

**输出**：
- 充值记录状态流转：
  ```
  PENDING (14:00:00)
  → CONFIRMING [2/19] (14:00:08)
  → CONFIRMING [12/19] (14:00:35)
  → [NETWORK_ERROR] (14:00:36)
  → CONFIRMING [12/19] (14:00:56, 重连后从上次进度继续)
  → CONFIRMING [19/19] (14:01:00)
  → CONFIRMED (14:01:00)
  ```

- 用户余额更新：
  ```
  User Balance Before: $50,000
  + Deposit: $10,000
  = User Balance After: $60,000 (14:01:05)
  ```

- 内部审计日志：
  ```
  DEPOSIT_LOG {
    tx_hash: 0xabc123
    chain: TRON
    amount: $10K
    created: 2026-04-09 14:00:00Z
    confirmed: 2026-04-09 14:01:05Z
    confirmation_time: 65s
    network_interruptions: 1
    final_confirmation_count: 19
    status: SUCCESS
  }
  ```

**监控点**：
- 未确认充值队列（按时间排序）
- 每笔充值的确认进度（X/19）
- 确认平均时间
- 网络中断事件与恢复情况
- 超时充值（>5分钟未确认）

**失败兜底**：
- 链上无此交易 → 标记充值失败，通知用户检查发送状态
- 交易被替换（RBP、加速等）→ 检查新tx_hash → 重新监听
- 网络中断超过10分钟 → 发P1告警 + 人工检查
- 充值金额与记录不符 → 区块链审计 + 人工调查

---

## SC-EC-006：提现挤兑 (大量用户同时提现)

**场景背景**：
突发事件（市场恐慌、竞争对手推广、平台小故障谣言等）引发用户大规模提现。1小时内提现请求总额$2M，但平台热钱包（立即可用资金）仅有$500K。系统需分级处理、触发紧急资金归集（从冷钱包转热钱包），防止提现完全中断。

**输入**：
- 热钱包余额：$500K
- 提现请求队列（1小时内）：
  ```
  Request 1: user_A, $50K, tier_normal
  Request 2: user_B, $30K, tier_normal
  Request 3: user_C, $200K, tier_large
  Request 4: user_D, $150K, tier_large
  Request 5: user_E, $15K, tier_normal
  Request 6: user_F, $300K, tier_xtra_large
  Request 7: user_G, $25K, tier_normal
  Request 8: user_H, $1.2M, tier_xtra_large  ← 单笔超大
  总需求: $1.97M
  ```

- 冷钱包余额（归集源）：$5M
- 热钱包转账延迟（链上操作）：15-30分钟

**决策规则**：
```
def handle_withdrawal_surge():
    total_requested = sum(all_pending_requests)
    available = hot_wallet_balance()

    if (total_requested > available):
        trigger_triage()

def trigger_triage():
    # 分级处理
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

    # 第一步：按优先级处理已有资金
    normal_requests = [r for r in queue if r.tier == NORMAL]
    large_requests = [r for r in queue if r.tier == LARGE]
    xtra_large_requests = [r for r in queue if r.tier == XTRA_LARGE]

    processed = 0
    remaining = hot_wallet_balance()

    # NORMAL优先
    for req in normal_requests[:]:
        if (remaining >= req.amount):
            execute_withdrawal(req)
            remaining -= req.amount
            normal_requests.remove(req)
        else:
            break

    # LARGE次优先
    for req in large_requests[:]:
        if (remaining >= req.amount):
            execute_withdrawal(req)
            remaining -= req.amount
            large_requests.remove(req)
        else:
            break

    # 第二步：触发紧急归集
    shortage = total_requested - hot_wallet_balance()
    if (shortage > 0):
        # 从冷钱包转入
        trigger_emergency_consolidation(shortage)
        # 预期到账：15-30分钟

    # 第三步：等待归集，继续处理剩余请求
    # 当热钱包有新资金入账时，继续处理LARGE/XTRA_LARGE
```

处理时间线示例：
```
14:00:00 - 提现请求涌入，检测到挤兑 (总$1.97M > 可用$500K)
           分级完毕：NORMAL $120K, LARGE $350K, XTRA_LARGE $1.5M

14:00:15 - 第一批处理：NORMAL全部批准 ($120K)
           热钱包剩余: $380K

14:00:30 - 第二批处理：LARGE部分批准 ($350K)
           热钱包耗尽: $30K (剩余)

14:00:35 - 触发紧急归集：从冷钱包转入 $1.5M+缓冲
           预期入账: 14:15-14:30

14:15:00 - 热钱包收到部分资金 (如 $800K)
           继续处理：LARGE剩余部分 + XTRA_LARGE部分

14:30:00 - 热钱包收到足额资金 ($1.5M+)
           所有XTRA_LARGE请求处理完毕
```

**输出**：
- 提现请求分级与处理状态：
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
        expected_completion: 14:01:00Z
      },
      LARGE: {
        count: 2,
        total: $350K,
        status: PARTIAL_APPROVED,
        approved_amount: $350K,
        expected_completion: 14:05:00Z
      },
      XTRA_LARGE: {
        count: 2,
        total: $1.5M,
        status: QUEUED,
        expected_start: 14:20:00Z (归集完成后)
      }
    }
  }
  ```

- 紧急归集指令：
  ```
  EMERGENCY_CONSOLIDATION {
    from: cold_wallet
    to: hot_wallet
    amount: $1.5M
    chain: TRON, ETH (多链)
    initiated: 14:00:35Z
    expected_arrival: 14:15-14:30Z
    status: IN_PROGRESS
  }
  ```

- 用户通知（分级）：
  ```
  [NORMAL用户] 您的提现已批准，预计5分钟内到账
  [LARGE用户] 您的提现已入队，预计30分钟内到账
  [XTRA_LARGE用户] 您的提现正在处理，由于提现量大，预计2小时内到账。
               平台正在补充流动性以加速处理。
  ```

**监控点**：
- 提现队列长度与总金额
- 各分级的待处理与已处理数
- 热钱包余额与耗尽速率
- 归集进度（链上确认）
- 用户投诉/纠纷数

**失败兜底**：
- 冷钱包转账延迟 >30分钟 → 联系运营补充热钱包（其他渠道）
- 冷钱包资金不足 → 外部融资或暂停新提现 (已申请用户继续处理)
- 热钱包耗尽未恢复 → 暂停提现，发P0告警，需人工干预
- 用户投诉处理延迟 → 自动补偿（如合理）

---

## SC-EC-007：资金费结算时HL与平台费率偏差

**场景背景**：
Hyperliquid资金费率（funding rate）每8小时结算一次。平台通过WebSocket缓存了HL的费率，并用于INTERNAL仓位的费用计算。但由于网络延迟或缓存更新不及时，平台缓存的费率（0.010%）与HL实际费率（0.012%）产生0.002%的差异。结算时需对账并处理差异。

**输入**：
- HL实际资金费率：0.012% / 8h
- 平台缓存费率：0.010% / 8h
- 差异：0.002%（平台缓存过低）

- INTERNAL仓位总额：$10M BTC多头（来自多个用户）
- 结算时间：2026-04-09 16:00:00 UTC（8小时结算周期）

- 费用计算：
  - HL实际应收：$10M * 0.012% = $1,200
  - 平台按缓存计算：$10M * 0.010% = $1,000
  - 偏差：$200（平台少收）

**决策规则**：
```
def settle_funding_rate():
    # 第一步：按缓存费率结算（正常流程）
    for position in internal_positions:
        cached_rate = get_cached_funding_rate(position.symbol)
        fee_collected = position.notional * cached_rate
        settlement_record = record_settlement(position, cached_rate, fee_collected)

    # 第二步：定期对账（每次结算后）
    actual_rate = fetch_latest_funding_rate_from_hl(symbol)

    if (abs(actual_rate - cached_rate) > tolerance):  # tolerance = 0.001%
        deviation = actual_rate - cached_rate

        # 计算偏差金额
        total_notional = sum(pos.notional for pos in internal_positions)
        deviation_amount = total_notional * deviation

        log_warning(f"费率偏差检测: {deviation}, 金额: ${deviation_amount}")

        # 分两种情况处理
        if (deviation_amount > 0):  # HL费率更高，平台少收
            # 从平台风险准备金补差
            log_action(f"平台承担偏差 ${deviation_amount}")
            transfer_from_reserve(deviation_amount)
            log_deviation(symbol, cached_rate, actual_rate, deviation_amount, "platform_absorb")

        elif (deviation_amount < 0):  # HL费率更低，平台多收
            # 返还用户（分配回各仓位）
            refund_amount = -deviation_amount
            distribute_refund_to_users(refund_amount)
            log_deviation(symbol, cached_rate, actual_rate, deviation_amount, "refund_users")

        # 更新缓存
        update_funding_rate_cache(symbol, actual_rate)

        # 告警
        alert(f"资金费率偏差 {deviation}, 已处理", severity=P2)
```

**输出**：
- 结算记录（每仓位）：
  ```
  FUNDING_SETTLEMENT {
    period: 2026-04-09 08:00-16:00 UTC
    user: user_XYZ
    symbol: BTC
    notional: $100K
    position_side: long
    rate_applied: 0.010% (cached)
    fee_charged: $10
    status: SETTLED
  }
  ```

- 事后对账发现偏差：
  ```
  FUNDING_RATE_DEVIATION_LOG {
    symbol: BTC
    cached_rate: 0.010%
    actual_rate: 0.012%
    deviation: +0.002%

    affected_notional: $10M
    deviation_amount: $200

    action: PLATFORM_ABSORB
    cost_to_platform: $200 (从风险准备金)

    timestamp: 2026-04-09 16:05:00Z
    severity: P2
  }
  ```

- 记录修正：
  - 风险准备金变更：$450K → $450K - $200 = $449,800
  - 用户仓位：无需调整（已按缓存费率结算）
  - 缓存更新：BTC费率 0.010% → 0.012%

**监控点**：
- 资金费率缓存与实际值的偏差（每次同步）
- 偏差超过阈值的事件
- 平台承担的偏差成本累计
- 费率更新延迟

**失败兜底**：
- 偏差发现过晚（多个结算周期后） → 事后多周期对账 → 分批补差
- 偏差金额过大（>$1K） → 事件升级为P1 + 人工审查
- 风险准备金不足以覆盖偏差 → 从其他资金源补充或延迟结算
- HL费率数据源不可用 → 保用缓存费率，待恢复后对账

---

## SC-EC-008：管理员误操作 (阈值设为$0)

**场景背景**：
Risk Manager通过管理后台调整路由阈值（对赌与转HL的分界点）。本应从$10K改为$15K，但误点击或手误输入了$0（零）。结果所有订单（包括$1小额订单）都被路由到INTERNAL，导致敞口快速飙升。系统需通过审计日志检测变更、自动阈值合理性校验、及时告警和自动修正。

**输入**：
- 原配置：routing_threshold = $10K
- 错误配置：routing_threshold = $0（误设）
- 修改人：risk_manager_01
- 修改时间：2026-04-09 14:30:00 UTC
- 修改原意：应改为$15K（但误操作）

- 后续订单（修改后）：
  ```
  14:30:30 Order 1: $2K → 判定: $2K <= $0 (false) → 异常逻辑
  应走HL，但由于阈值=0，可能陷入undefined行为
  ```

**决策规则**：
```
def apply_routing_threshold_change(new_threshold):
    # 第一步：基本校验
    if (new_threshold < 0):
        reject_change("Threshold cannot be negative")
        return

    if (new_threshold == 0 or new_threshold < $100):  # 下限检查
        log_warning(f"异常阈值: {new_threshold}, 小于最小值 $100")
        alert("路由阈值异常低", severity=P1)
        # 可选：拒绝此修改，或仅发告警

    # 第二步：记录变更（审计日志）
    audit_log({
        change_id: uuid(),
        timestamp: now(),
        changed_by: current_user,
        old_value: routing_threshold,
        new_value: new_threshold,
        status: PENDING_APPROVAL  # 敏感配置需审批
    })

    # 第三步：应用变更（仅在低风险时段）
    current_hour = now().hour
    if (current_hour >= 2 and current_hour <= 8):  # 交易低谷时段
        apply_threshold(new_threshold)
        log_info(f"阈值已更新: {new_threshold}")
    else:
        queue_threshold_change(new_threshold, apply_at=next_low_volume_hour)
        alert("敏感配置变更已排队，将在低交易量时段应用")

def monitor_threshold_anomaly():
    # 变更后，监控敞口变化
    if (routing_threshold == $0):
        # 所有订单都会进INTERNAL（or undefined）
        # 敞口会快速增长
        net_exposure = get_net_exposure()
        if (net_exposure > $400K):  # 预期阈值$0会导致快速增长
            alert("路由阈值异常: $0, 敞口快速增长到 ${net_exposure}", severity=P0)

            # 自动修正
            if (abs(now() - threshold_change_time) < 1hour):
                # 刚改过，可能是误操作 → 自动回滚
                revert_threshold_to_previous(reason="anomaly_detected")
                alert("已自动回滚异常阈值修改", severity=P1)
```

**输出**：
- 管理员修改操作记录：
  ```
  AUDIT_LOG {
    event_id: audit_20260409_1430_001
    timestamp: 2026-04-09 14:30:00Z
    action: THRESHOLD_UPDATE
    actor: risk_manager_01
    old_value: $10,000
    new_value: $0  ← 异常值！
    status: APPLIED
  }
  ```

- 实时告警序列：
  ```
  14:30:05 - P2告警: 路由阈值修改为异常低值 $0
  14:32:00 - P1告警: 净敞口达 $350K（正常预期$100-200K）
  14:32:30 - P0告警: 路由阈值 $0 导致敞口失控，自动回滚
  ```

- 自动修复动作：
  ```
  AUTO_RECOVERY {
    detected_anomaly: routing_threshold = $0
    trigger_time: 2026-04-09 14:32:30Z

    action: REVERT_THRESHOLD
    reverted_to: $10,000
    reason: anomaly_detected_and_net_exposure_surge

    status: SUCCESS
    new_net_exposure: $420K (平稳)
  }
  ```

- 通知：
  - Risk Manager: 「您的阈值修改 ($0) 被检测为异常并自动回滚到 $10K。请重新确认。」
  - CTO: P0告警 + 审计记录链接
  - 管理后台：修改历史显示"REVERTED"标记

**监控点**：
- 配置变更审计日志
- 变更后敞口增长速率
- 自动回滚触发次数
- 管理员操作频率与错误率

**失败兜底**：
- 自动回滚失败 → 人工回滚（Risk Manager确认后）
- 阈值设置规则缺失 → 立即实施校验和范围限制
- 敞口飙升已造成损失 → 事后复盘 + 补偿

---

## SC-EC-009：对赌执行与对冲指令竞态

**场景背景**：
L3（INTERNAL对赌）与L6（对冲引擎）是并行系统。用户下达一笔$9K INTERNAL订单同时，L6正在评估当前敞口并准备发出对冲指令。由于两个系统不是完全同步的，可能出现对赌成交后再计算对冲，导致对冲金额基于稍旧的敞口快照。系统需容忍这种短期偏差，设计最终一致性对账。

**输入**：
- 当前净敞口（L6缓存）：$95K
- 对冲阈值：$100K
- L6评估周期：每100ms
- L3对赌成交延迟：<10ms

- 时间线：
  ```
  T=0ms:    L6扫描: 净敞口$95K, < $100K阈值 → 不对冲
  T=5ms:    L3成交: 用户$9K订单 → INTERNAL敞口变为$104K
  T=10ms:   敞口变更事件入消息队列 (异步)
  T=100ms:  L6下一个周期扫描: 敞口更新为$104K > $100K → 触发50%对冲 = $52K
  T=150ms:  对冲指令发出并执行
  ```

- 竞态情景：如果L6在T=5-100ms之间没有及时更新敞口快照，可能基于$95K计算对冲$47.5K（不足）

**决策规则**：
```
# L3 (INTERNAL Execution) - 同步执行
def execute_internal_order(user_order):
    order.status = MATCHED
    user.position += order.notional
    net_exposure = recalculate_net_exposure()

    # 异步发布事件 (queue)
    publish_event({
        type: INTERNAL_ORDER_MATCHED,
        exposure_changed: order.notional,
        new_exposure: net_exposure,
        timestamp: now()
    })

    return order.status

# L6 (Hedge Engine) - 定期扫描 + 事件驱动
last_cached_exposure = $95K
last_cache_time = T=0

def hedge_engine_loop():
    while True:
        # 定期扫描
        current_exposure = get_net_exposure()  # 从数据库直接读

        if (current_exposure != last_cached_exposure):
            log_debug(f"敞口更新: {last_cached_exposure} → {current_exposure}")
            last_cached_exposure = current_exposure
            last_cache_time = now()

        # 检查对冲触发
        if (current_exposure > HEDGE_THRESHOLD):
            execute_hedge(amount=current_exposure * hedge_ratio)

        sleep(100ms)  # 扫描间隔

# 事件驱动补充（加速对冲触发）
def on_exposure_changed_event(event):
    # 如果新敞口已触发对冲，立即执行（无需等待下一个100ms周期）
    if (event.new_exposure > HEDGE_THRESHOLD):
        execute_hedge_immediately(event.new_exposure * hedge_ratio)
```

**输出**：
- 时间线（最终一致性）：
  ```
  T=0ms:    L3成交 $9K → 敞口$95K→$104K ✓
  T=50ms:   L6扫描 (可能还未感知) → 基于缓存$95K → 不对冲
  T=100ms:  L6扫描 → 读取最新敞口$104K → 触发50%对冲$52K
  T=150ms:  对冲指令发出
  T=200ms:  对冲执行完成

  最终状态：
    - INTERNAL敞口: $104K ✓ 正确
    - HL对冲: $52K ✓ 正确 (50% of $104K)
    - 缺口: 0 (L6最终读取了正确敞口)
  ```

- 监控指标：
  ```
  HEDGE_TIMING_LOG {
    order_id: order_abc
    internal_execution_time: 5ms
    exposure_change: +$9K
    exposure_before: $95K
    exposure_after: $104K

    hedge_trigger_latency: 100ms (L6下一个周期)
    hedge_execution_time: 50ms
    total_from_order_to_hedge: 150ms

    expected_hedge_amount: $52K
    actual_hedge_amount: $52K
    discrepancy: 0% ✓
  }
  ```

**监控点**：
- 对赌成交与对冲发出的时间差
- 对冲覆盖率（实际对冲 / 应对冲）
- 敞口缺口或溢出百分比
- 高频交易期间的缺口分布

**失败兜底**：
- 竞态导致对冲不足（如L6在100ms内都未更新敞口）
  → 下一个周期检测并修正 → 补充对冲
  → 时间间隔最长100ms → 可接受

- 对冲溢出（多对冲了）
  → 等待下一个敞口变更后自动回调（减少对冲）
  → 不对用户造成损害

- 极端情况（L3/L6都失败或数据不一致）
  → 每小时全量对账，修正偏差

---

## SC-EC-010：HL API限流导致对冲延迟

**场景背景**：
平台向Hyperliquid发送对冲指令（通过L4 HL Proxy），HL服务返回429 Too Many Requests。对冲指令被拒，需重试。但重试期间，INTERNAL敞口继续裸露（未被对冲）。系统需指数退避重试，同时监控敞口风险上升。

**输入**：
- 对冲指令：BTC多头$500K（3x杠杆，占用$166.67K）
- HL API限流策略：
  - 限流阈值：100请求/分钟（平台是高频用户，可能达到）
  - 响应码：429
  - Retry-After header：通常建议1s内重试

- 重试策略：指数退避
  ```
  Attempt 1: T=0ms    (失败: 429)
  Attempt 2: T=100ms  (失败: 429)
  Attempt 3: T=300ms  (失败: 429)
  Attempt 4: T=700ms  (失败: 429)
  Attempt 5: T=1500ms (成功: 200)
  ```

- 敞口暴露时间：1.5秒（从对冲决定到最终执行）

**决策规则**：
```
def submit_hedge_order_with_retry(order):
    max_retries = 10
    base_wait = 100ms

    for attempt in range(max_retries):
        try:
            response = call_hl_api(order)
            if (response.status == 200):
                log_info(f"对冲成功, 耗时{attempt * base_wait}ms")
                return SUCCESS

            elif (response.status == 429):
                # 限流
                wait_time = base_wait * (2 ^ attempt)  # 指数退避

                if (attempt < 5):  # 前5次快速重试
                    log_warning(f"API限流, 等待{wait_time}ms后重试")
                else:  # 超过5次，升级告警
                    log_alert(f"API限流持续, 对冲延迟{attempt * wait_time}ms", severity=P1)

                if (attempt == 5):  # 超时5s
                    # 敞口已暴露5s+，升级为P0
                    publish_alert({
                        type: "HEDGE_DELAYED",
                        reason: "HL_API_RATE_LIMIT",
                        exposure: order.notional,
                        delay: wait_time,
                        severity: P0
                    })

                sleep(wait_time)
                continue

            else:
                # 其他错误（非限流）
                log_error(f"HL API错误 {response.status}")
                return FAILURE

        except Exception as e:
            log_error(f"网络异常: {e}")
            sleep(base_wait * (2 ^ attempt))

    # 10次重试全部失败
    log_alert("对冲重试10次失败，切换HL_MODE", severity=P0)
    switch_routing_mode(HL_MODE)
    return FAILURE
```

**输出**：
- 对冲重试日志：
  ```
  HEDGE_RETRY_LOG {
    order_id: hedge_order_xyz
    initial_exposure: $500K
    target_hedge: $500K

    attempts: [
      {attempt: 1, time: 0ms, response: 429, wait_next: 100ms},
      {attempt: 2, time: 100ms, response: 429, wait_next: 200ms},
      {attempt: 3, time: 300ms, response: 429, wait_next: 400ms},
      {attempt: 4, time: 700ms, response: 429, wait_next: 800ms},
      {attempt: 5, time: 1500ms, response: 200, status: SUCCESS}
    ]

    total_delay: 1500ms
    exposre_duration: 1.5s
    final_status: SUCCESS
  }
  ```

- 告警序列：
  ```
  T=300ms: P2告警 - API限流，对冲延迟
  T=1500ms: P1告警 - API限流持续，敞口暴露>1s
  ```

- 敞口风险评估：
  ```
  暴露时间: 1.5秒
  期间价格波动: 假设±0.5% → 浮亏$2.5K（可控）
  对冲完成: 敞口正常覆盖 ✓
  ```

**监控点**：
- HL API限流事件频率
- 对冲重试成功率
- 平均重试延迟
- 限流期间的敞口暴露时间累计

**失败兜底**：
- 10次重试全部失败 → 切HL_MODE + P0告警
  ```
  系统判断: HL API完全不可用
  采取行动:
    - 所有新订单→HL（让HL处理这些大额订单）
    - 现有INTERNAL敞口→停止新对赌
    - 准备人工干预
  ```

- 限流限制过严（频繁触发） → 可能需与HL协商增加配额
- 部分对冲成功、部分失败 → 事后对账补差

---

## SC-EC-011：用户同时在INTERNAL和HL持有反向仓位

**场景背景**：
用户操作完全合法的风险对冲策略：在平台INTERNAL对赌中做多BTC $5K，同时在Hyperliquid账户中做空BTC $20K。两个仓位是独立的、互相隔离的、风险也是独立计算的。系统需正确处理多仓位清算、风险计算、头寸映射。

**输入**：
- 用户ID：user_strategy_trader_001

- INTERNAL仓位：
  - Symbol: BTC
  - Side: LONG
  - Notional: $5K
  - Leverage: 1x
  - Route: INTERNAL
  - Account Equity: $5K
  - Status: OPEN

- HL仓位（交易账户）：
  - Symbol: BTC
  - Side: SHORT
  - Notional: $20K
  - Leverage: 2x
  - Route: HL
  - Account Equity: $10K
  - Status: OPEN

- BTC价格变化：$60K → $55K (-8.3%)

**决策规则**：
```
def calculate_user_risk(user_id):
    # 获取用户所有仓位（跨系统）
    internal_positions = get_positions(user_id, system=INTERNAL)
    hl_positions = get_positions(user_id, system=HYPERLIQUID)

    # 分别计算每个系统的风险
    for pos in internal_positions:
        pos.floating_pnl = calculate_pnl(pos, system=INTERNAL)
        pos.margin_ratio = calculate_margin_ratio(pos, system=INTERNAL)
        # INTERNAL: 独立清算 (isolated per position in internal book)

    for pos in hl_positions:
        pos.floating_pnl = calculate_pnl(pos, system=HYPERLIQUID)
        # HL: 全仓清算 (cross-margin per HL account)
        # 需要聚合所有HL仓位的保证金比

    # 汇总（但不合并）
    total_pnl = sum_all_floating_pnl(internal + hl)

    return {
        internal: {positions: [...], total_pnl: X},
        hl: {positions: [...], total_pnl: Y},
        overall_pnl: X + Y
    }

def handle_liquidation_for_user(user_id, price_change):
    # 清算分别处理

    # 1. INTERNAL清算检查
    internal_pos = get_position(user_id, "BTC", system=INTERNAL)
    if (internal_pos.equity <= internal_pos.maintenance_margin):
        # INTERNAL清算
        liquidate_position(internal_pos, system=INTERNAL)

    # 2. HL清算检查（独立）
    hl_positions_user = get_all_positions(user_id, system=HYPERLIQUID)
    hl_account_equity = sum(p.equity for p in hl_positions_user)
    hl_total_maintenance = sum(p.maintenance_margin for p in hl_positions_user)

    if (hl_account_equity <= hl_total_maintenance):
        # HL全仓清算（可能多个仓位）
        cascade_liquidation(user_id, system=HYPERLIQUID)

# 本场景时间线：
初始状态:
  INTERNAL BTC多 $5K: 浮盈$0, 权益$5K, 维保$2.5K (40% ratio) ✓
  HL BTC空 $20K (2x): 浮盈+$1.67K, 权益$11.67K ✓

BTC跌8.3% (-$5K):
  INTERNAL BTC多 $5K: 浮亏-$416, 权益$4.58K, 维保$2.5K (183% ratio) ✓ 安全
  HL BTC空 $20K: 浮盈更多 +$1.67K (做空获利), 权益保持或增加 ✓ 安全

结论: 两个仓位都安全，不触发清算 ✓
反向对冲策略有效 ✓
```

**输出**：
- 用户持仓聚合视图：
  ```
  USER_POSITIONS_AGGREGATE {
    user_id: user_strategy_trader_001

    internal_system: {
      positions: [
        {symbol: BTC, side: LONG, notional: $5K, floating_pnl: -$416, status: SAFE}
      ],
      total_equity: $4.58K,
      total_maintenance: $2.5K,
      margin_ratio: 183%
    },

    hl_system: {
      positions: [
        {symbol: BTC, side: SHORT, notional: $20K, floating_pnl: +$1.67K, status: SAFE}
      ],
      account_equity: $11.67K,
      total_maintenance: $10K,
      margin_ratio: 117%
    },

    overall: {
      total_floating_pnl: -$416 + $1.67K = $1.25K (净盈利，对冲有效)
      combined_margin_ratio: N/A (不聚合，各自独立)
    }
  }
  ```

- 风险监控（分系统）：
  ```
  INTERNAL清算阈值: 权益 < 维保 (183% > 100% ✓)
  HL清算阈值: 账户权益 < 聚合维保 (117% > 100% ✓)
  ```

**监控点**：
- 用户级别的多系统持仓一致性
- 各系统独立的margin ratio
- 对冲策略的净PnL贡献
- 两系统持仓的反向相关性

**失败兜底**：
- 数据源不一致（INTERNAL记录有仓位，HL记录没有） → 事后对账 + 修复
- 一个系统清算但另一系统未同步收到 → 手动核实 + 补偿
- 用户投诉反向仓位被平仓不公 → 复盘清算逻辑，若有错误则补偿

---

## SC-EC-012：系统重启后状态恢复

**场景背景**：
平台系统需进行计划外或紧急重启（版本升级、故障恢复等）。重启期间所有内存数据丢失。系统需从持久化存储（数据库）重建所有状态：用户仓位、待成交限价单队列、净敞口聚合、HL对冲仓位映射。重启过程中HL连接断开，重启后需重新连接并全量同步。

**输入**：
- 重启原因：高优先级Bug修复，需服务重启
- 重启前系统状态：
  ```
  用户仓位 (INTERNAL):
    - user_A: BTC多 $50K, 状态OPEN
    - user_B: ETH多 $30K, 状态OPEN
    - user_C: SOL多 $20K, 状态OPEN
    总INTERNAL敞口: $100K

  HL对冲账户:
    - BTC多 $50K (1x)
    - ETH多 $15K (1x)
    总对冲: $65K

  待成交限价单队列:
    - order_1: user_D, BTC买入 @58000, $10K, 待成
    - order_2: user_E, ETH买入 @2000, $15K, 待成

  系统运行时长: 30天无故障
  ```

- 重启耗时预期：3-5分钟
- HL同步预期：<10秒

**决策规则**：
```
def system_startup_recovery():
    log_info("系统重启，开始状态恢复...")

    # 第一步：从DB加载所有状态
    log_info("Step 1: 从数据库加载状态...")
    users = db.load_all_users()
    internal_positions = db.load_internal_positions()
    hl_hedge_positions = db.load_hl_hedge_positions()
    pending_limit_orders = db.load_pending_orders()

    log_info(f"  已加载 {len(users)} 个用户")
    log_info(f"  已加载 {len(internal_positions)} 个INTERNAL仓位")
    log_info(f"  已加载 {len(pending_limit_orders)} 个待成交订单")

    # 第二步：重建内存快照
    log_info("Step 2: 重建内存快照...")
    net_exposure = calculate_net_exposure(internal_positions)
    hedge_mapping = build_hedge_mapping(hl_hedge_positions)
    order_queue = rebuild_order_queue(pending_limit_orders)

    log_info(f"  净敞口: ${net_exposure}")
    log_info(f"  对冲映射: {len(hedge_mapping)} 个币种")
    log_info(f"  订单队列: {len(order_queue)} 笔待成")

    # 第三步：重新连接HL WebSocket
    log_info("Step 3: 重新连接Hyperliquid...")
    try:
        ws.connect(hl_endpoint)
        ws_connected = True
        log_info("  WS连接成功")
    except Exception as e:
        log_error(f"  WS连接失败: {e}")
        ws_connected = False
        trigger_fallback_mode()  # 降级到REST轮询

    # 第四步：全量同步HL状态
    if (ws_connected):
        log_info("Step 4: 全量同步HL状态...")
        try:
            hl_account_state = fetch_account_snapshot()
            hl_open_orders = fetch_open_orders()

            # 对账
            discrepancies = reconcile_hl_state(hl_account_state, hedge_mapping)

            if (discrepancies):
                log_warning(f"  发现 {len(discrepancies)} 个不一致")
                for disc in discrepancies:
                    log_warning(f"    {disc}")
                    # 修正本地记录
                    update_local_state(disc)

            log_info("  同步完成")
        except Exception as e:
            log_error(f"  同步失败: {e}")
            trigger_alert("HL同步失败，进入受限模式", severity=P1)

    # 第五步：数据一致性检查
    log_info("Step 5: 数据一致性检查...")
    check_results = run_consistency_checks(
        internal_positions,
        hl_hedge_positions,
        order_queue
    )

    if (check_results.passed):
        log_info("  ✓ 所有检查通过")
    else:
        log_error(f"  ✗ {len(check_results.failures)} 项检查失败")
        for failure in check_results.failures:
            log_error(f"    {failure}")
        trigger_alert("数据一致性检查失败，暂停交易", severity=P0)
        skip_resumption = True

    # 第六步：恢复正常运营
    if (not skip_resumption):
        log_info("Step 6: 恢复正常运营...")

        # 恢复订单簿服务
        enable_new_orders()

        # 恢复清算检查
        start_liquidation_checks()

        # 恢复对冲引擎
        start_hedge_engine()

        # 恢复API服务
        start_api_service()

        log_info("✓ 系统已恢复正常")
        publish_notification({
            type: "SYSTEM_RECOVERED",
            timestamp: now(),
            downtime: calculate_downtime()
        })
    else:
        log_info("× 数据一致性问题，系统保持暂停状态")
        publish_notification({
            type: "SYSTEM_RECOVERY_FAILED",
            reason: "data_consistency_check_failed",
            manual_intervention_required: True
        })

def consistency_checks():
    checks = [
        # 1. 仓位数量一致性
        ("position_count", lambda: len(internal_positions) == db.count_positions()),

        # 2. 敞口聚合一致性
        ("net_exposure", lambda: abs(calculate_net_exposure() - db.get_cached_exposure()) < tolerance),

        # 3. 对冲映射完整性
        ("hedge_mapping", lambda: all(check_hedge_coverage())),

        # 4. 订单队列无重复
        ("order_queue_unique", lambda: len(set(o.id for o in order_queue)) == len(order_queue)),

        # 5. 用户余额非负
        ("user_balances", lambda: all(u.balance >= 0 for u in users)),
    ]

    return run_checks(checks)
```

**输出**：
- 重启日志（按时间顺序）：
  ```
  [2026-04-09 14:00:00] INFO: 系统启动...
  [2026-04-09 14:00:02] INFO: Step 1: 数据库加载完成
                             已加载 450 用户, 1,200 仓位, 85 待成交单
  [2026-04-09 14:00:05] INFO: Step 2: 内存快照重建完成
                             净敞口: $1.23M, 对冲映射: 8币种, 队列: 85笔
  [2026-04-09 14:00:10] INFO: Step 3: WS连接Hyperliquid... 成功
  [2026-04-09 14:00:15] INFO: Step 4: 全量同步... 完成 (无不一致)
  [2026-04-09 14:00:18] INFO: Step 5: 一致性检查... ✓ 通过
  [2026-04-09 14:00:20] INFO: Step 6: 恢复运营...
  [2026-04-09 14:00:22] INFO: ✓ 系统已恢复正常 (恢复耗时: 22秒)
  ```

- 恢复统计：
  ```
  SYSTEM_RECOVERY_REPORT {
    recovery_start: 2026-04-09 14:00:00Z
    recovery_complete: 2026-04-09 14:00:22Z
    total_recovery_time: 22s

    state_loaded: {
      users: 450
      internal_positions: 1,200
      hl_hedge_positions: 145
      pending_orders: 85
    },

    ws_reconnect_time: 8s
    hl_sync_time: 5s
    consistency_checks: 5/5 passed

    final_status: OPERATIONAL
  }
  ```

- 用户通知：
  ```
  系统已恢复 (2026-04-09 14:00:22Z)
  停机时长: 22秒

  您的仓位：
  - BTC多 $50K ✓
  - 待成交订单 ✓
  - 账户余额 ✓ (无变化)

  可立即继续交易。如有问题请联系支持。
  ```

**监控点**：
- 重启耗时分布（各阶段）
- 数据库加载速度（按数据量）
- WS重连成功率与延迟
- 一致性检查失败率与原因
- 首笔交易执行延迟（恢复后）

**失败兜底**：
- 数据库无法连接 → 尝试备份DB → 失败则等待DBA
- WS连接失败 → 降级REST轮询（延迟增加但功能可用）
- 一致性检查失败 → 暂停交易 + P0告警 + 人工审查
- 部分仓位加载失败 → 继续加载其他仓位 + 标记问题记录 + 事后修复
- 重启超时（>5分钟） → 可能需重新启动，检查底层问题

---

## 附录：异常场景触发条件速查表

| 场景ID | 触发条件 | 严重等级 | 恢复时间 |
|--------|--------|--------|--------|
| SC-EC-001 | WS断连>60s | P1 | 1-2min (重连) |
| SC-EC-002 | HL交易账户保证金率<300% | P0 | 10-30min (补资) |
| SC-EC-003 | 1小时波动>5% | P1 | <5min (模式切换) |
| SC-EC-004 | 账户权益≤维保额 | P0 | <10s (级联清算) |
| SC-EC-005 | 充值确认超时>5min | P2 | 可变 (链上确认) |
| SC-EC-006 | 1小时提现>热钱包余额 | P1 | 30min (归集) |
| SC-EC-007 | 费率偏差>0.001% | P2 | <1s (检测+修正) |
| SC-EC-008 | 管理员配置异常 | P1 | <1min (自动回滚) |
| SC-EC-009 | L3/L6并行执行 | P3 | 100ms (最终一致) |
| SC-EC-010 | HL API 429 | P1 | 1.5s (重试指数退避) |
| SC-EC-011 | 多系统反向仓位 | P3 | 无风险 (独立清算) |
| SC-EC-012 | 系统重启 | P0 | 20-30s (恢复) |

---

**文档完毕**
