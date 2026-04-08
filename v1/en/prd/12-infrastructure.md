---
doc_id: prd-en-infrastructure
title: Infrastructure — Database, Observability, Security, Deployment, Performance
tags: [infrastructure, database, observability, security, deployment, performance]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 1
---

# Infrastructure

## Performance Targets

| Metric | MVP Target |
|--------|-----------|
| Routing decision latency | < 5ms P99 |
| HL order forwarding latency | < 50ms |
| Internal execution latency | < 10ms |
| Liquidation detection latency | < 1 second (mark price change → trigger) |
| API response P95 | < 100ms |
| System availability | 99.9% |
| Standard withdrawal SLA (< $10K) | < 5 minutes |

## Database Design (Key Tables)

### Core Business Tables

| Table | Key Fields |
|-------|-----------|
| `users` | id, email, wallet_address, kyc_status |
| `wallets` | user_id, total_balance, available_balance, frozen_margin |
| `balance_logs` | user_id, type, amount, before, after, ref_id |
| `orders` | id, user_id, symbol, side, size, price, route, status |
| `positions` | id, user_id, symbol, direction, size, entry_price, margin_mode, position_source, status |
| `fills` | id, order_id, price, size, fee, source, hl_fill_price |
| `liquidations` | id, position_id, user_id, trigger_price, close_price, margin_zeroed |
| `funding_settlements` | id, position_id, rate, payment, settled_at |
| `deviation_logs` | id, position_id, xbit_pnl, hl_actual_pnl, deviation, deviation_rate, resolved_at |

### Fund Management Tables

| Table | Key Fields |
|-------|-----------|
| `deposits` | id, user_id, chain, asset, amount, tx_hash, status, confirmations |
| `withdrawals` | id, user_id, chain, asset, amount, address, status, tx_hash, processed_at |
| `aggregations` | id, from_address, to_address, chain, amount, gas_used, tx_hash, status |
| `hot_wallets` | chain, address, balance, last_updated |

### Risk & Reconciliation Tables

| Table | Key Fields |
|-------|-----------|
| `hedge_positions` | id, symbol, direction, size, entry_price, hl_position_id |
| `reconciliation_logs` | id, dimension, expected, actual, deviation_rate, status |
| `routing_config` | symbol, routing_mode (HL_MODE/NORMAL_MODE/BETTING_MODE), normal_threshold, betting_threshold, hl_mode_exposure_threshold, betting_mode_exposure_threshold, auto_switch_enabled, updated_at |
| `routing_mode_logs` | id, from_mode, to_mode, trigger (MANUAL/AUTO), operator_id, net_exposure_at_switch, created_at |
| `risk_config` | key, value, updated_by, updated_at |

## Observability

### Monitoring
- **Prometheus + Grafana**: Business metrics (routing stats, P&L, drift rate, withdrawal SLA, liquidation count)
- **Infrastructure metrics**: CPU, memory, DB connections, Redis hit rate

### Logging
- **ELK (Elasticsearch + Logstash + Kibana)**: Structured logs; searchable by order ID and user ID
- Log levels: DEBUG / INFO / WARN / ERROR

### P0 Alerts (5-minute response SLA)

| Alert | Level | Channel |
|-------|-------|---------|
| Liquidation engine failure | P0 | PagerDuty + Slack |
| HL disconnect > 1 minute | P0 | PagerDuty + Slack |
| HL margin ratio < 150% | P0 | PagerDuty + Slack |
| Hot wallet balance < $100K | P0 | PagerDuty + Slack |
| Single-trade drift rate > 5% | P0 | PagerDuty + Slack |
| User asset reconciliation deviation > 0.1% | P0 | PagerDuty + Slack |

## Security

| Layer | Measure |
|-------|---------|
| API auth | HMAC signature + JWT + API key management |
| Withdrawal protection | 2FA (TOTP) + email confirmation |
| Key management | Turnkey HSM (user deposit addresses + HL Agent Key) |
| Transport | Full TLS encryption |
| Network | CDN + WAF + rate limiting (anti-DDoS) |
| Audit | Immutable audit logs for all sensitive operations |

## Deployment Architecture

| Component | Solution |
|-----------|---------|
| Containerization | Docker + Kubernetes |
| CI/CD | GitHub Actions + canary releases |
| Database | PostgreSQL (primary-replica replication) |
| Cache | Redis Cluster |
| Multi-region | Primary-standby, DB replication |
| Disaster recovery | Degraded mode: all orders routed to HL |

## Disaster Recovery

**HL Connection Interruption (> 1 minute):**
1. Halt new order intake
2. Continue monitoring existing positions (use last valid price)
3. After recovery: full HL position state sync + consistency check

**Platform Service Outage:**
1. Switch to standby region
2. Restore from latest DB snapshot
3. Rebuild Redis cache from DB

**HL Account Liquidation (Catastrophic):**
> Should never happen under normal operations (design target).
1. Immediately halt all HL-related operations
2. Manual audit of all affected users
3. Compensate users from risk reserve + platform funds
