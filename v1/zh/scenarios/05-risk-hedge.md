# 风控与对冲场景

**文档版本**: v1.0
**最后更新**: 2026-04-09
**作者**: Claude Code
**状态**: 规范阶段 (Specification Phase)

---

## 场景概述

本文档详细描述了平台风险控制和对冲引擎（L6）的核心业务场景。涵盖净敞口监控、自动对冲触发、资金管理、路由模式切换、风险准备金管理、熔断机制、多币种优先级分配，以及对冲缓冲问题等8个关键场景。

每个场景基于"方案二（内部对赌 + 阈值对冲）"的双账户架构：
- **交易账户（HL Trading）**: 执行用户大额订单 (>$10K)
- **对冲账户（HL Hedge）**: 执行内部敞口对冲

---

## SC-RH-001：净敞口从零逐步增长到触发对冲

**场景背景**：
一天内用户持续做多BTC，平台通过L3（INTERNAL对赌）承接这些小额订单，内部净敞口从$0逐步增长。当累积净敞口突破$100K阈值时，L6对冲引擎触发自动对冲，在HL对冲账户开启对冲头寸。

**输入**：
- 多笔小额订单序列：Order1($3K), Order2($5K), Order3($4K), Order4($8K)
- 前置净敞口：$95K (所有INTERNAL多头)
- 新增订单：$8K BTC多头 (INTERNAL)
- 当前HL对冲账户杠杆：3x，可用净资本：$100K

**决策规则**：
```
净敞口 = 累计INTERNAL多头 - 累计INTERNAL空头

if (净敞口 > $100K):
    hedge_ratio = 50%
    hedge_amount = 净敞口 * hedge_ratio
    execute_hedge(amount=hedge_amount, direction="long", symbol="BTC")
else:
    skip_hedge()

当前净敞口 = $95K + $8K = $103K > $100K
→ hedge_amount = $103K * 50% = $51.5K
```

**输出**：
- L6生成对冲指令：`HEDGE_ORDER{symbol: BTC, side: long, notional: $51.5K, leverage: 3x}`
- L4代理执行到HL对冲账户，成交价记录
- 对冲账户持仓更新：BTC多头$51.5K (3x = $17.17K保证金占用)
- L6内部对冲映射表更新：`{symbol: BTC, internal_exposure: $103K, hedged: $51.5K, ratio: 50%}`
- 实时监控面板更新净敞口与对冲覆盖率

**监控点**：
- 净敞口实时值及增长速率
- 对冲执行延迟（目标：<50ms P99）
- 对冲账户保证金率（目标：>500%）
- 对冲成交价与实时价的偏差
- 用户订单成交与对冲成交的时间间隔

**失败兜底**：
- 对冲订单被HL拒绝（如限流）→ 指数退避重试3次（100ms, 200ms, 400ms）
- 3次重试全失败 → 发送P1告警 → 中断新INTERNAL订单接收 → 自动切换到`HL_MODE`（所有新订单→HL）
- 对冲账户保证金不足 → 触发SC-RH-002（资金补充流程）

---

## SC-RH-002：对冲账户资金不足

**场景背景**：
平台需要对冲$400K的内部敞口，但对冲账户仅有$100K可用资本。以3x杠杆只能覆盖$300K，资金缺口$100K。系统需要智能提升杠杆或触发资金补充，确保对冲能力不被资本限制。

**输入**：
- 净敞口需求：$400K BTC
- 目标对冲比例：50% → 对冲额$200K
- 对冲账户当前资本：$100K
- 当前杠杆：3x（覆盖容量$300K）
- 最高允许杠杆：5x（HL风控允许，平台风险策略设定）

**决策规则**：
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
        alert("资金不足", severity=P1)
```

在本场景中：
- `required_leverage = $200K / $100K = 2x < max_leverage(5x)` → 提升杠杆到2x（或2.5x保留缓冲）
- 执行完整$200K对冲
- 同时触发资金补充请求（入账时间：链上操作10-30分钟）

**输出**：
- 对冲账户杠杆从3x调整到2.5x（保留安全余量）
- 对冲指令执行：$200K BTC对冲
- 资金补充请求生成：`FUND_REQUEST{amount: $100K, chain: TRON, priority: HIGH}`
- 管理员收到Slack通知：「对冲账户资金缺口$100K，已申请补充，预计30分钟入账」
- 内部风控日志记录杠杆调整事件

**监控点**：
- 对冲账户杠杆变化历史
- 资本占用率（当前敞口 / 账户资本）
- 资金补充申请状态（待发起→链上处理→入账→确认）
- 每次杠杆调整的触发原因和时间戳

**失败兜底**：
- 杠杆提升申请失败（HL API返回错误） → 改为部分对冲：`partial_amount = $100K * 3x = $300K`，对冲$150K（75% of $200K）
- 继续提升杠杆到4x后重试 → 对冲$200K（4x = $50K成本）
- 如果仍失败或杠杆达上限 → 立即切换`HL_MODE`，新订单全部转HL规避敞口
- 资金补充延迟超过1小时 → 考虑人工补充或暂停对赌交易

---

## SC-RH-003：BTC急跌10%，对冲仓位浮亏

**场景背景**：
平台对冲账户持有BTC多头$500K（3x杠杆），基础资本$166.67K。突然BTC市价下跌10%，对冲仓位产生$50K浮亏，导致账户保证金率从500%跌至350%。系统需要从风险准备金紧急补充资金，维持对冲账户健康。

**输入**：
- 对冲账户持仓：BTC多头$500K @ 3x杠杆（占用保证金$166.67K）
- 账户总资本：$200K
- BTC价格变化：-10%
- 浮亏：-$50K
- 当前保证金率：`(账户权益 / 占用保证金) = ($200K - $50K) / $166.67K = 150% / 166.67K ≈ 300%`（实际计算HL定义）
- 目标保证金率下限：500%（平台风险政策）

**决策规则**：
```
account_equity = account_capital + unrealized_pnl = $200K - $50K = $150K
maintenance_margin_requirement = hedge_position / max_leverage_allowed
                               = $500K / 5x = $100K（保守估计）

margin_ratio = account_equity / maintenance_margin_requirement = $150K / $100K = 150%

if (margin_ratio < 500%):
    shortage = (target_ratio * maintenance - account_equity)
                = (5 * $100K) - $150K = $350K

    # 补充最小安全量
    supplement = min(shortage, available_reserve)
    transfer_from_reserve(supplement)

if (shortage > available_reserve):
    alert("准备金不足以覆盖浮亏", severity=P0)
    reduce_hedge_position()
```

在本场景：
- 账户权益$150K，保证金率低于安全阈值
- 从风险准备金转入$50K → 账户权益恢复$200K → 保证金率恢复500%
- 同时发P1告警给Risk Manager监控

**输出**：
- 风险准备金转账：$50K → 对冲账户（链上或内部转账，<1分钟）
- 对冲账户权益更新：$200K（包含浮亏）
- 保证金率实时监控：500% ✓ （恢复安全）
- 告警日志：`MARGIN_SHORTFALL_ALERT{account: hedge, shortage: $50K, supplement: $50K, reserve_balance_after: $XXK}`
- 发送Slack通知：「对冲账户浮亏$50K，已补充保证金」

**监控点**：
- 对冲账户权益（资本 + 浮亏）实时值
- 保证金率曲线（刷新频率：每秒）
- 浮亏触发补充的频次和金额累计
- 风险准备金余额变化（每次转账记录）
- BTC价格波动与对冲浮亏的关联性

**失败兜底**：
- 准备金不足（余额<$50K） → 触发SC-RH-005（准备金补充机制）
  - 暂停新INTERNAL订单，所有订单→HL
  - 发P0告警通知管理员补充
  - 对冲仓位分批减仓释放保证金
- 链上转账延迟（>5分钟） → 先按内部账本记录，异步确认链状态
- 仓位保证金率持续恶化（跌破150%） → 紧急减仓50%对冲头寸以释放保证金

---

## SC-RH-004：路由模式自动切换 (NORMAL → HL_MODE)

**场景背景**：
平台正常运行在NORMAL_MODE（$10K阈值），内部敞口稳定在$300K-$400K之间。突然一批大用户或机构订单涌入，使得INTERNAL净敞口快速飙升到$800K，远超最大安全容量。系统需自动检测并切换到HL_MODE，强制所有新订单路由到Hyperliquid，防止敞口失控。

**输入**：
- 当前路由模式：NORMAL_MODE（限额$10K）
- 当前净敞口：$550K（50%已对冲 = $275K）
- 新增大额用户订单序列：$100K + $150K + $120K = $370K（总计）
- 执行后净敞口：$550K + $370K = $920K
- HL_MODE切换阈值：$800K（平台风险配置）

**决策规则**：
```
def check_routing_mode():
    if (net_exposure > HL_MODE_threshold):
        if (current_mode != HL_MODE):
            switch_mode(NORMAL → HL_MODE)
            halt_internal_orders()
            notify_risk_manager()

check_point_A: net_exposure = $550K → OK, 保持NORMAL_MODE
check_point_B: net_exposure = $550K + $220K（前两笔订单） = $770K → 接近阈值，发告警
check_point_C: net_exposure = $770K + $150K = $920K > $800K → 触发自动切换

# 后续新订单全部转HL
Order_4($50K) → HL（而非INTERNAL）
Order_5($80K) → HL
```

**输出**：
- 路由模式切换：NORMAL_MODE → HL_MODE（事件记录时间戳）
- 系统广播：L2 Order Router 接收到模式切换信号
- 所有pending和后续新订单的路由规则：`amount > 0 → HL_MODE → forward to L4 (HL Proxy Execution)`
- INTERNAL对赌暂停：新订单不再进L3，现有INTERNAL仓位继续结算
- Risk Manager收到Slack通知：
  ```
  🚨 ROUTING MODE AUTO-SWITCH
  Previous: NORMAL_MODE | Current: HL_MODE
  Trigger: Net Exposure $920K > Threshold $800K
  Action: All new orders → Hyperliquid
  Time: 2026-04-09 14:32:15 UTC
  ```
- 内部告警面板更新：醒目显示当前模式和触发原因

**监控点**：
- 净敞口实时值及增长速率（每秒刷新）
- 模式切换事件历史（时间、原因、触发值）
- 模式切换后订单分流情况（新增订单全部→HL ✓）
- INTERNAL仓位清算进度（如有）
- 切换期间的用户体验（延迟、成功率）

**失败兜底**：
- 自动切换指令发送失败 → 重试3次 → 失败则发P0告警
- P0告警未被响应 → 系统自动降级：
  1. 硬拒绝所有新INTERNAL订单（返回HTTP 429）
  2. 所有订单自动转HL
  3. 等待Risk Manager手动确认和干预
- 模式切换指令已发但某些订单仍走INTERNAL → 数据一致性检查 → 事后对账

---

## SC-RH-005：风险准备金跌破$200K

**场景背景**：
平台累积了$450K风险准备金（来自用户在INTERNAL交易中的净亏损）。由于市场行情有利，连续几天用户盈利，准备金被消耗用于补充对冲浮亏和操作开支。准备金余额逐日下降，最终跌破$200K安全下限。系统需触发风险准备金补充流程，暂停或限制对赌交易。

**输入**：
- 前期准备金余额：$450K
- 日均耗用：$50K-$80K（对冲补充、浮亏覆盖）
- 当前余额：$180K（< $200K下限）
- 正常运营准备金目标：≥$500K
- 告警阈值：$300K / $200K（黄色/红色）

**决策规则**：
```
def monitor_reserve():
    if (reserve < $500K):
        severity = "YELLOW"  # 告警：建议补充
    if (reserve < $300K):
        severity = "ORANGE"  # 警告：开始限制对赌
    if (reserve < $200K):
        severity = "RED"     # 紧急：暂停对赌
        action = "halt_internal_trading"

当前 reserve = $180K < $200K → RED
→ halt_internal_trading()
→ switch_mode(NORMAL → HL_MODE / or FULL_HL)
→ notify_admin(urgency=P0)
```

**输出**：
- 风险准备金状态变更：HEALTHY → CRITICAL
- 交易限制启动：暂停所有新INTERNAL订单
  - 新订单全部转HL（或直接拒绝）
  - 现有INTERNAL仓位可继续平仓
- 管理层通知（多渠道）：
  ```
  【紧急通知】风险准备金余额 $180K < 安全下限 $200K
  - 已暂停INTERNAL对赌交易
  - 所有新订单路由Hyperliquid
  - 需立即补充准备金
  - 目标: 恢复到 $500K+
  ```
- 财务/运营部门收到补充请求：`RESERVE_REPLENISHMENT_REQ{target: $500K, current: $180K, gap: $320K}`
- 系统进入"受限运营"模式：保留基础服务，禁止高风险操作

**监控点**：
- 风险准备金余额（实时）
- 日变化趋势及速率预测
- 补充流程状态（申请→审批→转账→入账）
- 限制期间的交易流量（INTERNAL vs HL分比）
- 用户投诉/退出率（因对赌暂停导致的）

**失败兜底**：
- 补充延迟（未在1天内完成）
  - 继续维持HL_MODE直到入账
  - 发周期性P1告警（每4小时一次）
- 补充金额不足（仅到$250K，未达$500K）
  - 改为分阶段目标：先恢复$300K(ORANGE)，重启限制对赌
  - 继续申请补充到$500K
- 管理员无法完成补充（资金流动性问题）
  - 长期维持HL_MODE
  - 激进风控：限制杠杆、限额，逐步缩小INTERNAL业务

---

## SC-RH-006：日净亏损熔断触发

**场景背景**：
平台INTERNAL对赌业务每日都有净损益。正常日波动在±$20K，但某些极端行情日可能出现用户集中盈利、平台净亏损。系统设立日净亏损熔断器：当日INTERNAL净亏损达$500K时自动暂停对赌，防止极端风险。

**输入**：
- 交易日：2026-04-09（UTC 00:00 - 23:59）
- INTERNAL用户累计损益：+$520K（用户盈利 = 平台亏损）
- 熔断告警阈值：$100K
- 熔断触发阈值：$500K
- 熔断重置时间：次日00:00 UTC

**决策规则**：
```
# 每笔INTERNAL成交后，累加日净亏损
daily_loss = sum(all_internal_trades_pnl_today)

if (daily_loss < -$100K):
    log_alert("日亏损告警", severity=P2)
    notify_risk_manager()

if (daily_loss < -$500K):
    log_alert("日亏损熔断", severity=P0)
    halt_internal_trading()
    switch_mode(HL_MODE)
    update_circuit_breaker_status(TRIGGERED)

# 次日00:00 UTC自动重置
if (utc_hour == 0 and utc_minute == 0):
    reset_daily_loss_counter()
    circuit_breaker_status = RESET
    if (mode == HL_MODE_due_to_loss):
        # 自动尝试恢复NORMAL_MODE（如其他指标允许）
        try_restore_normal_mode()
```

时间轴示例：
```
09:30 - 日亏损 -$20K (OK)
12:15 - 日亏损 -$80K (OK)
14:45 - 日亏损 -$120K → 发告警 P2
16:20 - 日亏损 -$350K → 告警升级 P1
18:05 - 日亏损 -$510K > -$500K → 熔断触发 P0
        → 暂停INTERNAL对赌
        → 切HL_MODE
        → circuit_breaker = TRIGGERED
        → 通知Risk Manager + CTO

次日 2026-04-10 00:00:00 UTC:
        → daily_loss_counter = 0
        → circuit_breaker = RESET
        → 尝试恢复NORMAL_MODE（如敞口许可）
```

**输出**：
- 日净亏损实时累计值：$510K
- 熔断状态变更：ARMED → TRIGGERED
- 交易暂停指令：halt_internal_trading()
- 路由模式强制切换：NORMAL_MODE → HL_MODE（标记原因为"LOSS_CIRCUIT_BREAKER"）
- P0告警通知（多渠道）：
  ```
  🔴 CIRCUIT BREAKER TRIGGERED
  Type: Daily Loss Limit
  Current Loss: -$510K
  Threshold: -$500K
  Action: All internal trading HALTED
  Mode: HL_MODE (all orders → Hyperliquid)
  Auto-Reset: 2026-04-10 00:00:00 UTC
  ```
- 内部日志记录：`{timestamp, daily_loss, trigger_value, action, user_impact}`
- 用户API通知：后续对赌请求返回错误码（如429），提示«平台维护中»

**监控点**：
- 日净亏损实时值（每分钟聚合更新）
- 日亏损走势图（小时级）
- 告警和熔断触发事件历史
- 熔断期间的交易分流（100% → HL）
- 熔断恢复后的INTERNAL交易恢复情况

**失败兜底**：
- 熔断指令执行失败（某些订单仍走INTERNAL） → 事后对账 + 强制平仓
- 自动重置失败（次日00:00仍未重置） → 人工手动重置 + 告警
- 用户对熔断产生争议（认为损失不公） → 事件复盘，可能需人工审查

---

## SC-RH-007：多币种同时触发对冲，资金分配

**场景背景**：
平台支持多币种交易（BTC、ETH、SOL等）。在某个高波动日，多个币种同时积累了较大净敞口，都需要触发对冲。但对冲账户的可用资本有限，无法同时满足所有币种的完整对冲需求。系统需实现智能资金分配：按敞口规模优先级排序，优先覆盖大敞口，小敞口可能被切HL_MODE。

**输入**：
- 对冲账户总资本：$200K，当前杠杆：3x，可用容量：$600K
- 净敞口分布：
  - BTC 多：$600K（占总敞口 56%）
  - ETH 多：$300K（占总敞口 28%）
  - SOL 多：$150K（占总敞口 14%）
  - 总敞口：$1,050K
- 对冲比例：均为50%（按统一标准）
- 对冲需求：
  - BTC: $600K * 50% = $300K → 上调到$480K（为覆盖大头）
  - ETH: $300K * 50% = $150K
  - SOL: $150K * 50% = $75K
  - 总需求：$480K + $150K + $75K = $705K > 可用$600K

**决策规则**：
```
def allocate_hedge_across_symbols():
    total_demand = sum(symbol.exposure * hedge_ratio for symbol in symbols)
    total_capacity = account_capital * current_leverage

    if (total_demand <= total_capacity):
        # 充足，全部满足
        for symbol in symbols:
            execute_hedge(symbol, symbol.exposure * hedge_ratio)
    else:
        # 不充足，按规模优先级分配
        symbols_by_size = sort_by_exposure(symbols, desc=True)
        remaining_capacity = total_capacity

        for symbol in symbols_by_size:
            required = symbol.exposure * hedge_ratio
            if (remaining_capacity >= required):
                execute_hedge(symbol, required)
                remaining_capacity -= required
            else:
                # 无法满足，分两种处理
                if (remaining_capacity > 0):
                    execute_hedge(symbol, remaining_capacity)
                    remaining_capacity = 0
                # 剩余敞口切HL_MODE
                alert(f"{symbol} 敞口转HL_MODE")
                switch_symbol_to_hl(symbol)

本场景执行顺序：
1. BTC（$600K，最大）→ 对冲$480K（占用$160K资本 @ 3x）
   remaining = $600K - $160K = $440K

2. ETH（$300K，中等）→ 对冲$150K（占用$50K资本 @ 3x）
   remaining = $440K - $50K = $390K

3. SOL（$150K，最小）→ 需$75K
   实际可对冲：$390K / 3x = $130K（可覆盖$75K）
   对冲$75K（占用$25K资本 @ 3x）
   remaining = $390K - $25K = $365K

结论：全部币种可满足对冲 ✓
```

**输出**：
- 对冲指令序列（按优先级执行）：
  ```
  HEDGE_ORDER_1: {symbol: BTC, amount: $480K, leverage: 3x}
  HEDGE_ORDER_2: {symbol: ETH, amount: $150K, leverage: 3x}
  HEDGE_ORDER_3: {symbol: SOL, amount: $75K, leverage: 3x}
  ```
- 对冲账户持仓更新：
  - BTC多 $480K
  - ETH多 $150K
  - SOL多 $75K
  - 总占用保证金：($480K + $150K + $75K) / 3x = $235K（超容量$35K）

  **异常**：如果总占用>账户资本，则需缩减。例如实际容量应为$200K，则：
  - BTC: $400K（67%)
  - ETH: $120K（20%)
  - SOL: $60K（10%)
  - 不足部分(BTC还差$80K, ETH还差$30K) → 对应币种的INTERNAL订单转HL

- 多币种对冲映射表更新：`{BTC: {internal: $600K, hedged: $480K}, ETH: {...}, SOL: {...}}`
- 监控面板：三币种对冲覆盖率（BTC 80%, ETH 50%, SOL 50%）

**监控点**：
- 按币种的净敞口实时值
- 按币种的对冲覆盖率（目标50%）
- 对冲账户总保证金率
- 无法对冲而转HL的敞口比例及币种
- 多币种间的保证金借用/互补情况（如有）

**失败兜底**：
- 某个币种对冲订单执行失败（如HL流动性不足） → 该币种改为部分对冲 + 部分转HL
- 对冲账户突然爆仓（极端行情） → 所有币种强制减仓 50% → 对应INTERNAL敞口转HL
- 资金分配算法返回错误 → 保守方案：每币种按个别阈值判断，无法对冲则转HL

---

## SC-RH-008：用户集中平仓导致过度对冲

**场景背景**：
平台对一个大用户的BTC多头$800K（INTERNAL）进行了对冲，在HL对冲账户持有BTC多头$640K（50%对冲+10%缓冲）。该用户在1小时内突然决定全部平仓，INTERNAL敞口从$800K→$0。平台对冲账户需相应减仓从$640K→$0，但不能过快平仓（易产生滑点、市场冲击），需分批逐步减仓。

**输入**：
- 初始状态：
  - INTERNAL用户多头：$800K BTC
  - HL对冲账户多头：$640K BTC (3x杠杆，占用$213.3K资本)
  - 对冲比例：80% (= $800K * 80%)

- 用户平仓操作：
  - 时间：2026-04-09 14:00:00 UTC
  - 平仓速度：1小时内完成 (或连续30分钟)
  - 成交记录：
    - 14:10 -$150K
    - 14:20 -$200K
    - 14:35 -$250K
    - 14:50 -$200K
    - 总计-$800K，完成

- 平仓后状态：
  - INTERNAL敞口：$0
  - 对冲需求：$0 (低于$100K阈值，应完全撤除对冲)
  - 对冲账户应持仓：$0

**决策规则**：
```
def adjust_hedge_on_position_change():
    new_exposure = get_current_net_exposure()
    current_hedge = get_current_hedge_position()

    # 根据对冲阈值和比例计算目标对冲
    if (new_exposure < HEDGE_THRESHOLD):
        target_hedge = 0
    else:
        target_hedge = new_exposure * hedge_ratio

    hedge_adjustment = current_hedge - target_hedge

    if (hedge_adjustment > 0):
        # 需要减仓
        reduce_hedge(hedge_adjustment)
    elif (hedge_adjustment < 0):
        # 需要加仓
        increase_hedge(-hedge_adjustment)

# 本场景：
trigger_point_1: exposure = $800K - $150K = $650K
                target_hedge = $650K * 80% = $520K
                adjust = $640K - $520K = $120K (需减仓$120K)
                execute_reduce($120K, max_rate=$200K/min)

trigger_point_2: exposure = $450K
                target_hedge = $450K * 80% = $360K
                adjust = $520K - $360K = $160K (需减仓$160K)
                execute_reduce($160K, max_rate=$200K/min)

...最终...

trigger_point_N: exposure = $0 < $100K_threshold
                target_hedge = 0
                adjust = remaining_hedge (需完全减仓)
                execute_reduce_all(remaining_hedge, max_rate=$200K/min)
```

每次缩减限制速率：最多$200K/分钟（避免市场冲击）

**输出**：
- 对冲减仓计划生成：
  ```
  HEDGE_REDUCTION_PLAN:
  - Batch 1 (14:00-14:05): Reduce $200K
  - Batch 2 (14:05-14:10): Reduce $200K
  - Batch 3 (14:10-14:15): Reduce $240K (final)
  ```

- 对冲账户持仓逐步变化：
  ```
  时间        INTERNAL敞口   对冲账户   对冲率
  14:00       $800K         $640K    80%
  14:05       $650K         $440K    67.7%  (进度中)
  14:10       $450K         $240K    53.3%  (进度中)
  14:15       $0K           $0K      0%     (完成)
  ```

- 减仓成交记录：每批提交到HL，记录成交价、滑点、手续费
- 对冲映射表更新：BTC对冲从$640K→$0

**监控点**：
- INTERNAL敞口的变化速率（检测异常平仓）
- HL对冲账户持仓与预期敞口的偏差
- 每次减仓的执行时间、成交价、滑点
- 对冲账户保证金率变化（减仓释放保证金）
- 减仓完成所需总时间

**失败兜底**：
- 减仓滑点过大（>1%） → 降低减仓速率，拉长时间周期
  ```
  原计划: $200K/min → 降为 $100K/min
  计划重新生成，延长至8-10分钟完成
  ```

- 某笔减仓订单被拒（HL流动性不足） → 指数退避重试 + 标记延迟状态

- 减仓中途INTERNAL敞口反向增长（用户追加仓位）
  → 动态调整：重新计算目标对冲，可能暂停或反向加仓

- 减仓耗时超过30分钟 → 发告警 + 人工审查 + 可能采用市场单加速

---

## SC-RH-009：（预留）对赌执行与对冲指令竞态

此场景在"边缘与异常场景"(06-edge-cases.md) 的SC-EC-009详细描述。

---

## SC-RH-010：（预留）HL API限流导致对冲延迟

此场景在"边缘与异常场景"(06-edge-cases.md) 的SC-EC-010详细描述。

---

## 附录：关键指标与告警阈值汇总

| 指标 | 告警阈值(黄) | 触发阈值(红) | 动作 |
|------|-----------|----------|------|
| 净敞口 | $500K | $800K | 切HL_MODE |
| 对冲账户保证金率 | 300% | 150% | 补充资金 |
| 风险准备金 | $300K | $200K | 暂停对赌 |
| 日净亏损 | -$100K | -$500K | 熔断触发 |
| HL API延迟 | 200ms | 500ms | 降级处理/转HL |
| 数据同步延迟 | 1s | 5s | 告警+人工 |

---

**文档完毕**
