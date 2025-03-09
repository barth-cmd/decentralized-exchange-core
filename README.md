# Decentralized Exchange Core: Automated Market Maker (AMM) Smart Contract

**Version 1.0** | **Network**: Stacks Blockchain

### Overview

This repository contains a production-grade Automated Market Maker (AMM) implementation for decentralized exchanges, built on the Clarity smart contract language. The protocol enables permissionless token swaps, liquidity provisioning, and pool management using the **constant product formula** (`x * y = k`), with enhanced features for security, fee management, and administrative oversight.

### Key Features

1. **Pool Management**

   - Create liquidity pools for any token pair
   - Track reserves and LP shares with atomic precision
   - Emergency pool freezing/resuming via admin controls

2. **Liquidity Engine**

   - Mint/burn LP tokens proportional to deposited assets
   - Optimized share calculation during initial liquidity provision
   - Slippage protection for add/remove liquidity operations

3. **Swap Mechanism**

   - Fee-aware output calculation (0.3% default protocol fee)
   - Configurable minimum output thresholds
   - Bidirectional trading (X→Y or Y→X)

4. **Protocol Governance**
   - Dynamic fee adjustment by contract owner
   - Real-time reserve tracking for price oracles
   - Restricted admin functions for risk mitigation

### Core Components

#### 1. **Fungible Token Interface**

Implements `ft-trait` for ERC-20-like token interactions:

```clarity
(define-trait ft-trait
  (
    (transfer (uint principal principal) (response bool uint))
    (get-balance (principal) (response uint uint))
    ;; Additional standard functions...
  )
)
```

#### 2. **Pool Architecture**

```clarity
(define-map pools
  uint
  {
    token-x: principal,    // Address of first token
    token-y: principal,    // Address of second token
    reserve-x: uint,       // Token X liquidity
    reserve-y: uint,       // Token Y liquidity
    total-shares: uint,    // Outstanding LP tokens
    active: bool           // Operational status
  }
)
```

#### 3. **Swap Mathematics**

**Output Calculation Formula**:

```
amount_out = (input_amount * (1 - fee) * output_reserve) /
             (input_reserve + input_amount * (1 - fee))
```

Implemented via precision-aware integer arithmetic:

```clarity
(define-private (calculate-output-amount
  (input-amount uint)
  (input-reserve uint)
  (output-reserve uint)
)
```

### Critical Functions

#### Pool Creation (`create-pool`)

- **Requirements**: Contract owner authorization
- **Checks**: Token addresses must be distinct
- **Output**: Returns new `pool-id`

#### Add Liquidity (`add-liquidity`)

1. Transfers tokens from user to contract
2. Mints LP shares using geometric mean optimization:
   ```
   shares = min(
     (amount_x * total_shares) / reserve_x,
     (amount_y * total_shares) / reserve_y
   )
   ```
3. Enforces minimum share requirement

#### Token Swap (`swap-exact-tokens`)

- Validates pool activation status
- Executes cross-contract token transfers
- Updates reserves atomically
- Enforces slippage tolerance via `min-amount-out`

### Security Model

#### Error Handling

| Code                     | Description                    |
| ------------------------ | ------------------------------ |
| `ERR-NOT-AUTHORIZED`     | Unauthorized admin action      |
| `ERR-SLIPPAGE_TOLERANCE` | Price impact exceeds threshold |
| `ERR-ZERO_LIQUIDITY`     | Invalid pool initialization    |

#### Administrative Safeguards

- **Circuit Breakers**: `pause-pool`/`resume-pool` functions
- **Fee Governance**: `set-protocol-fee` with basis point validation
- **Ownership**: Hardcoded `CONTRACT-OWNER` for privileged operations

### Integration Guide

#### Prerequisites

1. **Token Contracts**: Must implement `ft-trait`
2. **Wallet Balance**: Users must approve token transfers

#### Workflow

1. **Pool Creation**

   ```clarity
   (create-pool token-x-contract token-y-contract)
   ```

2. **Initial Liquidity**

   ```clarity
   (add-liquidity
     pool-id
     token-x
     token-y
     amount-x
     amount-y
     min-shares
   )
   ```

3. **Execute Swap**
   ```clarity
   (swap-exact-tokens
     pool-id
     token-in
     token-out
     amount-in
     min-out
     swap-direction
   )
   ```
