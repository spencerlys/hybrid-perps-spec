---
doc_id: prd-zh-settlement
title: L8 结算与对账层 — PnL 公式、资金费结算、偏差兜底、三方对账
tags: [settlement, pnl, funding-rate, reconciliation, deviation, L8]
version: 1.0
lang: zh
updated: 2026-04-08
phase: Phase 3
---

# L8 结算与对账层

## PnL 计算公式

### 未实现 PnL（实时计算）

```
多头未实现 PnL = (mark_price - entry_price) × size
空头未实现 PnL = (entry_price - mark_price) × size
```

- `mark_price`：来自 HL 实时推送
- `entry_price`：**来自 HL 成交回执 fill_price**（非预估值）
- 适用于 INTERNAL 和 HYPERLIQUID 两种仓位

### 已实现 PnL（平仓/清算时结算）

**正常平仓：**
```
已实现 PnL = (close_price - entry_price) × closed_size    # 多头
已实现 PnL = (entry_price - close_price) × closed_size    # 空头
```
- `close_price`：来自 HL 平仓回执（HYPERLIQUID 仓位）或内部成交价（INTERNAL 仓位）

**部分平仓：**
```
剩余仓位 entry_price 不变（HL 行为）
已实现部分 PnL = (close_price - entry_price) × 平仓数量
```

**加仓后的均价：**
```
新 entry_price = (旧 entry_price × 旧 size + fill_price × 新 size) / (旧 size + 新 size)
```

### 手续费处理

```
手续费 = notional_value × fee_rate
```
- 开仓和平仓时各扣一次
- 从**可用余额**扣除（不影响 PnL 本身）
- HL 仓位手续费：从 HL 回执中读取实际扣除金额（含 builder fee）

## 资金费结算（完整公式）

### 计算公式

```
资金费支付额 = position_size × mark_price × funding_rate
```

- `funding_rate` > 0 → 多头付空头
- `funding_rate` < 0 → 空头付多头

### 结算规则

| 规则 | 说明 |
|------|------|
| 结算周期 | 每 8 小时（UTC 00:00、08:00、16:00） |
| 中途开仓 | 等到下一个结算时间点，不按比例计算 |
| 结算时机 | 结算时间点扫描所有持仓中仓位 |
| 精度 | 与 HL 对齐，使用 HL 公布的 funding_rate 原始值 |

### INTERNAL vs HYPERLIQUID 仓位的资金费处理

**INTERNAL 仓位：**
```
用户支付 → 平台账户（平台是对手方）
平台支付 → 用户账户（平台是对手方）
直接在内部账本扣减/增加
```

**HYPERLIQUID 仓位：**
```
HL 实际结算（从平台 HL 账户扣减/增加）
平台 复刻相同金额记到用户账上：
  用户应付资金费 → 从用户可用余额扣除
  用户应收资金费 → 增加用户可用余额
```

## 结算操作汇总

| 操作 | 结算逻辑 |
|------|---------|
| INTERNAL 开仓 | 冻结保证金（逐仓）或占用全仓额度 |
| INTERNAL 平仓 | 释放保证金 + 计算已实现 PnL |
| INTERNAL 清算 | 保证金归零 → 80% 平台利润 + 20% 准备金 |
| HL 开仓 | 冻结用户 平台 保证金 + 平台 HL 账户开仓 |
| HL 平仓 | 向 HL 发平仓 → 等回执 close_price → 结算 PnL → 释放保证金 |
| HL 清算 | 平台 判断 → 发平仓 → 等回执 → 用户保证金归零 |
| 资金费 | 每 8h 结算所有持仓仓位（INTERNAL + HL 复刻） |
| 手续费 | 开仓/平仓时从可用余额扣除 |

## 偏差兜底结算

仅适用于 HYPERLIQUID 仓位平仓/清算时：

```
偏差 = HL 回执实际 PnL - 平台 预计算 PnL

if 偏差 > 0（HL 实际亏损更多）：
  → 差额从风险准备金扣除
  → 用户按 平台 预计算结果结算
  → 记录 deviation_log

if 偏差 < 0（HL 实际亏损更少）：
  → 差额进入平台利润（或退还用户，业务策略）
  → 记录 deviation_log
```

## 对账引擎

### 三方对账

| 维度 | 公式 | 频率 |
|------|------|------|
| 用户资产 | `Σ(用户余额) + Σ(仓位保证金) + Σ(未实现PnL)` = 平台用户总资产 | 实时 |
| 平台对赌 | `净盈亏 = Σ(用户亏损) - Σ(用户盈利)`（仅 INTERNAL） | 每小时 |
| HL 账户 | `HL 总资产 ≈ 运营注入资金 + Σ(HL 仓位盈亏) + Σ(对冲盈亏)` | 每 5 分钟 |
| 各链热钱包 | `热钱包余额 ≥ 日提现需求` | 实时 |
| 计算偏差累计 | `日累计偏差 = Σ(单笔偏差)`（HYPERLIQUID 仓位） | 每日汇总 |

### 告警阈值

| 维度 | 告警 | 紧急 |
|------|------|------|
| 用户资产偏差 | > 0.01% | > 0.1% |
| HL 仓位规模偏差 | > 0.01% | > 0.1% |
| 单笔计算偏差率 | > 1% | > 5% |
| 日累计计算偏差 | > $1,000 | > $5,000 |

触发紧急告警 → 自动暂停相关操作 + P0 告警 + 5 分钟内响应
