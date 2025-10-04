# Makina Protocol - User Deposit Flow

```mermaid
sequenceDiagram
    actor User
    participant Depositor
    participant Machine
    participant FeeManager
    participant MachineToken
    participant OracleRegistry
    participant Caliber
    
    Note over User,Caliber: Standard Deposit Flow
    
    User->>Depositor: 1. Deposit Accounting Tokens
    activate Depositor
    
    Depositor->>Depositor: 2. Validate deposit amount
    
    Depositor->>Machine: 3. Transfer tokens to Machine
    activate Machine
    
    Machine->>OracleRegistry: 4. Get current token prices
    activate OracleRegistry
    OracleRegistry-->>Machine: Return prices
    deactivate OracleRegistry
    
    Machine->>Caliber: 5. Query Caliber AUM (all chains)
    activate Caliber
    Caliber-->>Machine: Return local AUM
    deactivate Caliber
    
    Machine->>Machine: 6. Calculate Total AUM<br/>AUM = Σ Balance_idle + Σ Caliber_AUM + Σ Pending_bridge
    
    Machine->>Machine: 7. Calculate Current Share Price<br/>SharePrice = Total AUM / shareSupply
    
    Machine->>FeeManager: 8. Calculate and mint fees
    activate FeeManager
    FeeManager->>FeeManager: Calculate Fixed Fees<br/>(based on AUM & time)
    FeeManager->>FeeManager: Calculate Performance Fees<br/>(if above watermark)
    FeeManager->>MachineToken: Mint fee shares
    FeeManager-->>Machine: Fees processed
    deactivate FeeManager
    
    Machine->>Machine: 9. Calculate shares to mint<br/>shares = depositAmount / sharePrice
    
    Machine->>MachineToken: 10. Mint shares to user
    activate MachineToken
    MachineToken->>MachineToken: Update total supply
    MachineToken-->>User: Transfer shares
    deactivate MachineToken
    
    Machine->>Machine: 11. Update idle balance
    Machine-->>Depositor: Deposit complete
    deactivate Machine
    
    Depositor-->>User: Return confirmation
    deactivate Depositor
    
    Note over User,Caliber: Pre-Deposit Flow (Before Strategy Launch)
    
    User->>PreDepositVault: 1. Deposit yield-bearing token
    activate PreDepositVault
    
    PreDepositVault->>PreDepositVault: 2. Accrue baseline yield
    
    PreDepositVault->>MachineToken: 3. Mint future Machine shares
    activate MachineToken
    MachineToken-->>User: Receive shares
    deactivate MachineToken
    
    Note over PreDepositVault: Strategy Launch
    
    PreDepositVault->>Machine: 4. Transfer minting authority
    activate Machine
    Machine->>Machine: Take over share management
    deactivate Machine
    
    PreDepositVault-->>User: No migration needed<br/>Shares already valid
    deactivate PreDepositVault
```

## Deposit Flow Details

### Standard Deposit Process

1. **User Initiates Deposit**
   - User approves and deposits Accounting Tokens to Depositor contract
   - Only the designated Accounting Token is accepted

2. **Validation**
   - Depositor validates the deposit amount and user permissions

3. **Token Transfer**
   - Accounting Tokens transferred from user to Machine contract

4. **Price Discovery**
   - Oracle Registry queried for current token prices
   - All tokens priced against the Accounting Token

5. **AUM Aggregation**
   - Query all Calibers (Hub + Spokes) for their local AUM
   - Includes idle balances, deployed positions, and pending bridges

6. **Total AUM Calculation**
   ```
   Total AUM = Σ Balance_idle + Σ Caliber_AUM + Σ Pending_bridge
   ```

7. **Share Price Calculation**
   ```
   Share Price = Total AUM / Total Share Supply
   ```

8. **Fee Processing**
   - **Fixed Fees**: Based on AUM and time elapsed
     - Split: Security Module, Operator, Makina DAO
   - **Performance Fees**: Only if above high watermark
     - Split: Operator, Makina DAO
   - Fee shares minted to respective recipients

9. **Share Minting**
   ```
   Shares to Mint = Deposit Amount / Current Share Price
   ```

10. **Token Distribution**
    - Machine Tokens (ERC-20 shares) minted to user's address
    - Total supply updated

11. **Balance Update**
    - Machine's idle balance increased by deposit amount

### Pre-Deposit Flow

**Purpose**: Build baseline liquidity before strategy launch

1. **Pre-Deposit Phase**
   - Users deposit yield-bearing tokens (e.g., stETH, aUSDC)
   - PreDepositVault holds these tokens
   - Vault's share price increases with accrued yield

2. **Share Minting**
   - PreDepositVault mints future Machine Shares immediately
   - Users earn baseline yield during pre-deposit period

3. **Strategy Launch**
   - Machine deploys and takes over minting authority
   - **No migration needed** - shares already valid Machine Tokens
   - Seamless transition from pre-deposit to active strategy

4. **Benefits**
   - Users earn yield during pre-launch
   - No gas costs for migration
   - Immediate liquidity at launch
