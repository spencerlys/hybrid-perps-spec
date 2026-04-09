---
doc_id: prd-zh-hl-execution
title: L4 HL 代理执行层 — 账户架构、仓位映射、回执校正、偏差监控
tags: [hyperliquid, execution, proxy, position-mapping, fill-price, drift, L4]
version: 1.0
lang: zh
updated: 2026-04-08
phase: Phase 2
---

# L4 HL 代理执行层

## HL 平台账户架构

### 为什么只能用一个统一账户

Hyperliquid 限制：**同一账户在同一币种同一方向只能有一个仓位。** 多个用户的 BTC 多单会自动合并成一个 BTC 多头仓位。

| 设计决策 | 选择 | 原因 |
|---------|------|------|
| 账户数量 | 平台统一账户 | HL 不支持同币种多仓位 |
| HL 端保证金模式 | 全仓（Cross Margin） | 仓位合并，逐仓无意义；全仓资金效率高 |
| HL 端清算 | 永远不让 HL 触发清算 | 所有清算由 平台 控制 |

### HL 账户保证金率管理规则

| 指标 | 安全线 | 告警线 | 紧急线 |
|------|--------|--------|--------|
| HL 账户保证金率 | > 500% | < 300% | < 150% |
| 动作 | 正常 | 从资金池补充 | 暂停 HL 新开仓 + 紧急补资 |

## HL 虚拟仓位映射

平台 为每个用户维护独立的虚拟仓位记录（HL 端实际是合并的）：

| 字段 | 说明 |
|------|------|
| position_id | 平台 内部仓位 ID |
| user_id | 用户 |
| symbol | 合约品种 |
| direction | 用户方向 |
| size | 持仓数量 |
| notional_value | 名义价值 |
| entry_price | 用户实际成交价（**来自 HL 回执 fill_price**） |
| leverage | 用户选择的杠杆 |
| margin_mode | `ISOLATED` / `CROSS` |
| isolated_margin | 逐仓保证金 |
| position_source | 固定为 `HYPERLIQUID` |
| status | `OPEN` / `CLOSED` / `LIQUIDATED` |

## HL 回执价格校正

### 核心原则

**平台 不依赖自行计算的价格作为仓位最终记录，而是用 HL 返回的实际成交价更新内部数据。**

这是消除开仓/平仓阶段计算偏差的关键设计。

### 开仓校正流程

```
1. 平台 按当前 mark_price 预计算 → 冻结保证金
2. 向 HL 发送开仓订单
3. HL 返回成交回执：fill_price, filled_size, fee
4. 用 fill_price 替换内部仓位的 entry_price
5. 用 fill_price 重新计算实际保证金冻结量
6. 差额部分释放或补扣
```

### 平仓校正流程

```
1. 平台 判断应平仓 → 向 HL 发送市价平仓
2. HL 返回平仓回执：close_price, filled_size, fee
3. 已实现 PnL 按 close_price 计算（而非预估价）
4. 最终保证金释放按 close_price 结果执行
```

### 部分成交处理

HL 市价单可能分多笔成交：
- 记录每笔成交的 fill_price 和 size
- 最终 entry_price = Σ(fill_price × size) / Σ(size)（加权均价）

## 计算偏差监控

### 偏差告警阈值

| 级别 | 条件 | 动作 |
|------|------|------|
| 记录 | 单笔偏差 > $10 | 写日志 |
| 告警 | 单笔偏差率 > 1% | Slack 告警 |
| 紧急 | 单笔偏差率 > 5% | 暂停该币种路由到 HL + P0 告警 |
| 审查 | 日累计偏差 > $1,000 | 风控人工审查 |

### 偏差兜底

```
HL 实际亏损 > 平台 计算亏损：
  → 差额 = HL_actual_loss - platform_calculated_loss
  → 从风险准备金扣除差额
  → 用户按 平台 计算结果结算（用户无感知）

HL 实际亏损 < 平台 计算亏损：
  → 差额进入平台利润（或退还用户，业务决策）
```

## 大额订单拆单策略（Phase 4）

当 HYPERLIQUID 订单名义价值超过拆单阈值时，应自动拆分以控制滑点和市场冲击。

### 拆单触发条件

```
名义价值 > 拆单阈值（默认 $500,000）→ 触发自动拆单
拆单阈值按币种可配置（流动性差的币种阈值更低）
```

### 拆单策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| 均分拆单 | 等额拆为 N 笔，间隔提交 | 默认策略 |
| TWAP | 按时间加权均匀分布到指定时间段 | 大额非紧急订单 |

### 拆单执行流程

```
原始订单 $2M BTC
  │
  ▼
拆为 4 笔 × $500K
  │
  ▼
逐笔向 HL 提交（间隔可配，默认 2 秒）
  │
  ▼
每笔回执记录 fill_price / filled_size
  │
  ▼
全部完成后汇总：
  → 加权均价 = Σ(fill_price × size) / Σ(size)
  → 统一回执给用户（用户无感知拆单）
```

### MVP 阶段

Phase 2–3 暂不实现拆单，大额订单整笔提交 HL。拆单能力预留到 Phase 4。

## HL 一致性校验

定期每 5 分钟执行：

```
Σ(用户 HL 虚拟仓位 size) == HL 账户实际仓位 size

偏差 > 0.01% → 告警
偏差 > 0.1%  → P0 紧急（暂停 HL 新开仓）
```
