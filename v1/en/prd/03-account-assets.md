---
doc_id: prd-en-account-assets
title: L1 Account & Assets Layer — Registration, Deposits, Withdrawals, Aggregation
tags: [account, wallet, deposit, withdrawal, aggregation, multi-chain, L1, turnkey]
version: 1.0
lang: en
updated: 2026-04-08
phase: Phase 1
---

# L1 Account & Assets Layer

## User Registration & Login

- **Web2**: Email/phone + OTP verification
- **Web3**: MetaMask/WalletConnect wallet signature
- Each user gets an independent account managed by **Turnkey HSM**
- Session: JWT + refresh token
- KYC/AML interface reserved; not implemented in MVP

## Wallet & Balance Model

### Balance Fields

| Field | Description |
|-------|-------------|
| total_balance | Total account balance |
| available_balance | Available for trading / withdrawal |
| frozen_margin | Frozen margin for isolated positions |
| cross_margin_used | Margin consumed by cross positions |

### Available Balance Formula

```
Available = Wallet Balance
          - Σ(isolated position margins)
          - Σ(cross position initial margins)
          + Σ(cross position unrealized PnL, positive values only)
```

### Balance Change Logging

Every balance mutation must record its type: deposit, withdrawal, open-freeze, close-release, liquidation, funding fee, trading fee.

## Deposits

### Supported Chains & Confirmations

| Chain | Asset | Confirmations | Priority |
|-------|-------|--------------|----------|
| TRON | USDT TRC20 | 19 | P0 |
| Ethereum | USDT/USDC ERC20 | 12 | P0 |
| Solana | USDT/USDC SPL | 32 slots | P1 |

### Deposit Flow

1. Each user gets a unique deposit address per chain (Turnkey-managed private key)
2. On-chain block scanner detects incoming transactions
3. After required confirmations: credit user's XBIT internal balance
4. Trigger aggregation evaluation

## Multi-Chain Asset Aggregation

### Aggregation Triggers

| Trigger | Description |
|---------|-------------|
| Single address balance > threshold | Avoid small-amount high-gas waste; threshold configured per chain |
| Scheduled scan (every 4 hours) | Periodic sweep of all user deposit addresses |
| Emergency aggregation | Triggered when hot wallet balance falls below safety threshold |

### Aggregation Flow

```
User deposit addresses (per chain)
  └─► [Aggregation tx] ──► Platform hot wallet (one per chain)
                              ├─► User withdrawals
                              ├─► HL operating capital top-up (cross-chain to Arbitrum)
                              └─► [High watermark] ──► Cold wallet
```

### Gas Management

| Chain | Gas Asset | Strategy |
|-------|-----------|----------|
| TRON | TRX | Pre-fund TRX to each deposit address |
| Ethereum | ETH | Pre-fund ETH to each deposit address |
| Solana | SOL | Pre-fund SOL for rent + transaction fees |

Gas fees are borne by the platform and charged to operating costs.

## Hot Wallet / Cold Wallet Management

### Hot Wallet Level Management

| Status | Condition | Action |
|--------|-----------|--------|
| Safe | > $500K | Normal operations |
| Warning | $100K – $500K | Trigger aggregation + cold wallet top-up |
| Critical | < $100K | Pause large withdrawals + emergency top-up |

### Cold Wallet Top-Up Process

Requires offline manual signing approval:
1. System generates transfer proposal
2. Risk officer approves offline
3. Broadcast transaction on-chain

## Withdrawals

See [10-withdrawal-liquidity.md](10-withdrawal-liquidity.md) for full details.

### Withdrawal Tiers

| Amount | Review Method | SLA |
|--------|---------------|-----|
| < $10,000 | Auto review + auto send | < 5 minutes |
| $10K – $100K | Auto review + manual confirm | < 30 minutes |
| > $100K | Risk review + manual approval | < 2 hours |
