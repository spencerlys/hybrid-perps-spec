---
doc_id: "domain-architecture-002"
title: "账本交易域与风控安全域 — 解耦架构、接口契约、调用规则"
tags: ["架构设计", "域设计", "解耦", "接口契约", "MVP"]
version: "1.0"
lang: "zh"
updated: "2026-04-09"
phase: "Phase 2"
---

# 账本交易域与风控安全域 — 解耦架构、接口契约、调用规则

## 概述

平台核心系统分为两个独立的业务域，通过严格的接口契约和单向依赖关系解耦：

1. **账本与交易处理域（Ledger & Trading Domain）** — 管理用户资产、订单流转、仓位记录、PnL计算
2. **风控与安全域（Risk Control Domain）** — 管理交易前审批、实时敞口监控、清算执行、风险准备金

两域严格禁止循环依赖，通过异步消息队列和指令发送实现通信。

---

## 域 1：账本与交易处理域（Ledger & Trading Domain）

### 职责范围

- **用户资产管理**（L1）：余额、冻结保证金、已用交叉保证金、未实现PnL
- **订单接收与验证**（L2）：订单格式验证、签名验证、基础规则检查（非风控）
- **订单路由决策**（L2）：根据金额、模式、风险准备金状态决定INTERNAL或HL路由
- **对赌执行**（L3）：INTERNAL成交记账、仓位初始化、成交费计入余额
- **HL代理执行**（L4）：HL API中继、回执验证、价格校正、HL仓位映射到用户账户
- **市场数据维护**（L7）：HL行情数据缓存、标记价格、资金费率、链上数据同步
- **结算对账**（L8）：PnL计算、资金费结算、对账修正、交易记录维护

### 核心模块

| 模块 | 功能 | 关键产出 |
|------|------|--------|
| L1 Account & Assets | 用户注册、多链充值、提现、余额管理 | 账户ID、钱包地址、余额快照 |
| L2 Order Routing | 订单分类、三模式路由、阈值决策 | 路由指令(INTERNAL/HL)、订单ID |
| L3 Platform B-book Execution | INTERNAL成交、仓位记账、费用结算 | 成交确认、仓位ID、成交价 |
| L4 HL Proxy Execution | HL下单、回执映射、价格校正 | HL订单ID、成交价、错误回执 |
| L7 Market Data | 行情订阅、缓存、同步 | 标记价格、资金费率、深度数据 |
| L8 Settlement & Reconciliation | PnL计算、资金费结算、对账 | 结算记录、盈亏单据、对账报告 |

### 禁止操作

❌ 直接修改用户余额（除成交/清算/提现）
❌ 自行触发清算（必须由风控域发指令）
❌ 对冲决策（完全由L6负责）
❌ 风控规则检查（超出订单格式验证范围）

---

## 域 2：风控与安全域（Risk Control Domain）

### 职责范围

- **预交易风控审批**：下单前检查保证金充足、头寸额度限制、用户风险等级、品种限制
- **盘中风险监控**：实时敞口计算、保证金率监控、大额订单告警、异常交易检测
- **清算检测与执行**：保证金率触发条件监控、清算优先级排序、执行编排（INTERNAL先清、HL次之）
- **对冲引擎**：净敞口聚合、对冲阈值判断、对冲比例计算、对冲指令下达
- **风险准备金管理**：准备金净值计算、入账/出账、阈值告警、准备金不足时冻结新INTERNAL订单
- **限额系统**：用户日成交额限额、风险值限额、头寸限额、跨品种关联限制
- **熔断开关**：极端行情熔断（波动>5%/小时）、HL连接故障熔断、清算链路故障熔断

### 核心模块

| 模块 | 功能 | 关键产出 |
|------|------|--------|
| Pre-Trade Risk Check | 下单前审批 | 通过/拒绝、原因、警告 |
| Exposure Monitor | 实时敞口计算、汇聚 | 净敞口、方向、实时更新 |
| Margin & Liquidation Engine | 保证金率计算、清算触发 | 保证金率、清算信号、执行ID |
| Hedge Engine (L6) | 对冲阈值、比例、指令生成 | 对冲指令、优先级、目标账户 |
| Risk Reserve Management | 准备金净值、流动性管理 | 准备金余额、可用额度、入账记录 |
| Circuit Breaker | 多维熔断监控 | 熔断状态、触发原因、回复条件 |

### 单向依赖规则

**风控域 → 账本域**：只读 + 发指令
- ✅ 读取：用户余额、仓位、净敞口、成交记录
- ✅ 发指令：清算执行、对冲指令、路由模式切换、冻结操作

**账本域 → 风控域**：只发事件
- ✅ 发事件：敞口变更（成交、平仓）、仓位变更、成交通知、清算执行结果

**禁止反向调用**
- ❌ 风控域不能直接修改账本数据
- ❌ 账本域不能自行触发清算、对冲、熔断
- ❌ 两域之间不能有同步RPC调用

---

## 6个核心接口完整定义

### 1️⃣ PreTradeCheck（预交易风控审批）

**目的**：订单下达前，验证是否满足风控条件

**请求参数**

```json
{
  "request_id": "order_req_20260409_001",
  "user_id": "user_12345",
  "symbol": "BTC",
  "side": "LONG",
  "size": 0.5,
  "notional": 20000,
  "leverage": 10,
  "margin_mode": "ISOLATED",
  "order_type": "MARKET",
  "timestamp": 1712688000000
}
```

**响应参数**

```json
{
  "approved": true,
  "order_id": "order_20260409_001",
  "reject_reason": null,
  "risk_warnings": [
    "account_equity_low: 已用保证金占总权益的92%, 建议补充资金"
  ],
  "metadata": {
    "margin_ratio": 0.92,
    "max_leverage_allowed": 15,
    "user_risk_level": "MEDIUM"
  }
}
```

**错误码**

| 错误码 | 含义 | HTTP Code |
|--------|------|-----------|
| INSUFFICIENT_MARGIN | 保证金不足 | 400 |
| SYMBOL_SUSPENDED | 品种暂停交易 | 400 |
| EXPOSURE_LIMIT_EXCEEDED | 敞口超限 | 400 |
| USER_RESTRICTED | 用户被限制 | 403 |
| RESERVE_INSUFFICIENT | 风险准备金不足，不能新开INTERNAL订单 | 400 |
| LEVERAGE_EXCEED | 杠杆超过用户额度 | 400 |

**幂等策略**

- 幂等键：`request_id`
- 缓存时间：100ms（防重复检查）
- 超时重试：不重试（由订单层处理）

**超时处理**

- 目标延迟：<10ms P99
- 超时时间：100ms
- 超时行为：返回500错误，不予放行

**调用者**：L2 订单路由模块
**使用频率**：每笔订单

---

### 2️⃣ ExposureChangeEvent（敞口变更事件）

**目的**：账本域向风控域通知INTERNAL敞口变化（异步、非阻塞）

**消息格式**

```json
{
  "event_id": "exposure_evt_20260409_001",
  "event_type": "OPEN",
  "user_id": "user_12345",
  "symbol": "BTC",
  "direction": "LONG",
  "position_size": 0.5,
  "entry_price": 40000,
  "delta_notional": 20000,
  "current_exposure": {
    "net_exposure": 45000,
    "long_notional": 65000,
    "short_notional": 20000
  },
  "timestamp": 1712688000000,
  "source": "L3_EXECUTION"
}
```

**生产者**：L3（对赌执行）、L4（HL执行回执）
**消费者**：L6（对冲引擎、敞口监控）
**传输方式**：Redis Streams，at-least-once
**确保单调性**：消费端使用消息ID排序，幂等去重

**超时处理**

- 消息队列堆积告警：>10000条未消费
- 消费延迟告警：>5秒
- 故障回复：人工触发全量对账

---

### 3️⃣ LiquidationTrigger（清算触发指令）

**目的**：风控域命令账本域执行用户仓位清算

**请求参数**

```json
{
  "trigger_id": "liq_trg_20260409_001",
  "position_id": "pos_20260408_123",
  "user_id": "user_12345",
  "symbol": "BTC",
  "direction": "LONG",
  "position_size": 0.5,
  "trigger_price": 38000,
  "margin_ratio": 0.095,
  "liquidation_type": "ISOLATED",
  "execution_target": "INTERNAL",
  "urgency": "HIGH",
  "timestamp": 1712688000000
}
```

**响应参数**

```json
{
  "execution_id": "liq_exe_20260409_001",
  "status": "EXECUTING",
  "position_id": "pos_20260408_123",
  "close_price": null,
  "filled_size": 0,
  "remaining_size": 0.5,
  "estimated_loss": -2000,
  "liquidation_fee": 2000,
  "timestamp": 1712688000000
}
```

**最终响应（成交后）**

```json
{
  "execution_id": "liq_exe_20260409_001",
  "status": "COMPLETED",
  "close_price": 38500,
  "filled_size": 0.5,
  "actual_loss": -2250,
  "liquidation_fee": 2000,
  "platform_profit": 1600,
  "reserve_fund_contribution": 400,
  "settlement_time_ms": 2500
}
```

**错误码**

| 错误码 | 含义 |
|--------|------|
| POSITION_ALREADY_CLOSED | 仓位已平 |
| POSITION_NOT_FOUND | 仓位不存在 |
| HL_UNAVAILABLE | HL清算失败（HL端仓位清算） |
| EXECUTION_TIMEOUT | 清算执行超时（>5s） |
| INSUFFICIENT_LIQUIDITY | 清算流动性不足 |
| UNAUTHORIZED | 指令来源非法 |

**幂等策略**

- 幂等键：`trigger_id`
- 已执行判断：查询成交记录，如存在则返回已完成状态
- 重试策略：3次重试，间隔1s，失败后告警

**超时处理**

- 目标延迟：<2000ms（清算时效要求）
- 超时时间：5000ms
- 超时后：标记为TIMEOUT，风控发告警，考虑启动HL_MODE

**调用者**：L6 风控域
**使用频率**：事件驱动（保证金率触发）

---

### 4️⃣ HedgeInstruction（对冲指令）

**目的**：风控域L6命令账本域L4在HL对冲平台净敞口

**请求参数**

```json
{
  "instruction_id": "hedge_instr_20260409_001",
  "symbol": "BTC",
  "direction": "SHORT",
  "size": 0.4,
  "leverage": 10,
  "max_slippage_bps": 100,
  "urgency": "NORMAL",
  "reason": "net_exposure_500k_exceeded",
  "timestamp": 1712688000000
}
```

**响应参数**

```json
{
  "execution_id": "hedge_exe_20260409_001",
  "status": "EXECUTING",
  "symbol": "BTC",
  "direction": "SHORT",
  "order_size": 0.4,
  "hl_order_id": "hl_order_99999",
  "timestamp": 1712688000000
}
```

**最终响应（成交后）**

```json
{
  "execution_id": "hedge_exe_20260409_001",
  "status": "COMPLETED",
  "symbol": "BTC",
  "fill_price": 40100,
  "filled_size": 0.4,
  "notional_filled": 16040,
  "taker_fee": 5.61,
  "net_cost": 5.61,
  "remaining_exposure": 100000,
  "settlement_time_ms": 3200
}
```

**错误码**

| 错误码 | 含义 |
|--------|------|
| HEDGE_ACCOUNT_INSUFFICIENT | 对冲账户保证金不足 |
| SLIPPAGE_EXCEEDED | 滑点超过限制 |
| HL_UNAVAILABLE | HL连接中断 |
| EXECUTION_TIMEOUT | 对冲指令超时 |
| INVALID_SIZE | 对冲大小超过限制 |

**幂等策略**

- 幂等键：`instruction_id`
- 查询HL订单，检查已成交部分
- 已执行判断：HL订单状态 = FILLED

**超时处理**

- 目标延迟：<500ms（对冲执行）
- 超时时间：30000ms
- 超时后：自动重试 OR 取消订单、告警

**调用者**：L6 对冲引擎
**使用频率**：阈值触发（敞口变化时）

---

### 5️⃣ RoutingModeChange（路由模式变更）

**目的**：切换L2的三种路由模式（NORMAL_MODE / HL_MODE / BETTING_MODE）

**请求参数**

```json
{
  "change_id": "mode_chg_20260409_001",
  "new_mode": "HL_MODE",
  "trigger_type": "AUTO",
  "operator_id": null,
  "reason": "net_exposure_exceeded_1m_threshold",
  "enforced": false,
  "timestamp": 1712688000000
}
```

**响应参数**

```json
{
  "applied": true,
  "change_id": "mode_chg_20260409_001",
  "previous_mode": "NORMAL_MODE",
  "new_mode": "HL_MODE",
  "effective_at": 1712688000100,
  "effective_for_ms": 0,
  "auto_fallback_threshold": {
    "trigger": "net_exposure_below_500k",
    "fallback_mode": "NORMAL_MODE"
  }
}
```

**错误码**

| 错误码 | 含义 |
|--------|------|
| ALREADY_IN_MODE | 已处于该模式 |
| UNAUTHORIZED | 操作者无权限 |
| INVALID_MODE | 模式不合法 |

**幂等策略**

- 幂等键：`change_id`
- 已执行判断：当前模式 = new_mode

**超时处理**

- 目标延迟：<100ms
- 超时时间：1000ms
- 超时后：报错，不修改模式

**调用者**：L6 风控域（自动）、L9 后台（手动）
**使用频率**：低频（分钟级）

---

### 6️⃣ BalanceQuery（余额与仓位查询 - 只读）

**目的**：风控域查询用户账户状态，用于决策

**请求参数**

```json
{
  "user_id": "user_12345",
  "include_details": false
}
```

**响应参数**

```json
{
  "user_id": "user_12345",
  "total_equity": 100000,
  "available_balance": 5000,
  "frozen_margin": 60000,
  "cross_margin_used": 35000,
  "unrealized_pnl": 2500,
  "positions": {
    "internal": [
      {
        "position_id": "pos_001",
        "symbol": "BTC",
        "direction": "LONG",
        "size": 0.5,
        "entry_price": 40000,
        "current_price": 41000,
        "margin_used": 20000,
        "margin_mode": "ISOLATED",
        "unrealized_pnl": 500
      }
    ],
    "hyperliquid": [
      {
        "symbol": "ETH",
        "direction": "SHORT",
        "size": 10,
        "entry_price": 2000,
        "current_price": 1950,
        "unrealized_pnl": 500
      }
    ]
  },
  "margin_ratios": {
    "account_level": 0.92,
    "isolated_positions": [0.85, 0.78]
  }
}
```

**错误码**

| 错误码 | 含义 |
|--------|------|
| USER_NOT_FOUND | 用户不存在 |
| ACCOUNT_LOCKED | 账户锁定 |

**超时处理**

- 目标延迟：<50ms
- 超时时间：200ms
- 超时后：返回缓存值（TTL 5s）

**调用者**：L6 对冲引擎、清算引擎、预风控检查
**使用频率**：高频（每秒多次）

---

## 跨域一致性策略

### 原则：同步为界、异步填补

| 范畴 | 通信方式 | 一致性保证 | 处理路径 |
|------|--------|---------|--------|
| 核心路径：下单→成交→记账 | 同步事务 | ACID | 数据库事务 + 分布式锁 |
| 跨域通知：敞口/仓位变更 | 异步消息 | At-least-once | Redis Streams + 消费端幂等 |
| 指令下达：清算/对冲/模式切换 | 异步消息 | At-least-once | 指令队列 + 重试 + 幂等键 |
| 读操作：查询余额/仓位 | 同步查询 | 最终一致 | 缓存 + TTL + 版本号 |

### 对账修正流程

**定时全量对账**（每小时一次）

```
1. 账本域导出：用户总equity、总仓位、总敞口、总风险值
2. 风控域导出：最新风险监控快照、清算黑名单、冻结列表
3. 比对维度：
   - 用户总equity是否一致
   - 净敞口数值是否一致
   - 清算被执行仓位是否同步
4. 发现偏差 → 生成对账单 → 告警 → 人工审核 → 补偿事务
```

**补偿事务示例**

| 偏差场景 | 根本原因 | 补偿方案 |
|--------|--------|--------|
| 清算执行失败，仓位未平 | HL连接断开或超时 | 重试3次(1s间隔) → 告警 → 人工确认 → 强制HL_MODE |
| 敞口数值不一致 | 成交事件丢失 | 扫描成交记录 → 补发敞口事件 → 重新计算对冲 |
| 风险准备金余额不对 | 入账记录丢失 | 核查用户亏损单据 → 补入账 → 告警 |
| 用户余额被篡改 | 异常操作日志 | 回滚到最近正确版本 + 人工审计 |

---

## 消息队列设计

### Topic 1: exposure_changes（敞口变更）

- **生产者**：L3（INTERNAL成交）、L4（HL回执）
- **消费者**：L6（对冲引擎、实时敞口计算）
- **消息格式**：见 ExposureChangeEvent
- **频率**：~每秒50-200条（根据交易量）
- **分片**：按用户ID分片（保证同用户消息顺序）
- **重试**：消费失败 → 死信队列 → 人工处理

### Topic 2: liquidation_triggers（清算指令）

- **生产者**：L5（保证金率监控）
- **消费者**：L3（INTERNAL清算）、L4（HL清算）
- **消息格式**：见 LiquidationTrigger
- **频率**：低频（事件驱动）
- **优先级**：INTERNAL优先清算，HL次之
- **重试**：最多3次，失败后告警 + 人工接管

### Topic 3: hedge_instructions（对冲指令）

- **生产者**：L6（对冲引擎）
- **消费者**：L4（HL对冲账户执行）
- **消息格式**：见 HedgeInstruction
- **频率**：中等（阈值触发，通常每小时5-10条）
- **确认机制**：L4消费后返回execution_id，L6追踪成交
- **失败处理**：告警 + 人工确认 → 重新对冲 OR 启动HL_MODE

### Topic 4: mode_changes（路由模式变更）

- **生产者**：L6（自动触发）、L9（人工操作）
- **消费者**：L2（订单路由逻辑）、L9（仪表盘更新）
- **消息格式**：见 RoutingModeChange
- **频率**：低频（分钟级）
- **有序性**：严格单线程消费，避免模式冲突
- **幂等性**：change_id去重，重复消息无害

---

## 给开发的落地要点

### 1. 同步事务 vs 异步消息的边界

**同步（强一致）**
- 订单创建 → 格式验证 → 路由决策 → L3/L4成交 → 余额更新
- 整个链路在数据库事务内完成
- 失败全部回滚，客户端同步获得成功/失败

**异步（最终一致）**
- 成交后立即向 exposure_changes 队列发消息
- L6异步消费，计算净敞口，决策对冲
- 延迟可容忍（100ms内）

**代码示例**
```python
# 同步路径：订单处理
with db.transaction():
    order = Order.create(user_id, symbol, side, size)
    if route == 'INTERNAL':
        position = Position.create(order)
        balance.update(fee=-fee_amount)
    else:
        hl_order = hl_api.create_order(...)
    db.commit()

# 异步通知：敞口变更
message_queue.publish('exposure_changes', {
    'event_id': uuid4(),
    'user_id': order.user_id,
    'delta_notional': order.notional,
    ...
})
```

### 2. 幂等键的生成与存储

每个跨域接口都有幂等键：
- PreTradeCheck: `request_id`（客户端生成 + timestamp）
- LiquidationTrigger: `trigger_id`（风控系统生成）
- HedgeInstruction: `instruction_id`（L6生成）
- RoutingModeChange: `change_id`（控制系统生成）

**存储方案**
```sql
CREATE TABLE idempotency_keys (
    key_type VARCHAR(50),
    key_value VARCHAR(256),
    result JSON,
    created_at TIMESTAMP,
    PRIMARY KEY (key_type, key_value),
    INDEX (created_at)
);
```

检查时：`SELECT result FROM idempotency_keys WHERE key_value = ?`

### 3. 消息队列的顺序与去重

**按用户分片**：确保同用户的敞口事件顺序
```python
partition_key = f"user_{event.user_id}"
message_queue.publish('exposure_changes', event, partition=partition_key)
```

**消费端去重**：idempotent consumer
```python
def consume_exposure_event(event):
    last_event_id = redis.get(f"last_event_{event.user_id}")
    if event.event_id <= last_event_id:
        return  # 已处理，跳过

    # 处理事件
    update_exposure(event)
    redis.set(f"last_event_{event.user_id}", event.event_id)
```

### 4. 风控接口的超时降级

PreTradeCheck 必须 <10ms，否则影响交易体验

**缓存策略**
```python
def pre_trade_check(order):
    cache_key = f"ptc_{order.user_id}_{order.symbol}"
    cached = cache.get(cache_key)
    if cached and cached['timestamp'] > now - 100ms:
        return cached['result']

    try:
        result = risk_service.check(order, timeout=10ms)
        cache.set(cache_key, result, ttl=100ms)
        return result
    except TimeoutError:
        # 使用上一次的结果或默认拒绝
        return cached or {'approved': False}
```

### 5. 清算执行的重试与告警

清算是高优先级操作，必须有重试机制

```python
def execute_liquidation(trigger):
    for retry in range(3):
        try:
            result = liquidation_service.execute(trigger, timeout=5000ms)
            if result.status == 'COMPLETED':
                return result
        except Exception as e:
            if retry < 2:
                sleep(1)
                continue
            else:
                alert(f"Liquidation {trigger.trigger_id} failed after 3 retries", severity=CRITICAL)
                # 强制进入HL_MODE，由人工处理
                routing_mode.set('HL_MODE')
                raise
```

### 6. 对账修正的自动化与人工介入

**小偏差自动修正**
- 偏差金额 <$100：自动修正
- 修正后生成审计日志

**大偏差人工审核**
- 偏差金额 ≥$100：暂停，告警
- 生成对账单，等待人工确认
- 人工确认后执行补偿事务

```python
def reconcile():
    discrepancies = ledger_domain.export() ^ risk_domain.export()

    for disc in discrepancies:
        if abs(disc.amount) < 100:
            auto_correct(disc)
        else:
            create_audit_ticket(disc, status='PENDING_REVIEW')
            alert(f"Large discrepancy detected: {disc.amount}")
```

---

## 总结

通过明确的域边界、单向依赖、同异混合通信，实现了：

✅ **解耦**：两域独立演化，接口稳定
✅ **可观测**：异步消息全链路可追踪
✅ **容错**：清算/对冲有重试 + 人工兜底
✅ **一致**：同步事务 + 异步对账双重保障
✅ **高效**：核心路径同步<10ms，非关键路径异步不阻塞

