# Hybrid Perps Spec — 产品文档库

> **Hybrid Perpetual Futures Engine — Product Documentation Repository**
>
> 📦 [`github.com/spencerlys/hybrid-perps-spec`](https://github.com/spencerlys/hybrid-perps-spec)

本仓库是 永续合约交易引擎的**产品规格与设计文档**。不包含实现代码，仅包含 PRD、业务场景记录和交易流程说明。

---

## 语言版本 / Language Versions

| 版本 | 路径 | 说明 |
|------|------|------|
| 🇨🇳 中文（权威版本） | [`v1/zh/`](v1/zh/index.md) | 所有文档以中文为准 |
| 🇺🇸 English (translated) | [`v1/en/`](v1/en/index.md) | English translation of Chinese source |

> **注意**：中文版本为权威来源。如中英文有歧义，以中文为准。

---

## 文档结构 / Document Structure

每个语言版本下包含三类文档目录：

```
v1/
├── zh/                      # 中文（权威）
│   ├── index.md             # 文档总索引
│   ├── prd/                 # 产品需求文档（PRD）
│   ├── scenarios/           # 业务场景记录
│   └── trading/             # 交易处理流程
└── en/                      # 英文（翻译版）
    ├── index.md             # Master document index
    ├── prd/                 # Product Requirements Documents
    ├── scenarios/           # Business scenario records
    └── trading/             # Trading process flows
```

### PRD — 产品需求文档

| 文件 | 模块 | 实施阶段 |
|------|------|---------|
| [01-overview](v1/zh/prd/01-overview.md) | 项目概述、背景、收入模型 | — |
| [02-routing](v1/zh/prd/02-routing.md) | L2 订单路由层 | Phase 2 |
| [03-account-assets](v1/zh/prd/03-account-assets.md) | L1 账户与资产层 | Phase 1 |
| [04-internal-execution](v1/zh/prd/04-internal-execution.md) | L3 平台对赌执行层 | Phase 3 |
| [05-hl-execution](v1/zh/prd/05-hl-execution.md) | L4 HL 代理执行层 | Phase 2 |
| [06-margin-liquidation](v1/zh/prd/06-margin-liquidation.md) | L5 保证金与清算模型 | Phase 3 |
| [07-risk-management](v1/zh/prd/07-risk-management.md) | L6 风控与敞口管理 | Phase 3 |
| [08-market-data](v1/zh/prd/08-market-data.md) | L7 市场数据层 | Phase 1 |
| [09-settlement](v1/zh/prd/09-settlement.md) | L8 结算与对账层 | Phase 3 |
| [10-withdrawal-liquidity](v1/zh/prd/10-withdrawal-liquidity.md) | 提现流动性保障 | Phase 1 |
| [11-admin](v1/zh/prd/11-admin.md) | L9 后台管理层 | Phase 2–3 |
| [12-infrastructure](v1/zh/prd/12-infrastructure.md) | 基础设施 | Phase 1 |

### Scenarios — 业务场景记录

| 文件 | 内容 |
|------|------|
| [01-order-routing](v1/zh/scenarios/01-order-routing.md) | 订单路由判断的各类场景 |
| [02-liquidation](v1/zh/scenarios/02-liquidation.md) | 清算触发与执行场景 |
| [03-funding-settlement](v1/zh/scenarios/03-funding-settlement.md) | 资金费结算场景 |
| [04-deposit-withdrawal](v1/zh/scenarios/04-deposit-withdrawal.md) | 充提币场景 |

### Trading Flows — 交易处理流程

| 文件 | 内容 |
|------|------|
| [01-order-lifecycle](v1/zh/trading/01-order-lifecycle.md) | 订单全生命周期 |
| [02-internal-flow](v1/zh/trading/02-internal-flow.md) | 内部对赌执行流程 |
| [03-hl-proxy-flow](v1/zh/trading/03-hl-proxy-flow.md) | HL 代理执行流程 |
| [04-settlement-flow](v1/zh/trading/04-settlement-flow.md) | 结算与清算流程 |

---

## 架构可视化 / Architecture Visualization

打开 [`xbit-architecture-mindmap.html`](xbit-architecture-mindmap.html)（浏览器直接打开）可查看系统架构思维导图。

---

## 核心架构概念 / Key Concepts

**平台 是唯一的风控和清算主体，Hyperliquid（HL）仅是执行通道。**

| 原则 | 说明 |
|------|------|
| **双路由** | 订单金额 ≤$10K → 平台对赌（INTERNAL）；>$10K → Hyperliquid |
| **双仓位** | 同一用户可同时持有 INTERNAL 和 HYPERLIQUID 仓位 |
| **统一清算** | 所有清算决策由 平台 做出，与仓位所在系统无关 |
| **数据复用** | 价格、资金费、杠杆参数全部来自 HL，不自建行情系统 |

### 术语表

| 术语 | 定义 |
|------|------|
| `INTERNAL` | 平台对赌仓位，不在 HL 实际开仓 |
| `HYPERLIQUID` | 路由到 HL 执行的仓位 |
| 路由阈值 | 订单名义价值分界线（默认 $10,000） |
| 净敞口 | 平台在某资产上的方向性风险敞口（仅 INTERNAL） |
| 客损 | 用户亏损时平台的对赌收益（核心 B-Book 收入） |
| 计算偏差 | 平台 计算 PnL 与 HL 实际 PnL 的差额 |

---

## 实施阶段 / Implementation Phases

| 阶段 | 内容 |
|------|------|
| **Phase 1** | 账户系统 + HL API 完整封装 + 市场数据直通 |
| **Phase 2** | 路由引擎 + 金丝雀测试（初期全量路由至 HL） |
| **Phase 3** | 内部执行引擎 + 统一清算 + 保证金模型 |
| **Phase 4** | 路由阈值调优 + 对冲策略优化 |

---

## 如何阅读本文档 / How to Read

1. **了解整体设计**：从 [`v1/zh/index.md`](v1/zh/index.md)（或英文 [`v1/en/index.md`](v1/en/index.md)）开始
2. **了解某个模块**：直接进入 `prd/` 对应文件，每个文件均可独立阅读
3. **了解某个业务场景**：进入 `scenarios/` 查看具体场景的输入、判断逻辑和预期输出
4. **了解数据流转**：进入 `trading/` 查看端到端流程图和步骤说明
5. **查看系统架构全图**：用浏览器打开 `xbit-architecture-mindmap.html`
