# Hybrid Perps Spec — 产品文档库

> **Hybrid Perpetual Futures Engine — Product Documentation Repository**
>
> 📦 [`github.com/spencerlys/hybrid-perps-spec`](https://github.com/spencerlys/hybrid-perps-spec)

本仓库是永续合约交易引擎的**产品规格与设计文档**。不包含实现代码，仅包含 PRD、业务场景记录、交易流程说明和 MVP 架构交付文档。
追求第一性原理，打破经验束缚，拆解问题至不可再分的基本事实，再通过演绎法推导解决方案。

**设计模型：方案二 — 内部对赌 + 阈值对冲（B-book）**，平台是唯一对手方与风控主体，Hyperliquid 仅是执行通道。

---

## 语言版本 / Language Versions

| 版本 | 入口 | 说明 |
|------|------|------|
| 中文（权威版本） | [`v1/zh/index.md`](v1/zh/index.md) | 所有文档以中文为准，完整目录与术语表见此 |
| English (translated) | [`v1/en/index.md`](v1/en/index.md) | English translation of Chinese source |

> **注意**：中文版本为权威来源。如中英文有歧义，以中文为准。

---

## 文档结构 / Document Structure

```
v1/
├── zh/                      # 中文（权威）
│   ├── index.md             # 文档总索引（完整模块列表与术语表）
│   ├── prd/                 # 产品需求文档（13 个模块 + 审计报告）
│   ├── scenarios/           # 业务场景记录（45+ 场景）
│   ├── trading/             # 交易处理流程（4 个流程）
│   └── mvp/                 # MVP 架构设计与交付文档（7 份）
└── en/                      # 英文（翻译版，结构同上）
mindmap/
└── index.html               # 全模块架构脑图（浏览器打开）
```

各目录的详细文件列表、模块说明和术语表请直接查看 [`v1/zh/index.md`](v1/zh/index.md)。

---

## 实施阶段 / Implementation Phases

| 阶段 | 核心交付 |
|------|---------|
| **Phase 1** | 账户系统 + HL API 中继（R0）+ 市场数据直通 + 多链充值 + 提现保障 |
| **Phase 2** | 路由引擎 + HL 代理执行 + 对冲引擎 + 净敞口监控 + 风险准备金（灰度：全量路由至 HL） |
| **Phase 3** | 内部对赌执行引擎 + 统一清算 + 保证金模型 + 风控仪表盘 |
| **Phase 4** | 路由阈值调优 + 对冲策略优化 + 大额拆单 + 跨阈值路由 |

---

## 如何阅读 / How to Read

1. **整体设计** → [`v1/zh/index.md`](v1/zh/index.md)（中文）或 [`v1/en/index.md`](v1/en/index.md)（English）
2. **某个模块** → `prd/` 对应文件，每个文件可独立阅读
3. **业务场景** → `scenarios/` 查看输入、判断逻辑和预期输出
4. **数据流转** → `trading/` 查看端到端流程图
5. **MVP 架构与交付** → `mvp/` 查看两域架构、开发计划、上线手册、验收清单
6. **架构全图** → 浏览器打开 [`mindmap/index.html`](mindmap/index.html)
