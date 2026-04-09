---
doc_id: scenarios-zh-index
title: 业务场景记录索引
version: 1.0
lang: zh
updated: 2026-04-09
---

# 业务场景记录

> 上级索引：[../index.md](../index.md)

场景记录用于捕捉具体业务场景的完整处理流程，包括正常路径和边界情况（异常/极端情况）。  
每个场景有唯一编号，可通过 ID 快速定位。

## 场景文件列表

| 文件 | 场景类别 | 场景数量 |
|------|---------|---------|
| [01-order-routing.md](01-order-routing.md) | 订单路由场景 | 8 |
| [02-liquidation.md](02-liquidation.md) | 清算场景 | 6 |
| [03-funding-settlement.md](03-funding-settlement.md) | 资金费结算场景 | 5 |
| [04-deposit-withdrawal.md](04-deposit-withdrawal.md) | 充提币场景 | 6 |
| [05-risk-hedge.md](05-risk-hedge.md) | 风控与对冲场景 | 8 |
| [06-edge-cases.md](06-edge-cases.md) | 边缘场景与极端情况 | 12 |

## 场景编写规范

每个场景包含以下字段：

```
场景 ID   : SC-XXX-001
场景名称  : 简短描述
前置条件  : 用户/系统初始状态
操作步骤  : 1. 2. 3. ...
预期结果  : 系统行为和最终状态
关联文件  : 相关 PRD 模块
```

## 快速检索

### 按路由类型
- INTERNAL 对赌场景 → [01-order-routing.md](01-order-routing.md) SC-RT-001 ~ SC-RT-003
- HYPERLIQUID 路由场景 → [01-order-routing.md](01-order-routing.md) SC-RT-004 ~ SC-RT-006

### 按清算模式
- 逐仓清算 → [02-liquidation.md](02-liquidation.md) SC-LQ-001 ~ SC-LQ-002
- 全仓清算 → [02-liquidation.md](02-liquidation.md) SC-LQ-003 ~ SC-LQ-004
- 混合模式清算 → [02-liquidation.md](02-liquidation.md) SC-LQ-005 ~ SC-LQ-006

### 按资产操作
- 充值 → [04-deposit-withdrawal.md](04-deposit-withdrawal.md) SC-DW-001 ~ SC-DW-002
- 提现 → [04-deposit-withdrawal.md](04-deposit-withdrawal.md) SC-DW-003 ~ SC-DW-006

### 按风控与对冲
- 净敞口触发对冲 → [05-risk-hedge.md](05-risk-hedge.md) SC-RH-001 ~ SC-RH-004
- 准备金与熔断 → [05-risk-hedge.md](05-risk-hedge.md) SC-RH-005 ~ SC-RH-008

### 按边缘场景
- 通道故障 → [06-edge-cases.md](06-edge-cases.md) SC-EC-001, SC-EC-010
- 资金危机 → [06-edge-cases.md](06-edge-cases.md) SC-EC-002, SC-EC-006
- 极端行情 → [06-edge-cases.md](06-edge-cases.md) SC-EC-003, SC-EC-004
- 系统异常 → [06-edge-cases.md](06-edge-cases.md) SC-EC-005, SC-EC-008, SC-EC-009, SC-EC-012
