# Makina Protocol - Governance and Security Flow

```mermaid
graph TB
    subgraph "Protocol-Wide Governance"
        AccessManager[Access Manager<br/>Global Roles]
        
        STRATEGY_DEPLOYER[STRATEGY_DEPLOYER_ROLE<br/>Deploy new strategies]
        STRATEGY_CONFIG[STRATEGY_CONFIG_ROLE<br/>Assign/update managers]
        INFRA_CONFIG[INFRA_CONFIG_ROLE<br/>Configure core contracts]
        
        AccessManager --> STRATEGY_DEPLOYER
        AccessManager --> STRATEGY_CONFIG
        AccessManager --> INFRA_CONFIG
    end
    
    subgraph "Per-Strategy Governance"
        subgraph "Management Entities"
            Operator[Operator<br/>Daily Operations<br/>Fee: Performance-based]
            RiskManager[Risk Manager<br/>Risk Parameters<br/>Review Instructions]
            SecurityCouncil[Security Council<br/>Emergency Override<br/>Veto Power]
            RootGuardians[Root Guardians<br/>Merkle Root Veto<br/>Final Authority]
        end
        
        Timelock[Timelock Contract<br/>Min 7-day delay<br/>Transparency window]
        
        RiskManager --> |Schedule changes| Timelock
        SecurityCouncil --> |Monitor & veto| Timelock
        RootGuardians --> |Veto root updates| Timelock
        Timelock --> |Execute after delay| Strategy
    end
    
    subgraph "Strategy Components"
        Strategy[Machine + Calibers]
        MerkleRoot[Merkle Root<br/>Allowed Instructions]
        RiskParameters[Risk Parameters<br/>Loss caps, cooldowns]
        
        Strategy --> MerkleRoot
        Strategy --> RiskParameters
    end
    
    subgraph "Safe Multisig Security"
        SafeOwners[Safe Owners<br/>Multiple signers]
        SafeGuard[Guard Contract<br/>Permissioned execution]
        Hypernative[Hypernative Guardian<br/>Malicious payload detection]
        
        SafeOwners --> SafeGuard
        SafeGuard --> Hypernative
        Hypernative --> |Block malicious tx| SafeOwners
    end
    
    subgraph "Monitoring"
        MakinaFoundation[Makina Foundation<br/>Active monitoring]
        CommunityDiscord[Community<br/>Discord/Telegram alerts]
        
        MakinaFoundation --> |Detect anomalies| SecurityCouncil
        CommunityDiscord --> |Report issues| SecurityCouncil
    end
    
    Operator --> |Execute instructions| Strategy
    RiskManager --> |Update parameters| RiskParameters
    RiskManager --> |Update root| MerkleRoot
    SecurityCouncil --> |Recovery Mode| Strategy
    
    SafeOwners -.->|Implements| Operator
    SafeOwners -.->|Implements| RiskManager
    SafeOwners -.->|Implements| SecurityCouncil
    SafeOwners -.->|Implements| RootGuardians
    
    style Operator fill:#c8e6c9
    style RiskManager fill:#fff9c4
    style SecurityCouncil fill:#ffccbc
    style RootGuardians fill:#f8bbd0
    style Timelock fill:#e1bee7
    style Strategy fill:#b3e5fc
    style Hypernative fill:#ffcdd2
```

## Root Update Lifecycle

```mermaid
sequenceDiagram
    actor Operator
    participant GitHub as GitHub Repo<br/>makina-merkletree
    participant SecurityCouncil as Security Council
    participant MakinaDev as Makina Developers
    participant RiskManager as Risk Manager
    participant Timelock
    participant RootGuardians as Root Guardians
    participant Community
    participant Caliber
    
    Note over Operator,Caliber: Phase 1: Proposal Submission
    
    Operator->>GitHub: 1. Submit Pull Request<br/>New instructions + Merkle Root
    activate GitHub
    
    GitHub->>SecurityCouncil: 2. Review PR
    activate SecurityCouncil
    SecurityCouncil->>SecurityCouncil: Analyze security risks
    SecurityCouncil-->>GitHub: Approve/Request changes
    deactivate SecurityCouncil
    
    GitHub->>MakinaDev: 3. Code review
    activate MakinaDev
    MakinaDev->>MakinaDev: Check implementation
    MakinaDev-->>GitHub: Approve/Request changes
    deactivate MakinaDev
    
    Note over GitHub: Required approvals received
    
    GitHub->>GitHub: 4. Merge to main branch
    deactivate GitHub
    
    Note over Operator,Caliber: Phase 2: On-Chain Scheduling
    
    RiskManager->>GitHub: 5. Retrieve merged Merkle Root
    activate RiskManager
    
    RiskManager->>Caliber: 6. Schedule root update
    activate Caliber
    Caliber->>Timelock: Register update request
    activate Timelock
    Timelock->>Timelock: Start countdown<br/>(Minimum 7 days)
    Timelock-->>Caliber: Request registered
    deactivate Timelock
    deactivate Caliber
    deactivate RiskManager
    
    Note over Operator,Caliber: Phase 3: Review & Veto Period
    
    Caliber->>Community: 7. Notify via Discord/Telegram
    activate Community
    Community->>Community: Review changes
    Community->>Community: Discuss implications
    deactivate Community
    
    SecurityCouncil->>Timelock: 8. Monitor timelock
    activate SecurityCouncil
    SecurityCouncil->>SecurityCouncil: Evaluate risk continuously
    
    opt If issues found
        SecurityCouncil->>Timelock: VETO update
        activate Timelock
        Timelock->>Timelock: Cancel scheduled update
        Timelock-->>SecurityCouncil: Update cancelled
        deactivate Timelock
        SecurityCouncil-->>Community: Notify cancellation + reason
    end
    deactivate SecurityCouncil
    
    RootGuardians->>Timelock: 9. Review Merkle Root
    activate RootGuardians
    RootGuardians->>RootGuardians: Verify instruction safety
    
    opt If root unsafe
        RootGuardians->>Timelock: VETO root update
        activate Timelock
        Timelock->>Timelock: Cancel update
        Timelock-->>RootGuardians: Root update cancelled
        deactivate Timelock
    end
    deactivate RootGuardians
    
    Note over Operator,Caliber: Phase 4: Execution
    
    Note over Timelock: Timelock period elapses<br/>No veto executed
    
    RiskManager->>Timelock: 10. Execute scheduled update
    activate Timelock
    Timelock->>Caliber: Apply new Merkle Root
    activate Caliber
    Caliber->>Caliber: Update stored root
    Caliber-->>Timelock: Root updated
    deactivate Caliber
    Timelock-->>RiskManager: Update complete
    deactivate Timelock
    
    Caliber->>Community: 11. Notify completion
    
    Note over Operator,Caliber: New instructions now active
    
    Operator->>Caliber: 12. Execute with new instructions
    activate Caliber
    Caliber->>Caliber: Verify proofs against new root
    Caliber-->>Operator: ✅ Instructions valid
    deactivate Caliber
```

## Recovery Mode Flow

```mermaid
sequenceDiagram
    actor Trigger as Trigger Event
    participant Monitor as Monitoring System
    participant SecurityCouncil as Security Council<br/>Multisig
    participant Machine
    participant HubCaliber as Hub Caliber
    participant SpokeCaliber as Spoke Caliber
    participant Operator
    participant Users
    
    Note over Trigger,Users: Detection Phase
    
    alt Suspicious Activity
        Trigger->>Monitor: ⚠️ Anomaly detected<br/>(price drop, unusual tx)
    else Operator Inactivity
        Trigger->>Monitor: ⚠️ No activity for X days
    else Hack/Exploit
        Trigger->>Monitor: 🚨 Security breach detected
    end
    
    activate Monitor
    Monitor->>SecurityCouncil: Alert with details
    deactivate Monitor
    
    Note over Trigger,Users: Emergency Decision
    
    activate SecurityCouncil
    SecurityCouncil->>SecurityCouncil: Assess threat severity
    SecurityCouncil->>SecurityCouncil: Multisig vote
    
    SecurityCouncil->>Machine: Activate Recovery Mode
    activate Machine
    Machine->>Machine: Revoke Operator permissions
    Machine->>Machine: Transfer control to Security Council
    Machine-->>SecurityCouncil: Recovery Mode ACTIVE
    deactivate Machine
    
    SecurityCouncil->>SpokeCaliber: Activate Recovery Mode
    activate SpokeCaliber
    SpokeCaliber->>SpokeCaliber: Revoke Operator permissions
    SpokeCaliber-->>SecurityCouncil: Recovery Mode ACTIVE
    deactivate SpokeCaliber
    deactivate SecurityCouncil
    
    Note over Trigger,Users: Restrictions Applied
    
    Machine->>Users: ❌ New deposits disabled
    Machine->>Machine: ❌ AUM updates paused
    Machine->>HubCaliber: ❌ Outbound bridges blocked
    
    SpokeCaliber->>SpokeCaliber: ❌ Can only DECREASE positions
    SpokeCaliber->>SpokeCaliber: ❌ Swaps ONLY to accounting token
    SpokeCaliber->>SpokeCaliber: ✅ Inbound bridges allowed (consolidate to Hub)
    
    Operator->>Machine: Attempt normal operation
    Machine-->>Operator: ❌ DENIED - No permissions
    
    Note over Trigger,Users: Recovery Actions
    
    SecurityCouncil->>SpokeCaliber: Close risky positions
    activate SpokeCaliber
    SpokeCaliber->>SpokeCaliber: Exit positions
    SpokeCaliber->>SpokeCaliber: Harvest rewards
    SpokeCaliber->>SpokeCaliber: Swap to accounting token
    deactivate SpokeCaliber
    
    SecurityCouncil->>SpokeCaliber: Bridge assets to Hub
    activate SpokeCaliber
    SpokeCaliber->>Machine: Transfer via bridge
    activate Machine
    Machine->>Machine: Consolidate assets
    deactivate Machine
    deactivate SpokeCaliber
    
    opt If Operator Slashing Needed
        SecurityCouncil->>SecurityModule: Slash Operator tokens
        activate SecurityModule
        SecurityModule->>SecurityModule: Burn operator stake
        SecurityModule->>Machine: Compensate losses
        deactivate SecurityModule
    end
    
    Note over Trigger,Users: Stabilization
    
    SecurityCouncil->>SecurityCouncil: Assess situation
    SecurityCouncil->>SecurityCouncil: Confirm safety
    
    Note over Trigger,Users: Exit Recovery Mode
    
    SecurityCouncil->>Machine: Deactivate Recovery Mode
    activate Machine
    Machine->>Machine: Restore Operator permissions
    Machine->>Machine: Re-enable deposits
    Machine->>Machine: Resume AUM updates
    Machine-->>SecurityCouncil: Normal mode restored
    deactivate Machine
    
    SecurityCouncil->>SpokeCaliber: Deactivate Recovery Mode
    activate SpokeCaliber
    SpokeCaliber->>SpokeCaliber: Restore full permissions
    SpokeCaliber-->>SecurityCouncil: Normal mode restored
    deactivate SpokeCaliber
    
    SecurityCouncil->>Users: 📢 Announce resolution
    
    Operator->>Machine: Resume normal operations
    Machine-->>Operator: ✅ Permissions restored
```

## Security Module Flow

```mermaid
sequenceDiagram
    actor User
    participant MachineToken as Machine Token<br/>(Strategy Shares)
    participant SecurityModule as Security Module<br/>(Insurance Pool)
    participant SecurityToken as Security Token<br/>(Staked Shares)
    participant FeeDistributor as Fee Distributor
    participant ReceiptNFT as Receipt NFT<br/>(Cooldown)
    
    Note over User,ReceiptNFT: Staking Flow
    
    User->>SecurityModule: 1. Stake Machine Tokens
    activate SecurityModule
    
    User->>MachineToken: 2. Transfer tokens to module
    activate MachineToken
    MachineToken-->>SecurityModule: Tokens locked
    deactivate MachineToken
    
    SecurityModule->>SecurityToken: 3. Mint Security Tokens
    activate SecurityToken
    SecurityToken-->>User: Receive security tokens
    deactivate SecurityToken
    
    SecurityModule-->>User: Staking complete
    deactivate SecurityModule
    
    Note over User,ReceiptNFT: Earning Rewards
    
    FeeDistributor->>SecurityModule: Distribute protocol fees
    activate SecurityModule
    SecurityModule->>SecurityModule: Accrue to stakers<br/>(proportional to stake)
    deactivate SecurityModule
    
    User->>SecurityModule: Claim rewards
    activate SecurityModule
    SecurityModule-->>User: Transfer accumulated fees
    deactivate SecurityModule
    
    Note over User,ReceiptNFT: Unstaking Flow (Cooldown)
    
    User->>SecurityModule: 1. Request unstake
    activate SecurityModule
    SecurityModule->>SecurityModule: Validate balance
    SecurityModule->>SecurityModule: Start cooldown<br/>(minimum 7 days)
    
    SecurityModule->>ReceiptNFT: 2. Mint Receipt NFT
    activate ReceiptNFT
    ReceiptNFT-->>User: Receive NFT<br/>(cooldown proof)
    deactivate ReceiptNFT
    
    SecurityModule-->>User: Cooldown started
    deactivate SecurityModule
    
    Note over User,ReceiptNFT: During Cooldown Period
    
    opt User changes mind
        User->>SecurityModule: Cancel cooldown
        activate SecurityModule
        SecurityModule->>ReceiptNFT: Burn Receipt NFT
        activate ReceiptNFT
        ReceiptNFT-->>SecurityModule: NFT burned
        deactivate ReceiptNFT
        SecurityModule->>SecurityModule: Restore full stake
        SecurityModule-->>User: Cooldown cancelled
        deactivate SecurityModule
    end
    
    Note over User,ReceiptNFT: After Cooldown Elapses
    
    User->>SecurityModule: Complete unstaking
    activate SecurityModule
    SecurityModule->>SecurityModule: Verify cooldown period passed
    
    SecurityModule->>ReceiptNFT: Burn Receipt NFT
    activate ReceiptNFT
    ReceiptNFT-->>SecurityModule: NFT burned
    deactivate ReceiptNFT
    
    SecurityModule->>SecurityToken: Burn Security Tokens
    activate SecurityToken
    SecurityToken-->>SecurityModule: Tokens burned
    deactivate SecurityToken
    
    SecurityModule->>MachineToken: Return original tokens
    activate MachineToken
    MachineToken-->>User: Receive Machine Tokens
    deactivate MachineToken
    
    SecurityModule-->>User: Unstaking complete
    deactivate SecurityModule
    
    Note over User,ReceiptNFT: Slashing Event (Loss Coverage)
    
    activate SecurityCouncil
    SecurityCouncil->>SecurityModule: Trigger slashing
    activate SecurityModule
    
    SecurityModule->>SecurityModule: Calculate loss amount
    SecurityModule->>SecurityModule: Apply max slashing limits
    
    SecurityModule->>SecurityToken: Burn tokens (up to limit)
    activate SecurityToken
    SecurityToken->>SecurityToken: Reduce balances proportionally
    SecurityToken-->>SecurityModule: Tokens burned
    deactivate SecurityToken
    
    SecurityModule->>Machine: Transfer value to cover loss
    activate Machine
    Machine->>Machine: Offset AUM shortfall
    deactivate Machine
    
    SecurityModule-->>SecurityCouncil: Slashing complete
    deactivate SecurityModule
    deactivate SecurityCouncil
    
    SecurityModule->>User: Notify slashing event
```

## Safe Multisig Security Architecture

```mermaid
graph TB
    subgraph "Safe Multisig Structure"
        Owners[Safe Owners<br/>N-of-M Signers]
        
        subgraph "Transaction Lifecycle"
            Propose[1. Propose Transaction]
            Simulate[2. Independent Simulation]
            Sign[3. Collect Signatures]
            Guard[4. Guard Verification]
            Hypernative[5. Hypernative Scan]
            Execute[6. Execute]
        end
        
        TimelockOwnership[Ownership Change Timelock<br/>Minimum 24 hours]
        FixedImplementation[Non-Upgradable Safe<br/>v1.4.1 Fixed]
    end
    
    subgraph "Security Layers"
        Layer1[Layer 1: Multisig Threshold<br/>Require M-of-N signatures]
        Layer2[Layer 2: Guard Contract<br/>Only authorized owners]
        Layer3[Layer 3: Hypernative<br/>Malicious payload detection]
        Layer4[Layer 4: Community Review<br/>Timelock transparency]
    end
    
    Owners --> Propose
    Propose --> Simulate
    Simulate --> Sign
    Sign --> Guard
    Guard --> Hypernative
    Hypernative --> Execute
    
    Owners --> TimelockOwnership
    Owners --> FixedImplementation
    
    Execute --> Layer1
    Execute --> Layer2
    Execute --> Layer3
    Execute --> Layer4
    
    Layer3 -.->|Block malicious| Execute
    
    style Hypernative fill:#ffcdd2
    style Guard fill:#fff9c4
    style TimelockOwnership fill:#e1bee7
    style FixedImplementation fill:#c8e6c9
```

## Governance Key Features

### Protocol-Wide Roles

| Role | Permissions | Scope |
|------|-------------|-------|
| **STRATEGY_DEPLOYER** | Deploy new strategies | Protocol-wide |
| **STRATEGY_CONFIG** | Assign/update strategy managers | Protocol-wide |
| **INFRA_CONFIG** | Configure core contracts (registries) | Protocol-wide |

### Per-Strategy Entities

| Entity | Responsibilities | Powers | Constraints |
|--------|-----------------|--------|-------------|
| **Operator** | Daily operations, position management, bridging | Execute pre-approved instructions | Max loss caps, cooldowns, instruction whitelist |
| **Risk Manager** | Update risk parameters, review instructions | Schedule parameter changes, propose root updates | All changes via timelock (min 7 days) |
| **Security Council** | Emergency oversight | Veto changes, activate Recovery Mode, slash operators | Cannot make routine changes |
| **Root Guardians** | Merkle root oversight | Veto root updates | Specific to instruction whitelisting |

---

## Timelock Mechanism

**Purpose**: Transparency and user protection

**Features:**
- **Minimum Delay**: 7 days (can be longer)
- **Transparency Window**: Users see upcoming changes
- **Veto Powers**: Security Council and Root Guardians can cancel
- **Scope**: All Risk Manager changes

**Protected Actions:**
- Merkle Root updates
- Risk parameter changes (loss caps, cooldowns)
- Base token additions/removals
- Fee parameter updates

---

## Recovery Mode Restrictions

### Hub Chain (Machine)
- ❌ New deposits disabled
- ❌ Outbound bridges (Hub → Spoke) blocked
- ✅ Inbound bridges (Spoke → Hub) allowed
- ❌ AUM updates paused
- ✅ Redemption queue still operates

### Spoke Chains (Calibers)
- ❌ Cannot increase or open new positions
- ✅ Can only decrease or close positions
- ❌ Swaps restricted to accounting token only
- ✅ Bridge assets back to Hub
- ❌ No cross-Spoke transfers

### Control Transfer
- All Operator permissions → Security Council
- Security Council acts as temporary operator
- Focus: Risk reduction and asset consolidation

---

## Safe Multisig Security Features

### 1. **Ownership Change Timelock**
- Minimum 24-hour delay
- Prevents sudden takeovers
- Community visibility

### 2. **Non-Upgradable Implementation**
- Fixed Safe version (v1.4.1)
- Cannot upgrade contract
- Eliminates upgrade attack vector

### 3. **Guard Contract**
- Restricts execution to authorized owners
- Prevents unauthorized transactions
- Additional permission layer

### 4. **Independent Simulation**
- Each owner simulates transactions separately
- Off-chain communication (Discord/Telegram)
- Multiple independent verifications

### 5. **Hypernative Guardian**
- Monitors transaction payloads
- Detects malicious code
- Blocks execution before on-chain
- Custom security policies

---

## Root Update Security

**Multi-Layer Review:**
1. **Operator Proposal** (GitHub PR)
2. **Security Council Review**
3. **Makina Developer Review**
4. **Merge to Public Repo**
5. **On-Chain Scheduling** (Risk Manager)
6. **Timelock Period** (7+ days)
7. **Community Review**
8. **Security Council Veto Window**
9. **Root Guardian Veto Window**
10. **Execution** (if no veto)

**Transparency:**
- All changes public in GitHub
- On-chain scheduling visible
- Community notifications
- Time for user exit if disagreement
