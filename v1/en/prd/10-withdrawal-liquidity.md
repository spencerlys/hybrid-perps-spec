---
doc_id: prd-en-withdrawal-liquidity
title: Withdrawal Liquidity Guarantee — 5-Minute SLA, Tiered Processing, Monitoring
tags: [withdrawal, liquidity, hot-wallet, cold-wallet, monitoring, 5-minutes]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 1
---

# Withdrawal Liquidity Guarantee

## Core Goal

**5-minute on-chain arrival guarantee for standard withdrawals under $10,000.**

## Liquidity Guarantee Mechanism

### Key Metric: Net User Deposits

```
User Net Deposit = Σ(all deposits) - Σ(all withdrawals)
Platform Total Liability = Σ(all user wallet balances)
```

`Platform Total Liability` = total funds the platform owes users (sum of all XBIT internal balances).

### Liquidity Safety Ratio

```
Sum of hot wallet balances ≥ Platform Total Liability × Safety Ratio (recommend 30%)
```

100% coverage is not required — not all users withdraw simultaneously. The ratio is adjusted dynamically based on historical withdrawal peak data.

## Withdrawal Tier Processing

| Amount | Review Method | SLA | Notes |
|--------|---------------|-----|-------|
| < $10,000 | Auto review + auto send | < 5 min | Fully automatic, 7×24 |
| $10K – $100K | Auto risk review + manual confirm | < 30 min | Auto rules pass → human final approval |
| > $100K | Full manual review + approval | < 2 hours | Full manual process |

### Auto Review Rules (< $10K)

Auto-reject or escalate to manual review if:
- Same address used > 3 times in 24 hours (frequency anomaly)
- Withdrawal address is blacklisted
- Abnormal login records on account
- Hot wallet balance below critical threshold

### Withdrawal Flow

```
User initiates withdrawal
  │
  ▼
Tier classification → auto review / manual review
  │
  ▼
2FA verification (TOTP or email confirmation)
  │
  ▼
Available balance check
  │
  ▼
Hot wallet balance check (per chain)
  │
  ├── Sufficient ──► Build tx → broadcast → await confirmation → complete
  │
  └── Insufficient ──► Trigger cold wallet top-up process
                        (requires offline approval; may delay)
                        → Resume withdrawal after top-up
```

## Hot Wallet Liquidity Alerts

Independent monitoring per chain:

| Status | Condition | Action |
|--------|-----------|--------|
| Safe | > $500K | Normal |
| Warning | $100K – $500K | Trigger aggregation + notify ops to replenish |
| Critical | < $100K | Pause withdrawals > $10K + alert + emergency cold wallet transfer |
| Emergency | < $20K | Halt all withdrawals + P0 alert |

## Monitoring Metrics

| Metric | Description | Alert Condition |
|--------|-------------|----------------|
| Per-chain hot wallet balance | Real-time | < $100K |
| Liquidity coverage ratio | Hot wallet / total liability | < 30% |
| Pending withdrawal queue | Current backlog | > 50 requests or > $500K |
| Avg processing time | 24h rolling average (< $10K tier) | > 3 minutes |
| Daily withdrawal volume | Intraday total | > 3× historical daily average |

## HL Account & Withdrawal Relationship

The HL account holds **platform operating capital**, not user withdrawal funds:

```
Normal:
  User withdrawal ──► Hot wallet (direct)

Extreme (last resort — both hot and cold wallets exhausted):
  → Extract from HL platform account
  → HL withdrawal via Arbitrum (hour-level delay)
  → Must never be triggered under normal operations
```

## Aggregation & Withdrawal Linkage

```
User deposit → balance credited → aggregation evaluation
                                         ↓
                              Balance > threshold → aggregate to hot wallet
                              Hot wallet sufficient → guarantee withdrawals
```

Important: Aggregation delay does not affect user's displayed **account balance**, but does affect the hot wallet's actual spendable balance. The system must distinguish between "book balance" and "available hot wallet funds."
