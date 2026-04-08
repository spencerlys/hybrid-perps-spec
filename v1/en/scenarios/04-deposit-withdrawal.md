---
doc_id: scenario-en-deposit-withdrawal
title: Deposit & Withdrawal Scenarios
tags: [deposit, withdrawal, hot-wallet, cold-wallet, aggregation, multi-chain]
version: 1.0
lang: en
updated: 2026-04-08
related-prd: [prd/03-account-assets.md, prd/10-withdrawal-liquidity.md]
---

# Deposit & Withdrawal Scenarios

---

## SC-DW-001: TRON USDT Deposit

**Preconditions:**
- User is registered; system has assigned a unique TRON deposit address (Turnkey-managed)
- User sends $1,000 USDT TRC20 to their deposit address

**Steps:**
1. On-chain block scanner detects incoming transaction
2. Waiting for confirmations (19 required on TRON)
3. 19th confirmation reached
4. Credit user XBIT internal balance: +$1,000
5. Evaluate aggregation: is address balance above aggregation threshold?

**Expected Results:**
- User XBIT account balance +$1,000
- deposits table: status=COMPLETED, confirmations=19
- balance_logs entry: type = "deposit"
- If address balance > aggregation threshold → trigger aggregation to hot wallet

---

## SC-DW-002: Ethereum Deposit — Aggregation to Hot Wallet

**Preconditions:**
- User sends $5,000 USDT ERC20 to their ETH deposit address
- Address balance $5,000 > aggregation threshold $2,000

**Steps:**
1. After 12 ETH confirmations, credit user balance: +$5,000
2. Aggregation evaluation: balance $5,000 > threshold → execute aggregation
3. Platform builds ETH aggregation transaction (consumes ETH for gas)
4. USDT transferred to platform ETH hot wallet

**Expected Results:**
- User balance +$5,000 (**credited in step 1 — independent of aggregation completion**)
- aggregations table entry written
- Platform ETH hot wallet balance +$5,000 (after aggregation completes)
- Gas fees borne by platform

---

## SC-DW-003: Small Auto Withdrawal (< $10K, 5-Min SLA)

**Preconditions:**
- User available balance: $5,000
- User requests $2,000 USDT withdrawal to TRON address
- TRON hot wallet balance: $300,000 (sufficient)

**Steps:**
1. Tier check: $2,000 < $10,000 → auto review
2. Risk rules pass (no blacklist, no frequency anomaly)
3. 2FA verification passes
4. Available balance check: $5,000 ≥ $2,000 ✓
5. Hot wallet balance check: $300,000 sufficient ✓
6. Build TRON USDT transfer transaction
7. Broadcast on-chain

**Expected Results:**
- End-to-end completed within 5 minutes
- User balance -$2,000 (remaining $3,000)
- withdrawals table: status=COMPLETED, processed_at < T+5min
- Hot wallet balance -$2,000

---

## SC-DW-004: Large Withdrawal — Manual Approval Flow

**Preconditions:**
- User balance: $200,000
- User requests $150,000 USDT withdrawal

**Steps:**
1. Tier check: $150,000 > $100,000 → risk review + manual approval
2. Risk team receives pending approval notification in admin panel
3. Manual review: check user KYC, source of funds
4. Approval granted → execute transfer

**Expected Results:**
- End-to-end target: < 2 hours
- Approval written to audit_log (approver, timestamp, IP)
- Transfer complete: withdrawals.status=COMPLETED

---

## SC-DW-005: Hot Wallet Insufficient — Trigger Cold Wallet Top-Up

**Preconditions:**
- ETH hot wallet balance: $80,000 (below critical threshold $100,000)
- User requests $50,000 ETH withdrawal

**Steps:**
1. Hot wallet balance check: $80,000 insufficient (must maintain minimum level)
2. System triggers cold wallet top-up flow
3. Generate cold wallet transfer proposal (amount = target level - current balance)
4. Risk officer approves offline (cold wallet requires offline signing)
5. Approval granted → broadcast on-chain → await confirmation
6. Hot wallet replenished → resume user withdrawal

**Expected Results:**
- User withdrawal queued; system notifies user of processing delay
- After cold wallet top-up completes, withdrawal resumes automatically
- Full audit trail maintained throughout

---

## SC-DW-006: Withdrawal Risk Intercept — Frequency Anomaly

**Preconditions:**
- User has made 5 withdrawals to the same address in the past 24 hours
- User initiates a 6th withdrawal of $500 to the same address

**Steps:**
1. Tier check: $500 < $10,000 → auto review
2. Risk rule: same address > 3 withdrawals in 24h → frequency anomaly
3. Auto-flag as suspicious; escalate to manual review

**Expected Results:**
- Withdrawal status → PENDING_MANUAL_REVIEW
- Risk team receives alert notification
- User's $500 temporarily frozen in account
- If manual review approves → proceed with transfer; if rejected → release frozen balance
