---
doc_id: v1-zh-index
title: XBIT 永续合约交易引擎 — 文档总索引
version: 1.0
lang: zh
updated: 2026-04-08
---

# XBIT 永续合约交易引擎 · 文档总索引

> **版本**：v1.0 | **状态**：正式版 | **语言**：中文（权威版本）| **更新**：2026-04-08

英文版本：[../en/index.md](../en/index.md)

---

## 文档结构

| 目录 | 用途 |
|------|------|
| [`prd/`](prd/00-index.md) | 产品需求文档（PRD 设计规划） |
| [`scenarios/`](scenarios/00-index.md) | 业务场景记录 |
| [`trading/`](trading/00-index.md) | 交易处理流程 |

---

## 核心架构原则

**XBIT 是唯一的风控和清算主体，Hyperliquid（HL）仅是执行通道。**

| 原则 | 说明 |
|------|------|
| 双路由系统 | ≤$10K → 平台对赌（INTERNAL）；>$10K → Hyperliquid（HYPERLIQUID） |
| 双仓位系统 | INTERNAL 内部仓位与 HYPERLIQUID 仓位可同时存在 |
| 统一清算 | 所有清算触发由 XBIT 判断，无论仓位在哪个系统 |
| 数据复用 | 价格、资金费、杠杆、币种全部复用 HL，不自建 |

---

## PRD 产品需求文档

| 文件 | 模块 | 阶段 |
|------|------|------|
| [00-index.md](prd/00-index.md) | PRD 总索引 | — |
| [01-overview.md](prd/01-overview.md) | 项目概述 | — |
| [02-routing.md](prd/02-routing.md) | L2 订单路由层 | Phase 2 |
| [03-account-assets.md](prd/03-account-assets.md) | L1 账户与资产层 | Phase 1 |
| [04-internal-execution.md](prd/04-internal-execution.md) | L3 平台对赌执行层 | Phase 3 |
| [05-hl-execution.md](prd/05-hl-execution.md) | L4 HL 代理执行层 | Phase 2 |
| [06-margin-liquidation.md](prd/06-margin-liquidation.md) | L5 保证金与清算模型 | Phase 3 |
| [07-risk-management.md](prd/07-risk-management.md) | L6 风控与敞口管理层 | Phase 3 |
| [08-market-data.md](prd/08-market-data.md) | L7 市场数据层 | Phase 1 |
| [09-settlement.md](prd/09-settlement.md) | L8 结算与对账层 | Phase 3 |
| [10-withdrawal-liquidity.md](prd/10-withdrawal-liquidity.md) | 提现流动性保障 | Phase 1 |
| [11-admin.md](prd/11-admin.md) | L9 后台管理层 | Phase 2–3 |
| [12-infrastructure.md](prd/12-infrastructure.md) | 基础设施 | Phase 1 |

---

## 场景记录

| 文件 | 场景类别 |
|------|---------|
| [00-index.md](scenarios/00-index.md) | 场景总索引 |
| [01-order-routing.md](scenarios/01-order-routing.md) | 订单路由场景 |
| [02-liquidation.md](scenarios/02-liquidation.md) | 清算场景 |
| [03-funding-settlement.md](scenarios/03-funding-settlement.md) | 资金费结算场景 |
| [04-deposit-withdrawal.md](scenarios/04-deposit-withdrawal.md) | 充提币场景 |

---

## 交易处理流程

| 文件 | 流程 |
|------|------|
| [00-index.md](trading/00-index.md) | 流程总索引 |
| [01-order-lifecycle.md](trading/01-order-lifecycle.md) | 订单全生命周期 |
| [02-internal-flow.md](trading/02-internal-flow.md) | 内部对赌执行流程 |
| [03-hl-proxy-flow.md](trading/03-hl-proxy-flow.md) | HL 代理执行流程 |
| [04-settlement-flow.md](trading/04-settlement-flow.md) | 结算与清算流程 |

---

## 术语表

| 术语 | 定义 |
|------|------|
| INTERNAL | 平台对赌仓位，不在 HL 上实际开仓 |
| HYPERLIQUID | 路由到 HL 执行的仓位 |
| 对赌（Internalization） | 平台作为用户对手方 |
| 客损 | 用户亏损时平台的对赌收益 |
| 路由阈值 | 订单金额分界线（默认 $10,000） |
| 净敞口 | 平台在某币种上的方向性风险总和 |
| 计算偏差 | XBIT 计算 PnL 与 HL 实际 PnL 的差额 |
| 归集 | 将分散在各链用户地址的资产转入平台热钱包 |
| 热钱包 | 在线签名钱包，用于日常提现出金 |
| 冷钱包 | 离线签名钱包，用于大额资金安全存储 |
