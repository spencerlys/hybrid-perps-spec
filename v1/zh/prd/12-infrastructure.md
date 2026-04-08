---
doc_id: prd-zh-infrastructure
title: 基础设施 — 数据库、可观测性、安全、部署、性能目标
tags: [infrastructure, database, observability, security, deployment, performance]
version: 1.0
lang: zh
updated: 2026-04-08
phase: Phase 1
---

# 基础设施

## 性能目标

| 指标 | MVP 目标 |
|------|---------|
| 路由决策延迟 | < 5ms P99 |
| HL 订单转发延迟 | < 50ms |
| 内部对赌成交延迟 | < 10ms |
| 清算检测延迟 | < 1 秒（标记价变动 → 触发） |
| API 响应 P95 | < 100ms |
| 系统可用性 | 99.9% |
| 标准提现到账（< $10K） | < 5 分钟 |

## 数据库设计（关键表）

### 核心业务表

| 表名 | 关键字段 |
|------|---------|
| `users` | id, email, wallet_address, kyc_status |
| `wallets` | user_id, total_balance, available_balance, frozen_margin |
| `balance_logs` | user_id, type, amount, before, after, ref_id |
| `orders` | id, user_id, symbol, side, size, price, route, status |
| `positions` | id, user_id, symbol, direction, size, entry_price, margin_mode, position_source, status |
| `fills` | id, order_id, price, size, fee, source, hl_fill_price |
| `liquidations` | id, position_id, user_id, trigger_price, close_price, margin_zeroed |
| `funding_settlements` | id, position_id, rate, payment, settled_at |
| `deviation_logs` | id, position_id, xbit_pnl, hl_actual_pnl, deviation, deviation_rate, resolved_at |

### 资金管理表

| 表名 | 关键字段 |
|------|---------|
| `deposits` | id, user_id, chain, asset, amount, tx_hash, status, confirmations |
| `withdrawals` | id, user_id, chain, asset, amount, address, status, tx_hash, processed_at |
| `aggregations` | id, from_address, to_address, chain, amount, gas_used, tx_hash, status |
| `hot_wallets` | chain, address, balance, last_updated |

### 风控与对账表

| 表名 | 关键字段 |
|------|---------|
| `hedge_positions` | id, symbol, direction, size, entry_price, hl_position_id |
| `reconciliation_logs` | id, dimension, expected, actual, deviation_rate, status |
| `routing_config` | symbol, threshold, enabled, updated_at |
| `risk_config` | key, value, updated_by, updated_at |

## 可观测性

### 监控
- **Prometheus + Grafana**：业务指标（路由统计、PnL、偏差率、提现时效、清算数量）
- **基础设施指标**：CPU、内存、数据库连接数、Redis 命中率

### 日志
- **ELK（Elasticsearch + Logstash + Kibana）**：结构化日志，支持按订单 ID、用户 ID 检索
- 日志分级：DEBUG / INFO / WARN / ERROR

### P0 告警（5 分钟响应 SLA）

| 告警 | 级别 | 渠道 |
|------|------|------|
| 清算引擎失效 | P0 | PagerDuty + Slack |
| HL 连接断开 > 1 分钟 | P0 | PagerDuty + Slack |
| HL 保证金率 < 150% | P0 | PagerDuty + Slack |
| 热钱包余额 < $100K | P0 | PagerDuty + Slack |
| 单笔偏差率 > 5% | P0 | PagerDuty + Slack |
| 用户资产对账偏差 > 0.1% | P0 | PagerDuty + Slack |

## 安全

| 层面 | 措施 |
|------|------|
| API 认证 | HMAC 签名 + JWT + API Key 管理 |
| 提现保护 | 2FA（TOTP）+ 邮件确认 |
| 私钥管理 | Turnkey HSM（用户充值地址 + HL 平台账户 Agent Key） |
| 传输 | TLS 全链路加密 |
| 网络 | CDN + WAF + 速率限制（防 DDoS） |
| 审计 | 所有敏感操作审计日志（不可篡改） |

## 部署架构

| 组件 | 方案 |
|------|------|
| 容器化 | Docker + Kubernetes |
| CI/CD | GitHub Actions + 金丝雀发布 |
| 数据库 | PostgreSQL（主从复制） |
| 缓存 | Redis Cluster |
| 多区域 | 主备部署，DB 复制 |
| 灾备 | 降级模式：路由全量转 HL |

## 灾难恢复

**HL 连接中断（> 1 分钟）：**
1. 暂停新订单接入
2. 已有仓位继续监控（用最后有效价格）
3. 恢复后：全量同步 HL 仓位状态，校验一致性

**XBIT 服务中断：**
1. 切换到备用区域
2. 使用最新 DB 快照恢复
3. 重建 Redis 缓存（从 DB 重建）

**HL 账户爆仓（灾难性）：**
> 理论上不应发生（设计目标）。
1. 立即暂停所有 HL 相关操作
2. 人工审计所有受影响用户
3. 从风险准备金 + 平台资金补偿用户
