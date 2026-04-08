---
doc_id: prd-zh-internal-execution
title: L3 平台对赌执行层 — 内部仓位模型、对赌记账
tags: [internal, b-book, execution, position, counterparty, L3]
version: 1.0
lang: zh
updated: 2026-04-08
phase: Phase 3
---

# L3 平台对赌执行层

## 对赌模型

平台持有与用户完全相反的仓位。仅在 平台 内部记录，不在 HL 上实际开仓（除非触发对冲）。

```
用户开 BTC 多头 $5,000（INTERNAL）
  └─► 平台自动持有 BTC 空头 $5,000（对手方）
      → 仅内部记账，无链上操作
```

## 内部仓位数据模型

| 字段 | 说明 |
|------|------|
| position_id | 仓位 ID |
| user_id | 用户 |
| symbol | 合约品种（如 BTC-PERP） |
| direction | `LONG` / `SHORT` |
| size | 持仓数量 |
| notional_value | 名义价值 = size × entry_price |
| entry_price | 开仓价（HL 成交时的标记价格） |
| leverage | 杠杆倍数 |
| margin_mode | `ISOLATED` / `CROSS` |
| isolated_margin | 逐仓保证金（全仓模式为 null） |
| position_source | 固定为 `INTERNAL` |
| status | `OPEN` / `CLOSED` / `LIQUIDATED` |
| created_at | 开仓时间 |
| closed_at | 平仓/清算时间 |

## 成交价格规则

| 订单类型 | 成交价格 |
|---------|---------|
| 市价单 | HL 当前最优买/卖价（taker price） |
| 限价单 | 价格触达限价时按限价成交 |
| 标记价格 | 用 HL mark price 计算未实现 PnL 和清算价 |

## 内部仓位操作

### 开仓
1. 路由决策为 INTERNAL
2. 按 HL mark price / best bid/ask 成交
3. 生成成交记录
4. 冻结用户保证金
5. 创建仓位记录（position_source = INTERNAL）
6. 创建平台对手方仓位
7. 推送 WebSocket 成交通知

### 加仓（同方向再次开仓）
- 新仓位独立记录（各有独立 entry_price），不合并均价
- 每笔订单独立路由判断（加仓金额可能走不同路由）

### 平仓
1. 用户发起平仓（市价 / 止盈止损）
2. INTERNAL 仓位强制在内部平（不走 HL）
3. 按当前 HL mark price / best bid/ask 成交
4. 计算已实现 PnL（见 [09-settlement.md](09-settlement.md)）
5. 释放保证金
6. 更新仓位状态为 CLOSED

### 清算（强制平仓）
由 L5 清算引擎触发（详见 [06-margin-liquidation.md](06-margin-liquidation.md)）：
1. 清算引擎判断触发条件
2. 按 HL mark price 内部结算
3. 用户保证金归零
4. 80% → 平台利润，20% → 风险准备金
5. 仓位状态更新为 LIQUIDATED

## 对赌收入模型

```
平台净盈亏（INTERNAL 仓位）= Σ(用户亏损) - Σ(用户盈利)

单笔客损分配：
  80% → 平台利润
  20% → 风险准备金（比例可后台配置）
```

## 限价单内部订单簿

- 内部维护 INTERNAL 限价单的待成交队列
- 以 HL 实时价格作为触发基准
- 价格触达 → 立即成交 → 按以上流程处理
- 取消：从内部队列移除，释放冻结保证金
