---
doc_id: trading-zh-internal-flow
title: 内部对赌执行流程
tags: [internal, b-book, flow, margin, pnl, L3]
version: 1.0
lang: zh
updated: 2026-04-08
关联PRD: [prd/04-internal-execution.md, prd/06-margin-liquidation.md, prd/07-risk-management.md]
---

# 内部对赌执行流程

## 开仓完整流程

```
L2 路由决策 → INTERNAL
      │
      ▼
[L3] 获取当前 HL mark_price / best bid-ask
      │
      ▼
[L3] 保证金预计算
     ├─ 逐仓：initial_margin = notional / leverage
     └─ 全仓：检查账户权益是否足够（权益 - 新初始保证金 > 维保需求）
      │
      ▼
[L3] 冻结用户保证金
     ├─ 逐仓：wallets.frozen_margin += initial_margin
     └─ 全仓：cross_margin_used += initial_margin
      │
      ▼
[L3] 创建 INTERNAL 仓位记录
     ├─ position_source = INTERNAL
     ├─ entry_price = 当前 taker price（HL best bid 或 ask）
     └─ status = OPEN
      │
      ▼
[L3] 创建平台对手方仓位（相反方向，相同 notional）
      │
      ▼
[L3] 写入 fills 表（source = INTERNAL）
      │
      ▼
[L6] 更新净敞口（实时聚合）
     └─ 检查是否触发对冲阈值
      │
      ▼
[WS] 推送成交通知给用户
```

## 保证金冻结逻辑

```
逐仓模式：
  frozen_amount = notional / leverage
  wallets.frozen_margin += frozen_amount
  wallets.available_balance -= frozen_amount

全仓模式：
  检查：账户权益 - 全仓维持保证金需求 ≥ 新初始保证金
  cross_margin_used += initial_margin
  （available_balance 根据全仓公式动态计算）
```

## 平仓流程

```
用户发起平仓
      │
      ▼
[L3] 获取当前 HL mark_price / best bid-ask（平仓价）
      │
      ▼
[L3] 计算已实现 PnL
     ├─ 多头：realized_pnl = (close_price - entry_price) × size
     └─ 空头：realized_pnl = (entry_price - close_price) × size
      │
      ▼
[L3] 释放冻结保证金
     └─ available_balance += frozen_margin（归还）
      │
      ▼
[L8] PnL 入账
     ├─ 盈利（pnl > 0）：available_balance += realized_pnl
     └─ 亏损（pnl < 0）：余额减少（保证金已在冻结时处理）
      │
      ▼
[L3] position.status = CLOSED
      │
      ▼
[L6] 净敞口更新（减去已平仓位）
      │
      ▼
[WS] 推送平仓通知
```

## 清算详细流程

```
[L5] 检测到 INTERNAL 仓位触发清算条件
      │
      ▼
[L3] 按 HL mark_price 内部结算（清算成交价 = mark_price）
      │
      ▼
[L8] 用户保证金归零
      │
      ▼
[L8] 客损分配
     ├─ 80% → 平台利润账户（platform_profit）
     └─ 20% → 风险准备金（risk_reserve）
      │
      ▼
[L3] position.status = LIQUIDATED
      │
      ▼
[L6] 净敞口更新（减去清算仓位）
      │
      ▼
[WS] 推送清算通知（含清算价格）
```

## 净敞口联动

```
每次 INTERNAL 仓位变化后，L6 实时更新净敞口：

BTC 净敞口 += 用户 BTC INTERNAL 多头 notional
BTC 净敞口 -= 用户 BTC INTERNAL 空头 notional

if 净敞口 ≥ $100K：
  ├─ $100K ~ $500K → L4 发起 50% 对冲指令
  ├─ $500K ~ $1M   → L4 发起 80% 对冲指令
  └─ > $1M         → 停止该币种 INTERNAL 新开仓 + L4 发起 80% 对冲
```

## 限价单内部队列

```
用户下 INTERNAL 限价单
      │
      ▼
加入内部待成交队列（按价格优先级排序）
      │
      ▼
L7 实时推送 mark_price → 扫描队列
      │
  价格触达 ──► 立即成交（按上述开仓流程）
      │
  用户取消 ──► 从队列移除，释放冻结保证金 → CANCELLED
```
