# Makina Protocol - Architecture and Flow Diagrams

## Overview

This repository contains comprehensive architecture diagrams and user flow documentation for the **Makina Protocol** - a highly agile, cross-chain DeFi infrastructure for executing sophisticated yield strategies in a non-custodial, transparent, and trust-minimized manner.

## 📋 Contents

1. **[System Architecture](#system-architecture)** - Complete protocol architecture
2. **[User Deposit Flow](#user-deposit-flow)** - How users deposit and receive shares
3. **[User Redemption Flow](#user-redemption-flow)** - Asynchronous redemption process
4. **[Cross-Chain Operations](#cross-chain-operations)** - Liquidity bridging and accounting
5. **[Governance & Security](#governance--security)** - Root updates and recovery mode

---

## 📐 System Architecture

**File:** [`architecture-diagram.md`](./architecture-diagram.md)

### Key Components

#### Machine (Hub Chain)
- **Depositor**: Entry point for user deposits
- **AsyncRedeemer**: ERC-721 based redemption queue (FIFO)
- **Machine Core**: AUM calculation and share price management
- **Machine Token**: ERC-20 tokenized strategy shares
- **Fee Manager**: Fixed and performance fee processing
- **Security Module**: Insurance pool with staking and slashing

#### Calibers (Execution Engines)
- **Hub Caliber**: Execution engine on the main chain
- **Spoke Calibers**: Execution engines on external chains (Arbitrum, Base, etc.)
- **MakinaVM**: Virtual machine executing pre-approved instructions
- **Positions**: Deployed assets in external DeFi protocols
- **Base Tokens**: Liquid tokens held by Caliber

#### Cross-Chain Infrastructure
- **Wormhole CCQ**: Cross-chain queries for aggregating accounting data
- **NTT (Native Token Transfer)**: For bridging Machine tokens
- **Bridge Adapters**: Support for multiple bridge protocols
- **Caliber Mailbox**: Communication hub for each chain

#### Governance Layer
- **Operator**: Daily strategy management and execution
- **Risk Manager**: Controls risk parameters and instruction whitelisting
- **Security Council**: Emergency override and veto powers
- **Root Guardians**: Veto authority for Merkle root changes
- **Timelock**: Minimum 7-day delay for transparency

---

## 💰 User Deposit Flow

**File:** [`user-deposit-flow.md`](./user-deposit-flow.md)

### Standard Deposit Process

1. **User deposits Accounting Tokens** to Depositor contract
2. **Tokens transferred** to Machine
3. **Oracle queries** for current token prices
4. **AUM aggregation** from all Calibers (Hub + Spokes + pending bridges)
5. **Share price calculation**: `SharePrice = Total_AUM / Total_Supply`
6. **Fee processing**: Fixed fees (AUM-based) and performance fees (watermark)
7. **Share minting**: `Shares = Deposit_Amount / Share_Price`
8. **Machine Tokens distributed** to user

### Pre-Deposit Flow

**Before strategy launch:**
- Users deposit yield-bearing tokens (e.g., stETH, aUSDC)
- PreDepositVault mints future Machine Shares
- Users earn baseline yield during pre-deposit
- **Seamless transition**: No migration needed at launch
- Minting authority transfers to Machine

---

## 🔄 User Redemption Flow

**File:** [`user-redemption-flow.md`](./user-redemption-flow.md)

### Why Asynchronous?

Assets are deployed into illiquid positions across chains. Atomic (instant) redemptions are impossible. **Solution**: Three-step FIFO queue with NFT-based claims.

### Three Steps

#### Step 1: Request Redemption (User)
- User locks Machine Shares in AsyncRedeemer queue
- Receives **ERC-721 NFT** representing redemption request
- NFT contains: Request ID, shares locked, timestamp, status
- Added to **FIFO queue**

#### Step 2: Settlement (Operator)
- Operator prepares liquidity (closes positions, harvests, swaps, bridges)
- Settles redemption queue (batch or individual)
- Machine burns shares at **current share price**
- Accounting Tokens transferred to queue
- Request marked as "settled"

#### Step 3: Claim Funds (User)
- User checks settlement status
- Burns redemption NFT
- Receives Accounting Tokens
- **Unlimited time to claim** after settlement

### Benefits
- ✅ Enables illiquid strategies
- ✅ Capital efficient (no large idle buffers)
- ✅ Fair pricing (settlement at current price)
- ✅ Flexible timing
- ✅ NFT transferability

---

## 🌐 Cross-Chain Operations

**File:** [`cross-chain-operations-flow.md`](./cross-chain-operations-flow.md)

### Liquidity Bridging (4-Step Process)

Bidirectional flow: **Hub ↔ Spoke**

1. **Schedule**: Generate message hash on sender chain
2. **Authorize**: Whitelist incoming transfer on recipient chain
3. **Send**: Execute bridge protocol (lock/burn tokens)
4. **Claim**: Finalize transfer on recipient chain (unlock/mint tokens)

**Security**: Two-step authorization, message hash verification, max loss caps

### Cross-Chain Accounting (Wormhole CCQ)

**Purpose**: Aggregate AUM from all chains

1. **Local Accounting**: Each Spoke Caliber maintains AUM
2. **CCQ Request**: Operator requests Wormhole to query all Spokes
3. **Data Collection**: Wormhole guardians query and sign data
4. **Submission**: Signed data submitted to Machine on Hub
5. **Aggregation**: `Total_AUM = Hub_AUM + Σ Spoke_AUM + Pending_bridges`
6. **Share Price Update**: `Share_Price = Total_AUM / Share_Supply`

### Position Management

**MakinaVM Execution:**
- Verify Merkle proof for instruction
- Execute pre-approved function calls
- Atomic transaction batching

**Safety Checks:**
- **Cooldown Period**: Minimum time between changes
- **Max Loss Cap**: Value loss limit per transaction
- **Oracle Verification**: Before/after value comparison

**Instruction Types:**
- Deploy (open/increase position)
- Close (decrease/exit position)
- Account (update position value)
- Harvest (claim rewards)

### Recovery Mode Effects

**Hub Chain:**
- ❌ Deposits disabled
- ❌ Outbound bridges blocked
- ✅ Inbound bridges allowed

**Spoke Chains:**
- ❌ Positions can only decrease
- ❌ Swaps only to accounting token
- ✅ Bridge assets back to Hub

---

## 🛡️ Governance & Security

**File:** [`governance-security-flow.md`](./governance-security-flow.md)

### Root Update Lifecycle

**Multi-layer review process for Merkle Root updates:**

1. **Operator submits PR** to public GitHub repo
2. **Security Council & Devs review** and approve
3. **PR merged** to main branch
4. **Risk Manager schedules** on-chain update
5. **Timelock starts** (minimum 7 days)
6. **Community notification** (Discord, Telegram)
7. **Veto period**: Security Council and Root Guardians can cancel
8. **Execution**: If no veto, new root becomes active

### Recovery Mode

**Triggered by Security Council during emergencies:**

**Detection Phase:**
- Suspicious activity (price drops, unusual transactions)
- Operator inactivity
- Hacks or exploits

**Emergency Actions:**
1. Security Council activates Recovery Mode
2. **Operator permissions revoked** → transferred to Council
3. **Restrictions applied**:
   - Hub: No deposits, no outbound bridges
   - Spokes: Only decrease positions, swap to accounting token only
4. **Recovery actions**: Close positions, consolidate assets to Hub
5. **Optional slashing**: Burn operator stake if needed
6. **Exit Recovery**: When situation stabilized

### Security Module

**Insurance pool protecting users:**

**Staking:**
- Users lock Machine Tokens
- Receive Security Tokens
- Earn boosted yields (protocol fees)

**Unstaking:**
- Request unstake → receive Receipt NFT
- **Cooldown period** (minimum 7 days)
- Can cancel during cooldown
- After cooldown: burn NFT, receive original tokens

**Slashing:**
- If loss event occurs, Security Tokens burned (up to limit)
- Burned value covers Machine losses
- **Max slashing limits** protect stakers

### Safe Multisig Security

**Five security layers:**
1. **M-of-N threshold**: Multiple signatures required
2. **Guard contract**: Only authorized owners execute
3. **Hypernative Guardian**: Detects and blocks malicious payloads
4. **Ownership timelock**: 24-hour delay for ownership changes
5. **Non-upgradable**: Fixed Safe implementation (v1.4.1)

---

## 🔑 Key Protocol Features

### Non-Custodial & Trust-Minimized
- Users maintain custody via Machine Tokens
- Operator restricted by whitelisted instructions
- Multi-layer oversight and veto powers

### Transparent
- All changes visible on-chain
- Public GitHub for instruction updates
- Timelock transparency window
- Open accounting via Wormhole CCQ

### Flexible & Agile
- Support for any on-chain strategy
- Easy protocol integrations (Merkle root update)
- Multiple bridge support
- Operator specialization

### Secure
- Multiple audits (Enigma Dark, SigmaPrime, ChainSecurity)
- Security Module insurance
- Recovery Mode emergency system
- Hypernative monitoring
- Max loss caps and cooldowns

---

## 📊 Protocol Flow Summary

```
Users Deposit
    ↓
Machine Mints Shares
    ↓
Operator Deploys to Calibers
    ↓
Calibers Execute on DeFi Protocols
    ↓
Harvest Rewards & Compound
    ↓
Cross-Chain Accounting via Wormhole
    ↓
Machine Updates Share Price
    ↓
Users Redeem (Async Queue)
    ↓
Operator Settles Redemptions
    ↓
Users Claim Funds
```

---

## 🏗️ Protocol Architecture Highlights

### Multi-Chain Support
- **Hub Chain**: Ethereum Mainnet (primary)
- **Spoke Chains**: Arbitrum, Base, Optimism, Polygon, Solana (via Wormhole)
- **Bridge Support**: Wormhole, Hyperlane, CCIP, LayerZero

### Strategy Types Supported
- On-chain yield aggregation
- Index products
- Long-short strategies
- Delta-hedged strategies
- Any fully on-chain executable strategy

### Fee Structure
**Fixed Fees** (AUM-based):
- Security Module share
- Operator share
- Makina DAO share

**Performance Fees** (high watermark):
- Operator share
- Makina DAO share

---

## 🎯 Quick Reference

| Component | Purpose | Location |
|-----------|---------|----------|
| Machine | Strategy vault, share management | Hub Chain |
| Caliber | Execution engine, position management | Each chain |
| MakinaVM | Instruction execution | Inside Caliber |
| Mailbox | Cross-chain communication | Each chain |
| Wormhole CCQ | Cross-chain accounting | Off-chain network |
| Security Module | Insurance pool | Hub Chain |
| Timelock | Governance delays | Hub Chain |
