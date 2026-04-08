---
doc_id: trading-zh-order-lifecycle
title: 订单全生命周期
tags: [order, lifecycle, open, close, cancel, routing, fill]
version: 1.0
lang: zh
updated: 2026-04-08
关联PRD: [prd/02-routing.md, prd/04-internal-execution.md, prd/05-hl-execution.md]
---

# 订单全生命周期

## 订单状态机

```
PENDING_ROUTING
    │
    ▼ 路由决策（< 5ms）
ROUTED（INTERNAL / HYPERLIQUID）
    │
    ├─ 市价单 ──► FILLING ──► FILLED
    │
    └─ 限价单 ──► OPEN（等待触价）
                    │
                    ├─ 价格触达 ──► FILLING ──► FILLED
                    └─ 用户取消 ──► CANCELLED
```

## 开仓完整流程

### 步骤 1：订单接收与验证

```
用户提交订单
  ├─ 合约品种校验（是否上架）
  ├─ 杠杆范围校验（≤ maxLeverage，来自 HL meta）
  ├─ 下单量精度校验（szDecimals，来自 HL meta）
  ├─ 最小下单量校验
  └─ 可用余额校验（保证金是否足够）
```

### 步骤 2：路由决策（L2，< 5ms）

```
名义价值 = size × HL mark_price
名义价值 ≤ 路由阈值 → INTERNAL
名义价值 > 路由阈值 → HYPERLIQUID
强制 HL 条件检查（净敞口 / 延迟 / 波动率 / 手动关闭）
```

### 步骤 3A：INTERNAL 执行路径（L3，< 10ms）

```
按 HL mark price / best bid-ask 成交
冻结保证金（逐仓：frozen_margin；全仓：cross_margin_used）
创建 position（position_source = INTERNAL）
创建平台对手方仓位（内部记账）
写入 fills 表
推送 WebSocket 成交通知
↓
L6 净敞口实时更新（检查对冲阈值）
```

### 步骤 3B：HYPERLIQUID 执行路径（L4，< 50ms）

```
预计算保证金，临时冻结
向 HL 发送开仓指令
等待 HL 回执（fill_price, filled_size, fee）
用 fill_price 替换 entry_price（价格校正）
重新计算保证金，差额释放或补扣
创建 position（position_source = HYPERLIQUID）
写入 fills 表（含 hl_fill_price 字段）
推送 WebSocket 成交通知
```

## 平仓完整流程

### 用户主动平仓

```
用户发起平仓指令（全平 / 部分平）
  │
  ├─ INTERNAL 仓位 ──► L3 内部平仓
  │   ├─ 按 HL mark price / best bid-ask 成交
  │   ├─ 计算已实现 PnL
  │   ├─ 释放保证金
  │   └─ position.status = CLOSED
  │
  └─ HYPERLIQUID 仓位 ──► L4 向 HL 发平仓指令
      ├─ 等待 HL 回执 close_price
      ├─ 按 close_price 计算已实现 PnL
      ├─ 释放保证金
      └─ position.status = CLOSED
```

### 止盈止损触发

```
L7 实时推送 mark_price
平台 扫描所有 TP/SL 订单
价格触达 TP/SL 价格 → 触发平仓
执行路径同上（INTERNAL / HYPERLIQUID 分别处理）
```

## 限价单生命周期

```
用户下限价单
  │
  ▼
路由决策（INTERNAL / HYPERLIQUID）
  │
  ├─ INTERNAL ──► 加入内部限价单队列（OPEN 状态）
  │                L7 实时价格触达限价 → 立即成交 → FILLED
  │                用户取消 → 移出队列，释放保证金 → CANCELLED
  │
  └─ HYPERLIQUID ──► 向 HL 提交限价单
                       HL 成交 → 回执推送 → FILLED
                       用户取消 → 向 HL 发取消指令 → CANCELLED
```

## 订单数据模型

| 字段 | 说明 |
|------|------|
| id | 订单唯一 ID |
| user_id | 用户 |
| symbol | 合约品种 |
| side | BUY / SELL |
| size | 数量 |
| price | 限价单价格（市价单为 null） |
| route | INTERNAL / HYPERLIQUID |
| status | PENDING_ROUTING / ROUTED / FILLING / FILLED / CANCELLED |
| routing_latency_ms | 路由决策耗时（毫秒） |

## 关键性能要求

| 操作 | 目标 |
|------|------|
| 路由决策 | < 5ms P99 |
| INTERNAL 成交 | < 10ms |
| HL 订单转发 | < 50ms |
| WebSocket 成交通知 | < 100ms |
