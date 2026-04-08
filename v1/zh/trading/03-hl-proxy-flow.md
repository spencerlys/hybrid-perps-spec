---
doc_id: trading-zh-hl-proxy-flow
title: HL 代理执行流程
tags: [hyperliquid, proxy, fill-price, drift, position-mapping, L4]
version: 1.0
lang: zh
updated: 2026-04-08
关联PRD: [prd/05-hl-execution.md, prd/09-settlement.md]
---

# HL 代理执行流程

## 开仓完整流程

```
L2 路由决策 → HYPERLIQUID
      │
      ▼
[L4] 预计算保证金（按当前 mark_price）
     └─ 临时冻结用户保证金（可用余额预扣）
      │
      ▼
[L4] 构建 HL 开仓指令
     ├─ 方向（BUY / SELL）
     ├─ 数量（按 HL szDecimals 规则取整）
     ├─ 类型（市价 / 限价）
     └─ 平台 Agent Key 签名
      │
      ▼
[L4] 向 HL 发送开仓请求（目标 < 50ms）
      │
      ▼
[L4] 等待 HL 回执
     ├─ fill_price  成交价
     ├─ filled_size 成交量
     └─ fee         手续费（含 builder fee）
      │
      ▼
[L4] 价格校正（Fill Price Correction）
     ├─ entry_price = fill_price（替换预估值）
     ├─ 重新计算实际保证金冻结量
     └─ 差额释放或补扣（available_balance 调整）
      │
      ▼
[L4] 创建 HYPERLIQUID 仓位记录
     ├─ position_source = HYPERLIQUID
     ├─ entry_price = fill_price（来自 HL 回执）
     └─ status = OPEN
      │
      ▼
[L4] 写入 fills 表（source = HL，含 hl_fill_price 字段）
      │
      ▼
[L4] 更新 HL 虚拟仓位映射（用户份额 → HL 合并仓位）
      │
      ▼
[WS] 推送成交通知
```

## 平仓完整流程

```
用户发起平仓 HYPERLIQUID 仓位
      │
      ▼
[L4] 构建 HL 平仓指令（市价平仓）
     └─ 数量 = 用户虚拟仓位 size
      │
      ▼
[L4] 向 HL 发送平仓请求
      │
      ▼
[L4] 等待 HL 回执（close_price, filled_size, fee）
      │
      ▼
[L8] 按 close_price 计算已实现 PnL
     ├─ 多头：pnl = (close_price - entry_price) × size
     └─ 空头：pnl = (entry_price - close_price) × size
      │
      ▼
[L8] 偏差计算与记录
     ├─ 偏差 = HL 回执 PnL - 平台 预计算 PnL
     ├─ 偏差 > $10 → 写 deviation_logs
     ├─ 偏差率 > 1% → Slack 告警
     └─ 偏差率 > 5% → P0 + 暂停该币种 HL 路由
      │
      ▼
[L8] 偏差兜底（若 HL 实际亏损 > 平台 计算）
     └─ 差额从风险准备金扣除，用户按 平台 计算结果结算
      │
      ▼
[L4] 释放用户保证金 → position.status = CLOSED
      │
      ▼
[L4] 更新 HL 虚拟仓位映射（减去对应 size）
      │
      ▼
[WS] 推送平仓通知
```

## 部分成交处理

HL 市价单可能分多笔成交（大单常见）：

```
Fill 1：price=$100,100，size=0.3
Fill 2：price=$100,050，size=0.5
Fill 3：price=$100,000，size=0.2

加权均价 entry_price：
= (100,100×0.3 + 100,050×0.5 + 100,000×0.2) / 1.0
= $100,055
```

## HL 清算触发流程

```
[L5] 平台 清算引擎判断用户需要清算（含 HYPERLIQUID 仓位）
      │
      ▼
[L4] 向 HL 发送市价平仓指令
     └─ 数量 = 用户虚拟仓位对应 size
      │
      ▼
[L4] 等待 HL 回执 close_price
      │
      ▼
[L8] 按 close_price 完成清算结算
     └─ 用户保证金归零
      │
      ▼
[L4] position.status = LIQUIDATED
```

## HL 一致性定期校验（每 5 分钟）

```
Σ(所有用户 HYPERLIQUID 虚拟仓位 BTC size)
  vs HL 账户实际 BTC 合并仓位 size

偏差 > 0.01% → 记录 reconciliation_logs + 告警
偏差 > 0.1%  → P0 紧急 + 暂停 HL 新开仓 + 人工审查
```
