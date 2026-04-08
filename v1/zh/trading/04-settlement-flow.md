---
doc_id: trading-zh-settlement-flow
title: 结算与清算流程
tags: [settlement, pnl, funding-rate, liquidation, reconciliation, L8, L5]
version: 1.0
lang: zh
updated: 2026-04-08
关联PRD: [prd/06-margin-liquidation.md, prd/09-settlement.md]
---

# 结算与清算流程

## 一、已实现 PnL 结算流程

```
仓位平仓（主动或清算）
      │
      ▼
确认 close_price 来源：
  ├─ INTERNAL 仓位 → HL mark_price / best bid-ask（L3 内部）
  └─ HYPERLIQUID 仓位 → HL 回执 close_price（L4）
      │
      ▼
计算已实现 PnL：
  ├─ 多头：pnl = (close_price - entry_price) × size
  └─ 空头：pnl = (entry_price - close_price) × size
      │
      ▼
手续费扣除：
  fee = notional × fee_rate
  available_balance -= fee
      │
      ▼
保证金释放：
  ├─ 逐仓：available_balance += isolated_margin
  └─ 全仓：cross_margin_used -= initial_margin
      │
      ▼
PnL 入账：
  ├─ 盈利（pnl > 0）：available_balance += pnl
  └─ 亏损（pnl < 0）：从保证金中扣除，剩余归还用户
      │
      ▼
写入 balance_logs（type = realized_pnl）
```

## 二、资金费结算流程

```
每 8 小时触发（UTC 00:00 / 08:00 / 16:00）：
      │
      ▼
[L7] 获取 HL 当期资金费率（所有品种）
      │
      ▼
[L8] 扫描所有持仓中仓位（status = OPEN）
      │
      ▼
[L8] 对每个仓位计算资金费：
     payment = size × mark_price × funding_rate
     ├─ payment > 0（多头付）：available_balance -= payment
     └─ payment < 0（空头收）：available_balance += |payment|
      │
      ▼
分仓位类型处理：
  ├─ INTERNAL 仓位：
  │   └─ 内部账本扣/增（平台对手方做相反操作）
  │
  └─ HYPERLIQUID 仓位：
      ├─ HL 实际在平台 HL 账户结算（自动）
      └─ XBIT 复刻相同金额到用户账户
      │
      ▼
写入 funding_settlements 表
      │
      ▼
[L5] 更新清算价格（资金费侵蚀保证金，清算价向不利方向移动）
      │
      ▼
[L8] 对账校验（XBIT 复刻总额 vs HL 实际结算金额）
```

## 三、清算检测与执行流程

### 清算检测（实时）

```
[L7] HL mark_price 推送（实时）
      │
      ▼
[L5] 清算引擎扫描所有受影响仓位（< 1s 目标）
      │
      ├─ 逐仓检查：
      │   (position_margin + unrealized_pnl) ≤ notional × maintenance_rate
      │   → 触发该仓位清算
      │
      └─ 全仓检查：
          account_equity ≤ cross_maintenance_requirement
          → 触发账户级清算
```

### 清算执行

```
[L5] 触发清算信号
      │
      ├─ INTERNAL 仓位 ──► [L3] 按 HL mark_price 内部结算
      │                         [L8] 保证金归零 + 客损 80/20 分配
      │                         position.status = LIQUIDATED
      │
      └─ HYPERLIQUID 仓位 ──► [L4] 向 HL 发市价平仓指令
                                   [L4] 等待 HL 回执 close_price
                                   [L8] 按 close_price 结算
                                   [L8] 偏差检测 + 兜底
                                   position.status = LIQUIDATED
      │
      ▼
[WS] 推送清算通知
[DB] liquidations 表写入记录
```

## 四、三方对账流程

```
实时：
  └─ Σ(用户余额 + 仓位保证金 + 未实现PnL) = 平台总用户资产

每 5 分钟：
  └─ HL 虚拟仓位 vs HL 实际仓位 size 对账

每小时：
  └─ 平台对赌净盈亏核算（INTERNAL 仓位汇总）

每日：
  └─ 计算偏差日累计汇总 → reconciliation_logs
  └─ 各链归集与提现对账
```

## 五、结算异常处理

| 异常场景 | 处理方式 |
|---------|---------|
| HL 回执超时 | 重试 3 次，仍无回执 → P0 告警 + 人工处理 |
| XBIT 计算 PnL vs HL 偏差 > 5% | 暂停该币种 HL 路由 + P0 告警 + 风控审查 |
| 用户资产对账偏差 > 0.1% | 立即暂停提现 + P0 告警 + 人工审计 |
| 风险准备金 < $200K | 暂停所有 INTERNAL 对赌（全量路由 HL） |
| HL 账户保证金率 < 150% | 暂停 HL 新开仓 + 紧急补资 |
