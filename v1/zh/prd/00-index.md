---
doc_id: prd-zh-index
title: XBIT Perp Engine PRD v1.0 — 文档索引
tags: [index, overview, prd]
version: 1.0
lang: zh
updated: 2026-04-08
---

# XBIT Perp Engine PRD v1.0 — 产品需求文档索引

> **v1.0 正式版** | 权威版本（中文）| 上级索引：[../index.md](../index.md)

每个文件对应一个独立模块，可单独检索，不依赖其他文件上下文。

## 文档结构

| 文件 | 模块 | 核心内容 | 阶段 |
|------|------|---------|------|
| [01-overview.md](01-overview.md) | 项目概述 | 背景、目标、范围边界、MVP 阶段、收入模型 | — |
| [02-routing.md](02-routing.md) | L2 订单路由层 | 路由规则、阈值、强制 HL 条件、平仓路由 | Phase 2 |
| [03-account-assets.md](03-account-assets.md) | L1 账户与资产层 | 注册、充提币、多链归集、热冷钱包 | Phase 1 |
| [04-internal-execution.md](04-internal-execution.md) | L3 平台对赌执行层 | 内部仓位模型、对赌记账、成交价格规则 | Phase 3 |
| [05-hl-execution.md](05-hl-execution.md) | L4 HL 代理执行层 | 账户架构、仓位映射、回执校正、偏差监控 | Phase 2 |
| [06-margin-liquidation.md](06-margin-liquidation.md) | L5 保证金与清算模型 | 逐仓/全仓公式、HL 精度复刻、清算引擎 | Phase 3 |
| [07-risk-management.md](07-risk-management.md) | L6 风控与敞口管理层 | 净敞口计算、对冲策略、风险准备金 | Phase 3 |
| [08-market-data.md](08-market-data.md) | L7 市场数据层 | HL 数据透传、精度规则同步、异常处理 | Phase 1 |
| [09-settlement.md](09-settlement.md) | L8 结算与对账层 | PnL 公式、资金费结算、偏差兜底、三方对账 | Phase 3 |
| [10-withdrawal-liquidity.md](10-withdrawal-liquidity.md) | 提现流动性保障 | 5 分钟到账保障、分级处理、流动性预警 | Phase 1 |
| [11-admin.md](11-admin.md) | L9 后台管理层 | 路由配置、风控仪表盘、权限管理（RBAC） | Phase 2–3 |
| [12-infrastructure.md](12-infrastructure.md) | 基础设施 | 数据库设计、可观测性、安全、部署、性能目标 | Phase 1 |

## 核心架构原则

**XBIT 是唯一的风控和清算主体，HL 仅是执行通道。**

- XBIT 内部维护每个用户的完整仓位表（含 INTERNAL 和 HYPERLIQUID 两类）
- XBIT 用 HL 标记价格，自行计算保证金率和清算条件
- INTERNAL 仓位：内部结算，平台赚取客损
- HYPERLIQUID 仓位：XBIT 判断触发时机 → 向 HL 发平仓指令
- 平台 HL 账户始终保持超额保证金，确保 HL 永远不会主动清算平台仓位

## 术语表

| 术语 | 定义 |
|------|------|
| INTERNAL | 平台对赌仓位，不在 HL 上实际开仓 |
| HYPERLIQUID | 路由到 HL 执行的仓位 |
| 对赌（Internalization） | 平台作为用户对手方，用户盈亏与平台相反 |
| 客损 | 用户亏损时平台作为对手方的收益 |
| 路由阈值 | 订单名义价值分界线（默认 $10,000） |
| 净敞口 | 平台在某币种上因对赌累积的方向性风险总和 |
| 计算偏差 | XBIT 计算 PnL 与 HL 实际 PnL 的差额 |
| 归集 | 将分散在各链用户充值地址的资产转入平台热钱包 |
| 热钱包 | 在线签名钱包，用于日常提现出金 |
| 冷钱包 | 离线签名钱包，用于大额资金安全存储 |
