---
doc_id: scenario-zh-order-routing
title: 订单路由场景
tags: [routing, internal, hyperliquid, threshold, force-route]
version: 1.0
lang: zh
updated: 2026-04-08
关联PRD: [prd/02-routing.md, prd/04-internal-execution.md, prd/05-hl-execution.md]
---

# 订单路由场景

---

## SC-RT-001：小额市价单 → 内部对赌

**前置条件：**
- BTC 当前 HL 标记价格 $100,000
- 路由阈值 $10,000
- 用户可用余额 $1,000，选择 10x 杠杆

**操作步骤：**
1. 用户下 BTC 多头市价单，数量 0.05 BTC
2. 路由引擎计算名义价值：0.05 × 100,000 = **$5,000 ≤ $10,000** → INTERNAL

**预期结果：**
- 按 HL 当前最优卖价成交（taker price）
- 创建 INTERNAL 仓位（position_source = INTERNAL）
- 平台自动创建对手方空头仓位（内部记账）
- 用户冻结保证金 $500（$5,000 / 10x）
- 路由日志：决策 INTERNAL，延迟 < 5ms

---

## SC-RT-002：大额市价单 → HL 代理执行

**前置条件：**
- ETH 当前 HL 标记价格 $4,000
- 路由阈值 $10,000
- 用户可用余额 $5,000，选择 5x 杠杆

**操作步骤：**
1. 用户下 ETH 多头市价单，数量 5 ETH
2. 路由引擎计算名义价值：5 × 4,000 = **$20,000 > $10,000** → HYPERLIQUID
3. 向 HL 平台账户发送开仓指令

**预期结果：**
- HL 返回成交回执：fill_price, filled_size, fee
- 用 fill_price 更新内部 HYPERLIQUID 仓位的 entry_price（价格校正）
- 用户冻结保证金 $4,000（$20,000 / 5x）
- HL 平台账户合并仓位增加 5 ETH 多头

---

## SC-RT-003：同一用户同币种两次开仓（分别路由）

**前置条件：**
- BTC 标记价 $100,000，路由阈值 $10,000
- 用户余额充足

**操作步骤：**
1. 第一次：BTC 多头，名义价值 $5,000 → **INTERNAL**
2. 第二次：BTC 多头，名义价值 $15,000 → **HYPERLIQUID**

**预期结果：**
- 用户持有两个独立 BTC 多头仓位：
  - BTC INTERNAL 仓位（$5K，独立 entry_price）
  - BTC HYPERLIQUID 仓位（$15K，独立 entry_price）
- 全仓模式：两仓位共享账户余额，分别有独立 entry_price，清算在账户层面统一计算
- 逐仓模式：两仓位各自独立清算，互不影响
- 平仓时各自跟随仓位来源：INTERNAL 内部平，HYPERLIQUID 向 HL 发指令

---

## SC-RT-004：强制路由到 HL — 净敞口达上限

**前置条件：**
- BTC 净敞口已达 $1M 上限（系统停止新增对赌）
- 用户下 BTC 多头订单，名义价值 $3,000（本应走 INTERNAL）

**操作步骤：**
1. 路由引擎检查 BTC 净敞口 → 已达限制
2. 强制路由到 HYPERLIQUID（即使 $3,000 < $10,000 阈值）

**预期结果：**
- 创建 HYPERLIQUID 仓位
- 路由日志记录强制 HL 原因：`net_exposure_limit`
- 用户无感知（相同价格、相同回执格式）

---

## SC-RT-005：强制路由到 HL — HL 通道延迟超限

**前置条件：**
- HL WebSocket 延迟 = 600ms（超过 500ms 阈值）
- 用户下 ETH 多头 $5,000 订单

**操作步骤：**
1. 路由引擎检测 HL 通道延迟 > 500ms → 暂停对赌
2. 强制路由到 HYPERLIQUID

**预期结果：**
- 路由到 HYPERLIQUID（保守策略）
- 风控告警：HL 延迟 > 500ms
- 路由日志记录强制原因：`hl_latency_exceeded`

---

## SC-RT-006：平仓路由跟随原始仓位

**前置条件：**
- 用户持有：BTC INTERNAL 多头仓位 + BTC HYPERLIQUID 多头仓位

**操作步骤：**
1. 用户发起平仓 BTC INTERNAL 仓位
2. 用户发起平仓 BTC HYPERLIQUID 仓位

**预期结果：**
- BTC INTERNAL 平仓 → L3 内部结算（不发指令到 HL）
- BTC HYPERLIQUID 平仓 → L4 向 HL 发市价平仓指令
- 禁止跨系统抵消：INTERNAL 仓位不能用来平 HYPERLIQUID 仓位

---

## SC-RT-007：连续加仓导致净敞口触发对冲

**前置条件：**
- BTC 标记价 $100,000，路由阈值 $10,000
- 当前 BTC 净敞口 = $80,000（多头方向）
- 对冲阈值：$100K 开始 50% 对冲

**操作步骤：**
1. 用户 A 下 BTC 多头 $5,000 → INTERNAL（$5K < $10K 阈值）
2. 用户 B 下 BTC 多头 $8,000 → INTERNAL
3. 用户 C 下 BTC 多头 $9,000 → INTERNAL

**预期结果：**
- 三笔订单均走 INTERNAL（各自名义价值 < $10K）
- 净敞口累计：$80K + $5K + $8K + $9K = $102K
- 第三笔成交后，L6 检测到净敞口 > $100K
- L6 触发对冲：在 HL 开 BTC 空头约 $51K（50% × $102K）
- 对冲指令走 L4 代理到 HL，独立于用户订单
- 用户全程无感知（用户看到的仍是正常 INTERNAL 仓位）

---

## SC-RT-008：多空用户自然对冲（净敞口归零）

**前置条件：**
- ETH 标记价 $4,000，路由阈值 $10,000
- 当前 ETH 净敞口 = $0

**操作步骤：**
1. 用户 A 下 ETH 多头 $8,000 → INTERNAL（平台持空头 $8K）
2. 用户 B 下 ETH 空头 $8,000 → INTERNAL（平台持多头 $8K）

**预期结果：**
- 两笔订单均走 INTERNAL
- 净敞口 = $8K - $8K = $0（多空完全抵消）
- 不触发任何对冲
- 平台赚取两笔手续费 + 未来资金费
- 这是方案二对赌模型的理想状态：零风险收手续费
