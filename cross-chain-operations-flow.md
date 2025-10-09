# Makina Protocol - Cross-Chain Operations Flow

```mermaid
graph TB
    subgraph "Hub Chain (Ethereum)"
        Machine[Machine<br/>Strategy Vault]
        HubCaliber[Hub Caliber<br/>Execution Engine]
        HubMailbox[Hub Mailbox<br/>Bridge Controller]
        HubBridgeAdapter[Bridge Adapter<br/>Protocol Interface]
    end
    
    subgraph "Cross-Chain Infrastructure"
        ExternalBridge[External Bridge<br/>Wormhole/Hyperlane/CCIP]
        WormholeNetwork[Wormhole CCQ Network<br/>Guardian Validators]
    end
    
    subgraph "Spoke Chain (Arbitrum/Base/etc)"
        SpokeBridgeAdapter[Bridge Adapter<br/>Protocol Interface]
        SpokeMailbox[Caliber Mailbox<br/>Access Control]
        SpokeCaliber[Spoke Caliber<br/>Execution Engine]
        
        subgraph "Caliber Assets"
            BaseTokens[Base Tokens<br/>USDC, ETH, etc]
            Positions[Positions<br/>Deployed Assets]
        end
        
        subgraph "MakinaVM"
            Instructions[Instructions<br/>Deploy/Close/Account]
            MerkleProof[Merkle Proof<br/>Verification]
        end
        
        SpokeCaliber --> BaseTokens
        SpokeCaliber --> Positions
        SpokeCaliber --> MakinaVM
    end
    
    subgraph "DeFi Protocols"
        Aave[Aave Lending]
        Uniswap[Uniswap V3 LP]
        Compound[Compound]
    end
    
    Machine --> HubCaliber
    HubCaliber --> HubMailbox
    HubMailbox --> HubBridgeAdapter
    HubBridgeAdapter --> ExternalBridge
    ExternalBridge --> SpokeBridgeAdapter
    SpokeBridgeAdapter --> SpokeMailbox
    SpokeMailbox --> SpokeCaliber
    
    SpokeCaliber --> |Deploy Assets| Aave
    SpokeCaliber --> |Deploy Assets| Uniswap
    SpokeCaliber --> |Deploy Assets| Compound
    
    SpokeCaliber --> |Accounting Data| WormholeNetwork
    WormholeNetwork --> |Signed AUM| Machine
    
    style Machine fill:#e1f5ff
    style SpokeCaliber fill:#fff4e1
    style WormholeNetwork fill:#e8f5e9
    style ExternalBridge fill:#e8f5e9
```

## Liquidity Bridging Flow

```mermaid
sequenceDiagram
    actor Operator
    participant Machine
    participant HubMailbox
    participant BridgeAdapter
    participant ExternalBridge
    participant SpokeMailbox
    participant SpokeCaliber
    
    Note over Operator,SpokeCaliber: Hub → Spoke: Outbound Transfer
    
    Operator->>Machine: 1. Schedule outbound transfer
    activate Machine
    Machine->>Machine: Generate message hash
    Machine->>HubMailbox: Forward transfer request
    deactivate Machine
    
    activate HubMailbox
    HubMailbox->>SpokeMailbox: 2. Authorize transfer (message hash)
    activate SpokeMailbox
    SpokeMailbox->>SpokeMailbox: Register incoming message hash
    SpokeMailbox-->>HubMailbox: Authorization confirmed
    deactivate SpokeMailbox
    deactivate HubMailbox
    
    Operator->>HubMailbox: 3. Send transfer via bridge
    activate HubMailbox
    HubMailbox->>BridgeAdapter: Execute bridge protocol
    activate BridgeAdapter
    BridgeAdapter->>ExternalBridge: Lock/burn tokens
    activate ExternalBridge
    ExternalBridge-->>BridgeAdapter: Transfer initiated
    deactivate ExternalBridge
    deactivate BridgeAdapter
    deactivate HubMailbox
    
    Note over ExternalBridge: Cross-chain message relay<br/>Time: minutes to hours
    
    ExternalBridge->>SpokeMailbox: Relay message
    
    Operator->>SpokeMailbox: 4. Claim transfer
    activate SpokeMailbox
    SpokeMailbox->>SpokeMailbox: Verify message hash
    SpokeMailbox->>SpokeCaliber: Unlock/mint tokens
    activate SpokeCaliber
    SpokeCaliber->>SpokeCaliber: Update balance
    SpokeCaliber-->>SpokeMailbox: Tokens received
    deactivate SpokeCaliber
    SpokeMailbox-->>Operator: Transfer complete
    deactivate SpokeMailbox
    
    Note over Operator,SpokeCaliber: Spoke → Hub: Return Transfer (Symmetric)
    
    Operator->>SpokeCaliber: 1. Schedule return transfer
    activate SpokeCaliber
    SpokeCaliber->>SpokeMailbox: Transfer tokens
    deactivate SpokeCaliber
    
    activate SpokeMailbox
    SpokeMailbox->>HubMailbox: 2. Authorize incoming
    SpokeMailbox->>ExternalBridge: 3. Send via bridge
    deactivate SpokeMailbox
    
    ExternalBridge->>HubMailbox: Relay message
    
    Operator->>HubMailbox: 4. Claim at Hub
    activate HubMailbox
    HubMailbox->>Machine: Transfer to Machine
    activate Machine
    Machine->>Machine: Update idle balance
    deactivate Machine
    HubMailbox-->>Operator: Transfer complete
    deactivate HubMailbox
```

## Cross-Chain Accounting Flow

```mermaid
sequenceDiagram
    actor Operator
    participant SpokeCaliber1 as Spoke Caliber<br/>(Arbitrum)
    participant SpokeCaliber2 as Spoke Caliber<br/>(Base)
    participant WormholeGuardians as Wormhole Guardians<br/>(Validator Network)
    participant Machine
    participant OracleRegistry
    
    Note over Operator,OracleRegistry: Step 1: Update Caliber Accounting
    
    Operator->>SpokeCaliber1: Execute accounting instructions
    activate SpokeCaliber1
    
    SpokeCaliber1->>OracleRegistry: Get token prices
    activate OracleRegistry
    OracleRegistry-->>SpokeCaliber1: Return prices
    deactivate OracleRegistry
    
    SpokeCaliber1->>SpokeCaliber1: Calculate position values<br/>Value = tokens × price
    
    SpokeCaliber1->>SpokeCaliber1: Update local AUM<br/>AUM = Σ base_tokens + Σ positions
    
    SpokeCaliber1-->>Operator: Accounting updated
    deactivate SpokeCaliber1
    
    Operator->>SpokeCaliber2: Execute accounting instructions
    activate SpokeCaliber2
    SpokeCaliber2->>SpokeCaliber2: Update local AUM
    SpokeCaliber2-->>Operator: Accounting updated
    deactivate SpokeCaliber2
    
    Note over Operator,OracleRegistry: Step 2: Cross-Chain Query (Wormhole CCQ)
    
    Operator->>WormholeGuardians: Request CCQ for all Spokes
    activate WormholeGuardians
    
    WormholeGuardians->>SpokeCaliber1: Query getAccountingData()
    activate SpokeCaliber1
    SpokeCaliber1-->>WormholeGuardians: Return AUM data
    deactivate SpokeCaliber1
    
    WormholeGuardians->>SpokeCaliber2: Query getAccountingData()
    activate SpokeCaliber2
    SpokeCaliber2-->>WormholeGuardians: Return AUM data
    deactivate SpokeCaliber2
    
    WormholeGuardians->>WormholeGuardians: Validators sign data<br/>(Quorum required)
    
    WormholeGuardians-->>Operator: Return signed accounting data
    deactivate WormholeGuardians
    
    Note over Operator,OracleRegistry: Step 3: Submit to Machine
    
    Operator->>Machine: Submit signed Spoke accounting
    activate Machine
    
    Machine->>Machine: Verify Wormhole signatures
    
    Machine->>Machine: Calculate total AUM:<br/>Total = Hub_AUM + Spoke1_AUM + Spoke2_AUM + Pending_bridges
    
    Machine->>Machine: Update share price:<br/>Price = Total_AUM / Share_supply
    
    Machine-->>Operator: Machine AUM updated
    deactivate Machine
```

## Position Management on Spoke Chain

```mermaid
sequenceDiagram
    actor Operator
    participant SpokeCaliber
    participant MakinaVM
    participant OracleRegistry
    participant Aave
    participant Uniswap
    
    Note over Operator,Uniswap: Deploy Assets to Positions
    
    Operator->>SpokeCaliber: Execute position management<br/>(with Merkle proof)
    activate SpokeCaliber
    
    SpokeCaliber->>SpokeCaliber: Check cooldown period
    
    SpokeCaliber->>MakinaVM: Verify instruction
    activate MakinaVM
    MakinaVM->>MakinaVM: Verify Merkle proof<br/>against stored root
    MakinaVM-->>SpokeCaliber: Instruction valid
    deactivate MakinaVM
    
    SpokeCaliber->>OracleRegistry: Get value before action
    activate OracleRegistry
    OracleRegistry-->>SpokeCaliber: Total value = $X
    deactivate OracleRegistry
    
    SpokeCaliber->>Aave: Deploy 100,000 USDC
    activate Aave
    Aave->>Aave: Deposit to lending pool
    Aave-->>SpokeCaliber: Receive aUSDC
    deactivate Aave
    
    SpokeCaliber->>Uniswap: Deploy ETH-USDC LP
    activate Uniswap
    Uniswap->>Uniswap: Add liquidity
    Uniswap-->>SpokeCaliber: Receive LP NFT
    deactivate Uniswap
    
    SpokeCaliber->>OracleRegistry: Get value after action
    activate OracleRegistry
    OracleRegistry-->>SpokeCaliber: Total value = $Y
    deactivate OracleRegistry
    
    SpokeCaliber->>SpokeCaliber: Check max loss:<br/>Loss = |X - Y| < max_loss_cap
    
    alt Loss within limit
        SpokeCaliber->>SpokeCaliber: Update position balances
        SpokeCaliber->>SpokeCaliber: Start cooldown timer
        SpokeCaliber-->>Operator: Position management complete
    else Loss exceeds limit
        SpokeCaliber-->>Operator: ❌ Transaction reverted<br/>Loss exceeded limit
    end
    
    deactivate SpokeCaliber
    
    Note over Operator,Uniswap: Harvest and Swap Rewards
    
    Operator->>SpokeCaliber: Execute harvest instruction
    activate SpokeCaliber
    
    SpokeCaliber->>Aave: Claim rewards
    activate Aave
    Aave-->>SpokeCaliber: Transfer AAVE tokens
    deactivate Aave
    
    SpokeCaliber->>Uniswap: Harvest fees
    activate Uniswap
    Uniswap-->>SpokeCaliber: Transfer fee tokens
    deactivate Uniswap
    
    SpokeCaliber->>SpokeCaliber: Check cooldown for swaps
    
    SpokeCaliber->>OracleRegistry: Get value before swap
    SpokeCaliber->>SpokeCaliber: Swap AAVE → USDC<br/>(via aggregator)
    SpokeCaliber->>OracleRegistry: Get value after swap
    
    SpokeCaliber->>SpokeCaliber: Verify slippage within bounds
    
    SpokeCaliber->>SpokeCaliber: Convert to base token
    SpokeCaliber-->>Operator: Harvest complete
    deactivate SpokeCaliber
```

## Key Cross-Chain Components

### 1. Liquidity Bridging (4-Step Process)

**Bidirectional flow: Hub ↔ Spoke**

| Step | Action | Location | Purpose |
|------|--------|----------|---------|
| 1 | Schedule | Sender | Generate message hash |
| 2 | Authorize | Recipient | Whitelist incoming transfer |
| 3 | Send | Sender | Execute bridge protocol |
| 4 | Claim | Recipient | Finalize transfer |

**Security Features:**
- Two-step authorization (schedule + authorize)
- Message hash verification
- Max loss caps on bridge transfers
- Only pre-approved bridges allowed

---

### 2. Cross-Chain Accounting (Wormhole CCQ)

**Purpose**: Aggregate AUM from all chains to Hub Machine

**Process:**
1. **Local Accounting**: Each Spoke Caliber maintains local AUM
   ```
   Caliber_AUM = Σ base_tokens + Σ position_values
   ```

2. **CCQ Request**: Operator requests Wormhole to query all Spokes

3. **Data Collection**: Wormhole guardians:
   - Call `getAccountingData()` on each Spoke Caliber
   - Collect AUM, positions, base tokens
   - Sign data with validator quorum

4. **Submission**: Signed data submitted to Machine on Hub

5. **Aggregation**: Machine calculates total:
   ```
   Total_AUM = Hub_AUM + Σ Spoke_AUM + Pending_bridges
   ```

6. **Share Price Update**:
   ```
   Share_Price = Total_AUM / Share_Supply
   ```

---

### 3. Caliber Mailbox Functions

**Access Control:**
- Specifies authorized operators per chain
- Prevents unauthorized cross-chain calls
- Chain-specific permission management

**Bridge Management:**
- Interface with multiple bridge adapters
- Schedule and claim transfers
- Verify message hashes

**Accounting Interface:**
- Expose `getAccountingData()` for Wormhole
- Return detailed AUM breakdown
- Include position values and base token balances

---

### 4. Position Management

**MakinaVM Execution:**
- Verify Merkle proof for instruction
- Execute pre-approved function calls
- Atomic transaction batching

**Safety Checks:**
- **Cooldown Period**: Minimum time between position changes
- **Max Loss Cap**: Value loss limit per transaction
- **Oracle Verification**: Before/after value comparison

**Instruction Types:**
- **Deploy**: Open or increase position
- **Close**: Decrease or exit position
- **Account**: Update position value
- **Harvest**: Claim rewards

---

### 5. Oracle Integration

**Price Feeds:**
- Chainlink oracles on each chain
- Direct or hop-based pricing (Token A → USD → Token B)
- Consistent pricing across chains

**Usage:**
- Value positions in accounting token
- Verify swap slippage
- Calculate AUM for each Caliber
- Ensure accurate share price

---

## Recovery Mode Effects on Cross-Chain

### During Recovery Mode:

**Hub Chain:**
- ❌ No outbound bridges (Hub → Spoke blocked)
- ✅ Inbound bridges allowed (Spoke → Hub)
- ❌ Deposits disabled
- ❌ AUM updates paused

**Spoke Chains:**
- ❌ Positions can only decrease (no new deployments)
- ✅ Swaps only to accounting token
- ✅ Bridge assets back to Hub
- ❌ No outbound to other Spokes

**Purpose**: Consolidate assets to Hub during emergency

---

## Bridge Adapters

**Multi-Bridge Support:**
- Wormhole NTT (Native Token Transfer)
- Hyperlane
- Chainlink CCIP
- LayerZero
- Custom implementations

**Adapter Interface:**
- Standardized send/receive functions
- Protocol-specific implementations
- Governance-approved only

**Benefits:**
- Redundancy (multiple bridge options)
- Optimization (choose best for each route)
- Risk diversification
