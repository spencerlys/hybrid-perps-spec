---
doc_id: prd-zh-market-data
title: L7 市场数据层 — HL 数据透传、精度规则同步、异常处理
tags: [market-data, websocket, price, funding-rate, mark-price, precision, L7]
version: 1.0
lang: zh
updated: 2026-04-08
phase: Phase 1
---

# L7 市场数据层

## 数据来源

MVP 阶段所有市场数据均来自 Hyperliquid，XBIT 做格式适配和透传。

## 实时数据（WebSocket 透传）

| 数据类型 | HL 频道 | 更新频率 |
|---------|---------|---------|
| 实时价格 / 标记价格 | `allMids` | 实时 |
| 订单簿深度 | `l2Book` | 实时 |
| K 线数据 | `candle` | 按周期 |
| 资金费率 | `activeAssetCtx` | 实时 |
| 最新成交 | `trades` | 实时 |
| 用户仓位数据流 | `user` | 用户事件驱动 |

### WebSocket 多路复用

- 上游：一条 HL WS 连接（减少 HL 端连接限制）
- 下游：服务多个客户端
- 引用计数订阅：最后一个客户端取消订阅时才取消上游订阅
- 心跳检测 + 自动重连（指数退避，最大 30 秒）

## 静态数据（REST 同步）

| 数据类型 | 来源 | 同步频率 |
|---------|------|---------|
| 合约元数据（上架币种、杠杆范围） | HL `/info` | 每小时 |
| 数值精度规则（szDecimals、数量步进） | HL `/info` | 每小时 |
| 维保率分级表 | HL `/info` | 每小时 |
| 预测资金费率 | HL REST | 每小时 |
| 当前资金费率 | HL REST / WS | 实时 |

### 精度规则同步

从 HL `/info` 拉取的元数据中包含：
- `szDecimals`：价格/数量小数位，每币种不同
- `maxLeverage`：最大杠杆
- `marginTableTiers`：维保率分级表（按名义价值区间）

这些规则用于：
1. L5 清算价格计算（分级维保率）
2. 订单数量/价格取整（对齐 HL 精度）
3. 最小下单量校验

## HL 标记价格

**XBIT 不自行计算标记价格，直接使用 HL 推送的 mark price。**

- HL mark price = oracle_price × (1 + impact funding premium)
- 通过 `allMids` WebSocket 订阅实时获取
- 用于：未实现 PnL 计算、清算检测、保证金率计算

### 标记价格延迟监控

```
全链路延迟 = HL 生成时间 → XBIT 接收时间 → 用户展示时间

告警：> 200ms
紧急：> 500ms → 暂停对赌（不在极端行情下对赌）
```

## 资金费率透传

- 资金费率来源：HL，不自行计算
- 结算周期：每 8 小时（UTC 00:00、08:00、16:00）
- INTERNAL 仓位：按 HL 费率，XBIT 平台作为对手方收/付
- HYPERLIQUID 仓位：HL 实际结算，XBIT 镜像记录到用户账上

详见 [09-settlement.md](09-settlement.md) 资金费结算公式。

## 异常处理

### HL 断连

```
断连 → 立即告警
尝试重连：指数退避（1s, 2s, 4s, ... 最大 30s）
断连期间：
  - 暂停对赌（无实时价格不能对赌）
  - HL 订单转发暂停
  - 用户看到"数据加载中"
恢复后：全量快照刷新，增量继续
```

### 价格异常跳变

```
检测：单次价格变动 > 配置阈值（如 BTC 5% / 单次更新）
处理：使用最后有效价格
通知：告警 + 暂停对赌
恢复：连续 N 次正常更新后恢复
```

### 数据一致性校验

- 序列号校验：发现跳号 → 触发全量快照重建
- 定时全量比对：每 5 分钟将 XBIT 缓存价格与 HL REST 接口对比
