---
doc_id: scenario-zh-funding-settlement
title: 资金费结算场景
tags: [funding-rate, settlement, internal, hyperliquid, reconciliation]
version: 1.0
lang: zh
updated: 2026-04-08
关联PRD: [prd/08-market-data.md, prd/09-settlement.md]
---

# 资金费结算场景

资金费每 8 小时结算一次（UTC 00:00 / 08:00 / 16:00）。

---

## SC-FS-001：INTERNAL 仓位资金费 — 多头付空头

**前置条件：**
- 用户持有 BTC INTERNAL 多头：size=0.1 BTC，mark_price=$100,000
- 当期资金费率 funding_rate = +0.01%（正值 = 多头付空头）

**结算计算：**
```
资金费支付额 = 0.1 × 100,000 × 0.0001 = $1.00
```

**预期结果：**
- 用户账户扣除 $1.00（多头付）
- 平台对手方账户收入 $1.00
- balance_logs 写入记录，type = "funding_fee"
- funding_settlements 表写入记录
- 清算价格重新计算（资金费侵蚀保证金，清算价向不利方向移动）

---

## SC-FS-002：HYPERLIQUID 仓位资金费 — 空头收资金费

**前置条件：**
- 用户持有 ETH HYPERLIQUID 空头：size=5 ETH，mark_price=$4,000
- 当期资金费率 funding_rate = +0.005%（正值 = 多头付，空头收）

**结算计算：**
```
资金费收入 = 5 × 4,000 × 0.00005 = $1.00（空头收）
```

**预期结果：**
- HL 实际在平台 HL 账户结算 +$1.00
- XBIT 复刻相同金额记到用户账上：用户可用余额 +$1.00
- XBIT 复刻金额 = HL 实际金额（严格对齐）
- 若 XBIT 复刻与 HL 实际存在分歧 → 触发对账告警

---

## SC-FS-003：负资金费率 — 空头付多头收

**前置条件：**
- BTC 资金费率 = -0.005%（负值 = 空头付多头）
- 用户 BTC INTERNAL 多头：size=0.1 BTC，mark_price=$100,000

**结算计算：**
```
多头收入 = 0.1 × 100,000 × 0.00005 = $0.50
```

**预期结果：**
- 用户（多头）可用余额 +$0.50
- 平台对手方（空头）付出 $0.50
- 清算价小幅下移（资金费增加保证金缓冲，多头方向）

---

## SC-FS-004：中途开仓 — 当期资金费处理

**前置条件：**
- 资金费周期：UTC 00:00 ~ 08:00
- 用户在 UTC 04:00 开仓 BTC INTERNAL 多头

**预期结果：**
- 该仓位在 UTC 08:00 参与资金费结算（按完整当期费率，不按持仓时间比例折算）
- HL 的行为：只要在结算时间点持仓就按全额结算
- 第一次完整参与时间点：UTC 08:00

---

## SC-FS-005：资金费对账 — XBIT vs HL 偏差检测

**前置条件：**
- 多个用户持有 HYPERLIQUID 仓位
- UTC 00:00 资金费结算

**操作步骤：**
1. HL 完成资金费结算，平台 HL 账户变化 +$500
2. XBIT 按 HL 公布的 funding_rate 对每个用户的 HYPERLIQUID 仓位复刻结算
3. XBIT 复刻总额 = $498
4. 检测到偏差 = $2（0.4%）

**预期结果：**
- 偏差 0.4% < 1% → 记录 deviation_log，不触发告警
- 偏差 > 1% → Slack 告警
- 偏差 > 5% → P0 紧急，暂停该币种 HL 路由
- 每日汇总偏差 → reconciliation_logs
