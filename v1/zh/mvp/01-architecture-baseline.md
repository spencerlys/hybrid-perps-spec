# 统一架构基线（MVP）

## 设计原则

### 原则 1：最低成本保命

- **能复用就不自建**：HL已有的功能（价格、订单管理、保证金）能用API就用API，不重复实现
- **能配置化就不硬编码**：阈值、费率、杠杆倍数都应放在配置表，灰度期快速调整
- **能异步就不阻塞**：对冲指令、清算编排、事件通知都应异步，保证用户下单响应 <100ms
- **能人工兜底就先不上自动化**：偏差修正、异常清算、回滚决策在Phase 2可人工介入

### 原则 2：两域解耦

整个系统分为两个独立域，通过事件总线通信：

- **账本与交易处理域**（Ledger & Trading Domain）
  - 职责：用户账户、订单路由、对赌执行、HL代理、市场数据、结算对账
  - 特点：交易逻辑主导，响应性要求高，失败后自动恢复

- **风控与安全域**（Risk Control Domain）
  - 职责：预交易风控、盘中敞口监控、清算触发编排、对冲指令下达、风险准备金管理
  - 特点：安全第一，每个决策都需追踪审计，失败时主动断路

**禁止规则**：
- 两域间禁止同步调用（RPC），必须通过消息队列
- 禁止循环依赖（A→B→A）
- 风控域不能直接修改账本数据，只能通过事件触发账本域执行

### 原则 3：双账户隔离

平台在HL上维护两个账户，物理隔离，各有独立风险限额：

- **Trading Account**（交易账户）
  - 用途：处理用户大单（>$10K）、HL_MODE时的所有单
  - 最小资本：≥$500K
  - 目标保证金率：>500%（永远不能触发HL清算）

- **Hedge Account**（对冲账户）
  - 用途：执行净敞口对冲指令
  - 最小资本：≥$200K
  - 目标保证金率：>300%
  - 用途：吸收内部敞口，隔离交易账户风险

**监控规则**：
- 两账户余额独立监控，5分钟同步一次
- 若任一账户保证金率跌破200%，立即告警 + 进入HL_MODE
- 对冲完成后，两账户敞口应满足：|Trading Exposure| < $100K

### 原则 4：方案二纯粹性

- **对赌而非撮合**：平台是所有≤$10K订单的唯一对手方，不存在用户间匹配
- **单一风险承接者**：用户亏损→平台收益；用户盈利→平台支出
- **完整风险链闭合**：用户损益 + 对冲成本 + 准备金 = 平台损益
- **术语统一**：全文不出现"撮合"、"撮合引擎"、"匹配队列"等词汇，只用"对赌"

### 原则 5：MVP第一性

- **先闭环再完美**：优先实现最小可运行路径，不追求功能完整
- **先上线再优化**：灰度期发现问题后调整，不在上线前过度设计
- **可配置优于可扩展**：通过配置快速调整策略，不需要写代码
- **缺什么告警什么**：对关键路径每个环节都设置监控点，可见性优先

---

## 两域架构图

```
┌────────────────────────────────────────────────────────────────┐
│                   API Gateway (R0 - API Relay)                  │
│                  HL WebSocket/REST → 内部格式转换               │
└──────────────────────────┬─────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           │            事件总线            │
           │    (Redis Streams / Kafka)     │
           │               │               │
    ┌──────▼──────────┐   │     ┌─────────▼──────────┐
    │  账本与交易域    │────┼────▶│   风控与安全域      │
    │ Ledger Domain   │   │     │ Risk Domain     │
    │                 │   │     │                 │
    ├─────────────────┤   │     ├────────────────────┤
    │                 │   │     │                    │
    │ • L1 账户资产    │───┼─┐   │ • 预交易风控        │
    │   Users, Wallet │   │ │   │   (下单前检查)      │
    │                 │   │ │   │                    │
    │ • L2 订单路由    │───┼─┤   │ • 盘中敞口监控      │
    │   Route Engine  │   │ │   │   净敞口/限额      │
    │                 │   │ │   │                    │
    │ • L3 对赌执行    │◀──┼─┤   │ • 清算编排          │
    │   Internal B-K  │   │ │   │   触发/优先级      │
    │                 │   │ │   │                    │
    │ • L4 HL代理执行  │───┼─┤   │ • L6 对冲引擎      │
    │   HL Proxy      │   │ │   │   阈值/杠杆/倍数  │
    │                 │   │ │   │                    │
    │ • L7 市场数据    │───┼─┐   │ • 风险准备金        │
    │   Market Feed   │   │ │   │   累积/告警        │
    │                 │   │ │   │                    │
    │ • L8 结算对账    │───┼─┤   │ • 限额与熔断        │
    │   Settlement    │   │ │   │   日限/小时限      │
    │                 │   │ │   │                    │
    │ • L9a 后台(交易)  │───┼─┘   │ • L9b 后台(风控)   │
    │   Admin - Trade │   │     │   Admin - Risk  │
    │                 │   │     │                 │
    └────────┬────────┘   │     └────────┬────────┘
             │            │             │
             └─────┬──────┴──────┬──────┘
                   │             │
        ┌──────────▼──┐ ┌───────▼───────┐
        │  HL交易账户 │ │ HL对冲账户    │
        │  Trading    │ │ Hedge Account │
        │  ≥$500K     │ │ ≥$200K        │
        └─────────────┘ └───────────────┘
             │                  │
    ┌────────┼──────────────────┼────────┐
    │        │                  │        │
 ┌──▼─┐ ┌──▼──┐  ┌──────┐  ┌──▼─┐ ┌──▼──┐
 │TRON│ │Eth  │  │Solana│  │   │ │     │
 │钱包 │ │钱包 │  │钱包  │  │   │ │     │
 └────┘ └─────┘  └──────┘  └───┘ └─────┘
```

**图例说明**：
- 绿色箭头（`──▶`）：异步事件推送
- 蓝色箭头（`◀──`）：查询/反馈
- 红色箭头（`──┼──`）：关键通信

---

## 两域接口契约

### 接口 A：下单请求验证（交易域 → 风控域）

**触发时机**：用户提交新订单，L2路由决策完成后

**请求消息**

```
Message: ORDER_SUBMITTED
{
  "request_id": "req_20260409_001",      // 幂等键
  "timestamp": 1712639999000,             // Unix ms
  "user_id": "usr_alice_001",
  "order_id": "ord_20260409_001",
  "symbol": "BTC-USD",
  "side": "LONG",                         // LONG / SHORT
  "size": 0.5,                            // BTC
  "notional": 21500,                      // USD
  "leverage": 2,                          // 平台限制 ≤ 10x
  "margin_mode": "ISOLATED",              // ISOLATED / CROSS
  "route": "INTERNAL",                    // INTERNAL / HYPERLIQUID
  "limit_price": 43000.00,                // 仅对限价单非空
  "order_type": "LIMIT"                   // MARKET / LIMIT
}
```

**响应消息**

成功：
```
Message: ORDER_APPROVED
{
  "request_id": "req_20260409_001",
  "order_id": "ord_20260409_001",
  "approved": true,
  "reasons": []
}
```

拒绝：
```
Message: ORDER_REJECTED
{
  "request_id": "req_20260409_001",
  "order_id": "ord_20260409_001",
  "approved": false,
  "error_code": "RISK_EXPOSURE_EXCEED",  // 见下表
  "reason": "Net exposure after order would exceed $1M limit",
  "suggested_action": "RETRY_IN_BETTING_MODE"
}
```

**失败码**

| 错误码 | 含义 | 恢复方案 |
|------|------|--------|
| `RISK_EXPOSURE_EXCEED` | 净敞口超限 | 缩小仓位 / 风控手动调整限额 |
| `DAILY_LIMIT_EXCEED` | 日交易限额满 | 等待明天 |
| `RISK_RESERVE_LOW` | 准备金不足 | 人工补充准备金 |
| `ROUTING_MODE_HL_ONLY` | 当前仅允许HL路由 | 使用大单通道 |

**幂等键**：`request_id`（交易域需缓存，24小时内重复请求返回相同结果）

**超时**：<10ms（同步调用改异步后仍需提前响应，转为事件通知）

---

### 接口 B：敞口变更事件（交易域 → 风控域）

**触发时机**：订单成交、部分成交、平仓、清算执行后

**事件消息**

```
Message: EXPOSURE_CHANGED
{
  "event_id": "evt_20260409_001",
  "event_type": "ORDER_FILLED",          // ORDER_FILLED / PARTIAL_FILLED / POSITION_CLOSED / LIQUIDATED
  "timestamp": 1712640100000,
  "user_id": "usr_alice_001",
  "symbol": "BTC-USD",
  "side": "LONG",
  "delta_size": 0.5,                     // 本次变化量
  "delta_notional": 21500,               // 本次变化金额
  "execution_price": 43000.00,           // 实际成交价
  "route": "INTERNAL",                   // 在哪个系统成交

  // 累计敞口快照
  "snapshot": {
    "total_internal_long": 2.5,          // BTC
    "total_internal_short": 0.0,
    "total_hl_long": 1.0,
    "total_hl_short": 0.0,
    "net_exposure": {
      "BTC": {
        "internal": 2.5,
        "hyperliquid": 1.0,
        "notional": "128500",            // USD
        "direction": "LONG"
      }
    },
    "user_total_equity": "150000"        // USD
  }
}
```

**风控域响应**

```
Message: EXPOSURE_ACKNOWLEDGED
{
  "event_id": "evt_20260409_001",
  "status": "PROCESSED",
  "hedge_triggered": false,              // 是否触发了对冲
  "hedge_job_id": null,
  "actions": []
}
```

或

```
Message: HEDGE_INITIATED
{
  "event_id": "evt_20260409_001",
  "status": "HEDGE_QUEUED",
  "hedge_job_id": "hedge_20260409_001",
  "hedge_instruction": {
    "symbol": "BTC-USD",
    "direction": "SHORT",                // 与内部敞口反向
    "size": 1.25,                        // 50% hedge ratio
    "leverage": 1,
    "target_account": "HEDGE"
  }
}
```

**失败码**

| 错误码 | 含义 | 恢复 |
|------|------|-----|
| `SNAPSHOT_MISMATCH` | 敞口快照与风控记录不一致 | 触发对账，人工审查 |
| `HEDGE_QUEUE_FULL` | 对冲队列满 | 等待队列处理，降低接收速率 |
| `STALE_EVENT` | 事件时间戳过旧 | 交易域重新发送 |

**幂等键**：`event_id`

**超时**：<50ms（异步处理，风控域应在此时间内落库）

---

### 接口 C：清算触发指令（风控域 → 交易域）

**触发时机**：风控域检测到保证金率跌破清算线

**指令消息**

```
Message: LIQUIDATION_COMMAND
{
  "command_id": "liq_20260409_001",
  "timestamp": 1712640200000,
  "user_id": "usr_alice_001",
  "trigger_type": "MARGIN_RATIO_BREACH",  // MARGIN_RATIO / PRICE_LIMIT / MANUAL

  "positions": [
    {
      "position_id": "pos_20260409_001",
      "symbol": "BTC-USD",
      "side": "LONG",
      "size": 0.5,
      "route": "INTERNAL",                // 该仓位在哪个系统
      "margin_mode": "ISOLATED"           // 隔离清算
    }
  ],

  "liquidation_type": "PARTIAL",         // PARTIAL / FULL_ACCOUNT
  "execution_target": "INTERNAL",        // 在哪个系统执行平仓
  "priority": 1,                          // 1(紧急) ~ 5(普通)
  "timeout_ms": 5000                     // 清算最多等待5秒
}
```

**执行结果**

成功：
```
Message: LIQUIDATION_EXECUTED
{
  "command_id": "liq_20260409_001",
  "status": "COMPLETED",
  "positions_closed": ["pos_20260409_001"],
  "total_loss": 500.00,                 // 用户亏损金额
  "platform_gain": 500.00               // 平台收益
}
```

失败：
```
Message: LIQUIDATION_FAILED
{
  "command_id": "liq_20260409_001",
  "status": "FAILED",
  "error_code": "INTERNAL_ORDER_QUEUE_FULL",
  "error_msg": "无法立即执行，队列拥塞",
  "fallback_action": "ROUTE_TO_HL",      // 改为HL平仓
  "retry_count": 1,
  "max_retries": 3
}
```

**失败码**

| 错误码 | 含义 | 自动回滚 |
|------|------|--------|
| `INTERNAL_ORDER_QUEUE_FULL` | 内部订单队列满 | 是 → 路由到HL |
| `INSUFFICIENT_HL_BALANCE` | HL账户余额不足 | 是 → 告警 + 人工兜底 |
| `POSITION_ALREADY_CLOSED` | 仓位已平 | 是 → 标记完成 |
| `LIQUIDATION_PRICE_MOVED` | 目标价格已滑移 | 是 → 重新计价 + 重试 |

**幂等键**：`command_id`（3小时内重复发送返回相同结果，防止重复平仓）

**超时**：<1000ms（清算必须快速响应，由风控域发起重试）

---

### 接口 D：对冲指令下达（风控域 → 交易域 L4）

**触发时机**：敞口变更后，风控域计算得出应对冲头寸

**指令消息**

```
Message: HEDGE_INSTRUCTION
{
  "hedge_job_id": "hedge_20260409_001",
  "created_at": 1712640300000,

  "symbol": "BTC-USD",
  "direction": "SHORT",                  // 与内部敞口反向
  "size": 1.25,                          // 对冲头寸（单位：币）
  "leverage": 1,                         // 对冲账户通常1倍
  "target_account": "HEDGE",             // 固定值

  "hedge_ratio": 0.5,                    // 当前敞口的50%
  "net_exposure_before": 2500,           // 对冲前净敞口（USD）
  "target_exposure_after": 1250,         // 对冲后目标敞口

  "max_slippage_bps": 20,                // 最大滑点 20 basis points = 0.2%
  "timeout_ms": 30000,                   // 对冲指令30秒超时
  "retry_policy": {
    "max_attempts": 3,
    "backoff_ms": 1000
  }
}
```

**执行结果**

成功：
```
Message: HEDGE_COMPLETED
{
  "hedge_job_id": "hedge_20260409_001",
  "status": "COMPLETED",
  "execution_details": {
    "symbol": "BTC-USD",
    "direction": "SHORT",
    "size_executed": 1.25,
    "average_price": 42950.00,
    "total_cost": 1050.00,               // 对冲成本（利差）
    "slippage_bps": 15                   // 实际滑点 15 bps
  },
  "hl_order_id": "hl_ord_20260409_001",
  "new_net_exposure": 1250,
  "hedge_reserve_used": 100.00           // 从准备金中扣除
}
```

**失败码**

| 错误码 | 含义 | 处理 |
|------|------|-----|
| `HEDGE_PRICE_TOO_VOLATILE` | 价格波动过大 | 降低对冲大小 → 重试 |
| `HEDGE_TIMEOUT` | 对冲指令超时 | 降级为部分对冲 → 告警 |
| `HEDGE_ACCOUNT_LOW_MARGIN` | 对冲账户保证金不足 | 补充对冲账户资金 |

**幂等键**：`hedge_job_id`

**超时**：<30s（异步执行，不阻塞用户下单）

---

### 接口 E：路由模式变更（风控域 → 交易域 L2）

**触发时机**：风控域评估系统风险状态，需要切换路由策略

**指令消息**

```
Message: ROUTING_MODE_CHANGE
{
  "command_id": "route_change_20260409_001",
  "timestamp": 1712640400000,

  "new_mode": "HL_MODE",                 // HL_MODE / NORMAL_MODE / BETTING_MODE
  "trigger_reason": "HL_ACCOUNT_MARGIN_LOW",
  "trigger_details": {
    "current_margin_ratio": 240,         // %
    "threshold": 300,                    // %
    "deficit": "$60K"
  },

  "operator": "SYSTEM",                  // SYSTEM / admin_user_id
  "approval_required": false,            // MVP期可由系统自动切换

  "effective_immediately": true,
  "fallback_mode": "NORMAL_MODE"         // 故障降级模式
}
```

**执行结果**

```
Message: ROUTING_MODE_CHANGED
{
  "command_id": "route_change_20260409_001",
  "status": "COMPLETED",
  "old_mode": "NORMAL_MODE",
  "new_mode": "HL_MODE",
  "effective_at": 1712640401000,
  "pending_orders": {
    "affected_count": 5,
    "action": "ROUTE_PENDING_TO_HL"      // 将待成交订单改向HL
  }
}
```

**失败码**

| 错误码 | 含义 | 回退 |
|------|------|-----|
| `MODE_ALREADY_ACTIVE` | 已是目标模式 | 忽略 |
| `INVALID_MODE_TRANSITION` | 非法模式转移 | 驳回，不执行 |

**幂等键**：`command_id`

**超时**：<5ms（路由模式切换需极快）

---

### 接口 F：余额与风险查询（风控域查询交易域）

**触发时机**：风控决策前需要最新账户状态

**查询请求**

```
Message: ACCOUNT_STATE_QUERY
{
  "query_id": "query_20260409_001",
  "user_id": "usr_alice_001",
  "include_fields": [
    "available_balance",
    "total_equity",
    "margin_used",
    "margin_ratio",
    "unrealized_pnl",
    "exposure_by_symbol",
    "liquidation_price"
  ]
}
```

**查询响应**

```
Message: ACCOUNT_STATE_SNAPSHOT
{
  "query_id": "query_20260409_001",
  "timestamp": 1712640500000,
  "user_id": "usr_alice_001",

  "available_balance": 95000.00,         // 可用资金
  "total_equity": 150000.00,             // 总权益 = 初始 + 盈亏
  "margin_used": 55000.00,               // 已用保证金
  "margin_ratio": 272,                   // % (总权益 / 保证金)

  "unrealized_pnl": 5000.00,             // 未实现盈亏
  "total_notional": 150000.00,           // 总名义敞口

  "exposure": {
    "BTC-USD": {
      "internal_long": 0.5,
      "internal_short": 0.0,
      "hl_long": 1.0,
      "hl_short": 0.0,
      "net_long": 1.5,
      "notional_usd": 64500,
      "liquidation_price": 20000.00      // 这个价格会被清算
    }
  },

  "cumulative_loss_today": 2000.00,      // 日累计亏损
  "cumulative_gain_today": 7000.00       // 日累计盈利
}
```

**失败码**

| 错误码 | 含义 | 处理 |
|------|------|-----|
| `USER_NOT_FOUND` | 用户不存在 | 返回404 |
| `BALANCE_STALE` | 数据超过5秒 | 返回数据但标记为stale |

**超时**：<50ms（查询操作应极快，使用本地缓存 + 定期刷新）

---

## 一致性策略

### 核心原则：最终一致性 + 定时对账修正

**假设**：两域间通过异步事件通信，可能出现短暂不一致。通过定时对账和补偿事务保证最终一致性。

### 对账频率表

| 对账项 | 频率 | 容忍偏差 | 超出处理 |
|------|------|--------|--------|
| **用户资产实时** | 实时 | ±$1 | 标记告警，冻结账户 |
| **HL仓位映射** | 5分钟 | ±0.01个币 | 发送修正指令 |
| **B-book敞口** | 小时级 | ±$10K | 人工审查 + 修正 |
| **风险准备金** | 日级 | ±$100 | 次日调整 |
| **用户权益** | 日级 | ±$100 | 标记异常，人工审查 |

### 补偿事务（Compensating Transactions）

**场景 1：清算指令执行失败**

1. 风控域发送 `LIQUIDATION_COMMAND` 给交易域 L3
2. L3 应在 5秒内返回 `LIQUIDATION_EXECUTED` 或 `LIQUIDATION_FAILED`
3. 若失败：
   - 若失败原因是 L3 队列满 → L3 自动改向 L4（HL 清算）
   - 若失败原因是 HL 账户余额不足 → 告警 + 人工干预
   - 失败超过 3 次 → 强制进入 HL_MODE，防止进一步亏损

**场景 2：对冲指令丢失**

1. 风控域发送 `HEDGE_INSTRUCTION` 给 L4（HL 代理）
2. L4 应返回 `HEDGE_COMPLETED` 或 `HEDGE_FAILED`
3. 若 30 秒未收到回复 → 风控域重新查询 HL 账户敞口
4. 若敞口未变化 → 重新发送对冲指令，增加重试计数
5. 重试 3 次仍失败 → 告警 + 手动介入

**场景 3：敞口快照不一致**

1. 交易域发送 `EXPOSURE_CHANGED` 事件，包含敞口快照
2. 风控域每小时重新计算用户的累计敞口，与快照对比
3. 若偏差 > $10K → 发送 `REBALANCE_INSTRUCTION` 给交易域
4. 交易域根据指令调整内部账本

### 调用规则

| 规则 | 含义 | 违反后果 |
|------|------|--------|
| **风控只读查询** | 风控域可读交易域所有数据，但不能直接修改 | 代码审查 + 架构告警 |
| **通过事件修改** | 若风控需要修改账本（如冻结账户），必须通过事件 | 事务失败 + 人工兜底 |
| **禁止RPC同步** | 两域间禁止同步RPC调用，全部异步化 | 超时 → 自动降级 |
| **幂等键必须** | 所有跨域消息必须携带幂等键，接收端缓存结果 | 消息重复处理 → 告警 |
| **超时自动重试** | 消息未收到回复，发送端自动重试，但不阻塞主流程 | 可配置重试次数 |

---

## 系统层次总览

| 层 | 名称 | 所属域 | Phase | 核心职责 |
|---|------|------|-------|---------|
| **R0** | API中继 | 账本交易域 | 1 | HL WebSocket/REST 代理；格式转换；透明迁移 |
| **L1** | 账户与资产 | 账本交易域 | 1 | 用户注册；钱包管理；多链存取 |
| **L2** | 订单路由 | 账本交易域 | 2 | 三模式路由决策；下单前审批（调风控） |
| **L3** | 对赌执行 | 账本交易域 | 3 | 内部仓位管理；对赌订单队列；平仓执行 |
| **L4** | HL代理执行 | 账本交易域 | 2 | HL API包装；填单优化；仓位映射 |
| **L5** | 保证金与清算 | 账本交易域 | 3 | 隔离/全仓保证金计算；统一清算引擎（由风控触发） |
| **L6** | 风险与对冲 | 风控安全域 | 2 | 净敞口监控；对冲阈值决策；风险准备金管理 |
| **L7** | 市场数据 | 账本交易域 | 1 | HL 实时价格、资金费率、订单簿 |
| **L8** | 结算对账 | 账本交易域 | 3 | 盈亏计算；资金费用结算；跨系统对账 |
| **L9a** | 后台(交易) | 账本交易域 | 2 | 用户查询；订单历史；账户管理 |
| **L9b** | 后台(风控) | 风控安全域 | 2 | 路由模式切换；对冲账户监控；风险准备金配置 |
| **Infra** | 基础设施 | 两域共享 | 1-3 | 数据库；消息队列；监控告警；部署框架 |

---

## 给开发的落地要点

### 1. 消息队列是两域的唯一通信通道

- 使用 Redis Streams 或 Kafka
- 交易域发送事件到 `topic:ledger.events`
- 风控域消费该主题，发送指令到 `topic:risk.commands`
- **禁止**：RPC 调用、共享内存、直接数据库修改

### 2. 风控域是"观察者 + 断路器"，不是"执行者"

- 风控域**不执行**用户订单、清算、对冲
- 风控域**决定**是否清算、如何对冲，并发出指令
- 交易域**执行**这些指令，并返回结果

### 3. 清算的完整编排

```
1. 风控域：检测保证金率 < 清算线
2. 风控域：发送 LIQUIDATION_COMMAND 给 L3/L5
3. L3/L5：执行平仓订单
4. L3/L5：发送 LIQUIDATION_EXECUTED 回风控
5. 风控域：收到成功确认，标记清算完成
6. L8：计算最终损益，写入结算单
```

**关键**：若 L3 失败 → L5 自动改为 HL 平仓；若都失败 → 告警 + 人工兜底

### 4. HL 双账户使用不同的 Turnkey 地址

- **Trading Account**：`0x...HL_TRADING_...`，专用于用户大单
- **Hedge Account**：`0x...HL_HEDGE_...`，专用于对冲

两账户独立监控，不混用。在系统配置中硬编码两个地址。

### 5. 所有跨域调用必须带幂等键

- 交易域发送：`request_id` 或 `event_id`
- 风控域发送：`command_id` 或 `hedge_job_id`
- 接收端缓存 24 小时内的历史消息，重复请求返回缓存结果

### 6. 对冲指令异步执行，不阻塞用户下单流程

- 用户下单 → 立即返回 200 OK
- 敞口变化 → 异步通知风控域
- 风控域判断需要对冲 → 发送对冲指令到 L4
- L4 异步执行对冲，期间用户仍可下单

### 7. 灰度期 Phase 2：全量路由 HL，风控域仅观测模式

MVP 上线后，Phase 2 期间的策略：
- L2 订单路由：所有订单（不分大小）路由到 HL（即 HL_MODE）
- L3 对赌执行：代码完成但**关闭**，不接受订单
- L6 对冲引擎：**观测模式**，计算对冲但不发送指令
- 风险准备金：照常累积，但不用于对冲

**目的**：通过全 HL 路由验证系统可靠性，同时为 L3 上线预热。一旦系统稳定 3 周 + 所有告警都处理完，进入 Phase 3，启动 L3 对赌执行。

### 8. 每个域独立部署、独立扩缩容

- 账本与交易域：Pod 数量跟随"下单 QPS"
- 风控与安全域：Pod 数量跟随"敞口变化频率"

两域部署分离，故障隔离。交易域故障不影响风控域的监控和告警。

---

## 性能目标

| 操作 | P99 延迟 | 可靠性 |
|------|---------|------|
| 路由决策 | <5ms | 99.99% |
| HL 订单转发 | <50ms | 99.95% |
| 内部对赌执行 | <10ms | 99.99% |
| 清液检测 | <1s | 99.99% |
| API 响应 | <100ms | 99.9% |
| 敞口快照一致性 | <5分钟 | 100% |

---

## 关键配置表

这些值在 Phase 2 应全部可配置（数据库表），不硬编码：

```sql
-- 风控配置表示例
CREATE TABLE risk_config (
  id INT,
  param_name VARCHAR(64),          -- 如 NORMAL_MODE_INTERNAL_LIMIT
  param_value INT,                 -- 如 10000
  unit VARCHAR(16),                -- 如 USD
  min_value INT,
  max_value INT,
  enabled BOOLEAN,
  updated_at TIMESTAMP,
  updated_by VARCHAR(64)
);

INSERT INTO risk_config VALUES
  (1, 'NORMAL_MODE_INTERNAL_LIMIT', 10000, 'USD', 1000, 100000, TRUE, ...),
  (2, 'BETTING_MODE_INTERNAL_LIMIT', 50000, 'USD', 10000, 500000, TRUE, ...),
  (3, 'HEDGE_THRESHOLD_MIN', 100000, 'USD', 50000, 500000, TRUE, ...),
  (4, 'HEDGE_THRESHOLD_MAX', 1000000, 'USD', 500000, 5000000, TRUE, ...),
  (5, 'HEDGE_RATIO_LOW', 0.5, 'RATIO', 0.1, 0.9, TRUE, ...),
  (6, 'HEDGE_RATIO_HIGH', 0.8, 'RATIO', 0.1, 0.9, TRUE, ...);
```

---

## 版本信息

| 项 | 值 |
|----|---|
| 文档版本 | v1.1-MVPArch |
| 修订日期 | 2026-04-09 |
| 适用对象 | 后端/风控/测试团队 |
| 审核状态 | 待技术评审 |
