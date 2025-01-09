# Swap Commission fee UML Schema

![Untitled diagram-2025-01-09-161516](https://github.com/user-attachments/assets/d159a6b9-cd3c-4b6c-806f-e17c42cded74)


## Key Components:

1. **Pool Type & Spread Factor Resolution**
   - Supports multiple pool types (Stableswap, Concentrated Liquidity)
   - Each pool type implements GetSpreadFactor differently
   - Validates spread factor against pool's minimum requirements

2. **CFMM Operations**
   - Converts pool to CFMM interface
   - Calculates swap amounts using constant product formula
   - Applies spread factor for fee calculation

3. **Validation & Safety**
   - Ensures positive output amounts
   - Validates minimum output requirements
   - Prevents same denomination swaps
   - Handles panic recovery

4. **Settlement Process**
   - Updates pool state
   - Executes token transfers
   - Emits events
   - Updates liquidity metrics

## Code Implementation Details

### 1. Spread Factor Validation
```go
// From swap.go
poolSpreadFactor := pool.GetSpreadFactor(ctx)
if spreadFactor.LT(poolSpreadFactor.QuoInt64(2)) {
    return osmomath.Int{}, fmt.Errorf("given spread factor (%s) must be greater than or equal to half of the pool's spread factor (%s)", 
        spreadFactor, poolSpreadFactor)
}
```

### 2. CFMM Pool Operations
```go
// Convert to CFMM Pool
cfmmPool, err := asCFMMPool(pool)
if err != nil {
    return osmomath.Int{}, err
}

// Execute swap calculation
tokenOutCoin, err := cfmmPool.SwapOutAmtGivenIn(ctx, tokensIn, tokenOutDenom, spreadFactor)
if err != nil {
    return osmomath.Int{}, err
}
```

### 3. Amount Validation
```go
// Validate output amount
if !tokenOutAmount.IsPositive() {
    return osmomath.Int{}, errorsmod.Wrapf(types.ErrInvalidMathApprox, "token amount must be positive")
}

if tokenOutAmount.LT(tokenOutMinAmount) {
    return osmomath.Int{}, errorsmod.Wrapf(types.ErrLimitMinAmount, "%s token is lesser than min amount", tokenOutDenom)
}
```

### 4. Settlement Process
```go
// Handle token transfers and pool updates
err = k.bankKeeper.SendCoins(ctx, sender, pool.GetAddress(), sdk.Coins{tokenIn})
if err != nil {
    return err
}

err = k.bankKeeper.SendCoins(ctx, pool.GetAddress(), sender, sdk.Coins{tokenOut})
if err != nil {
    return err
}

// Emit events and update metrics
events.EmitSwapEvent(ctx, sender, pool.GetId(), tokensIn, tokensOut)
k.hooks.AfterCFMMSwap(ctx, sender, pool.GetId(), tokensIn, tokensOut)
k.RecordTotalLiquidityIncrease(ctx, tokensIn)
k.RecordTotalLiquidityDecrease(ctx, tokensOut)
```

### Key Calculations:
1. **Spread Factor (Fee) Calculation**:
   - Minimum allowed spread factor = Pool's spread factor / 2
   - Effective Output = Calculated Output * (1 - spreadFactor)
   - Fee Amount = Calculated Output * spreadFactor

2. **CFMM (Constant Function Market Maker) Properties**:
   - Maintains constant product formula: x * y = k
   - Pool balances are updated after each swap
   - Spread factor is applied to the calculated output amount

3. **Safety Checks**:
   - Validates input and output denominations are different
   - Ensures output amount is positive
   - Verifies minimum output amount requirements
   - Checks pool liquidity sufficiency

### 5. Spread Factor Configuration

#### Setting Spread Factor
The spread factor (commission fee) is configured in multiple places:

1. **Pool Creation**:
```go
// When creating a pool, the initial spread factor is set
func NewBalancerPool(
    poolId uint64,
    poolParams balancer.PoolParams, // Contains SpreadFactor
    poolAssets []balancer.PoolAsset,
    blockTime time.Time,
) (Pool, error) {
    // spread_factor validation: 0 <= spread_factor <= 1
    if poolParams.SpreadFactor.IsNegative() || poolParams.SpreadFactor.GTE(one) {
        return Pool{}, fmt.Errorf("spread_factor must be between 0 and 1")
    }
}
```

2. **Swap Execution**:
```go
// During swap execution, spread factor is passed as parameter
func (k Keeper) SwapExactAmountIn(
    ctx sdk.Context,
    sender sdk.AccAddress,
    pool poolmanagertypes.PoolI,
    tokenIn sdk.Coin,
    tokenOutDenom string,
    tokenOutMinAmount osmomath.Int,
    spreadFactor osmomath.Dec,  // Passed during swap execution
)
```

#### Spread Factor Validation Rules:
1. Must be between 0 and 1 (0% to 100%)
2. During swaps, provided spread factor must be ≥ pool's spread factor / 2
3. Typical values range from 0.001 (0.1%) to 0.01 (1%)

#### Example Configuration:
```go
// Common spread factor values
standardSpreadFactor := osmomath.NewDecWithPrec(3, 3)  // 0.3%
lowSpreadFactor := osmomath.NewDecWithPrec(1, 3)      // 0.1%
highSpreadFactor := osmomath.NewDecWithPrec(10, 3)    // 1.0%
```

The spread factor directly impacts:
- Trading costs for users
- Revenue generation for liquidity providers
- Pool's competitiveness in the market

### 6. Spread Factor Context and Management

#### Spread Factor Storage and Access
The spread factor is managed through the Context system and is accessed via the `GetSpreadFactor` interface method:

```go
// Interface method in poolmanager/types/pool.go
GetSpreadFactor(ctx sdk.Context) osmomath.Dec
```

#### Spread Factor Lifecycle

1. **Initial Setting**:
   - Set during pool creation in genesis or pool creation transactions
   - Stored in pool-specific state
   - Must be between 0 and 1 (0% to 100%)

2. **Access Pattern**:
   - Retrieved through pool interface using Context
   - Context provides access to:
     - State store (ms MultiStore)
     - Transaction info (txBytes)
     - Gas management (gasMeter)
     - Event emission (eventManager)

3. **Validation Flow**:
```go
// During swap operations:
poolSpreadFactor := pool.GetSpreadFactor(ctx)
if spreadFactor.LT(poolSpreadFactor.QuoInt64(2)) {
    return osmomath.Int{}, fmt.Errorf(
        "given spread factor (%s) must be greater than or equal to half of the pool's spread factor (%s)", 
        spreadFactor, 
        poolSpreadFactor,
    )
}
```

4. **Key Implementation Points**:
   - Spread factor is immutable after pool creation
   - Validated on every swap operation
   - Used in CFMM calculations for:
     - SwapExactAmountIn
     - SwapExactAmountOut
     - CalcOutAmtGivenIn
     - CalcInAmtGivenOut

5. **Context Usage**:
   - The Context struct provides necessary infrastructure for:
     - State access and modification
     - Event emission for spread factor usage
     - Gas accounting for operations
     - Transaction validation

6. **Typical Values and Constraints**:
   - Default pool spread factor: 0.003 (0.3%)
   - Minimum swap spread factor: poolSpreadFactor/2
   - Maximum spread factor: 1.0 (100%)
   - Common range: 0.1% to 1%

### 7. Spread Factor Implementation Details

#### Pool-Specific Implementations

1. **Stableswap Pools**
```go
// In x/gamm/pool-models/stableswap/pool.go
func (p Pool) GetSpreadFactor(ctx sdk.Context) osmomath.Dec {
    return p.PoolParams.SwapFee
}

// Definition in stableswap_pool.pb.go
type PoolParams struct {
    SwapFee cosmossdk_io_math.LegacyDec `json:"swap_fee" yaml:"swap_fee"`
}
```

2. **Concentrated Liquidity Pools**
```go
// In x/concentrated-liquidity/model/pool.go
func (p Pool) GetSpreadFactor(ctx sdk.Context) osmomath.Dec {
    return p.SpreadFactor
}

// Definition in pool.pb.go
type Pool struct {
    SpreadFactor cosmossdk_io_math.LegacyDec `json:"spread_factor" yaml:"spread_factor"`
}
```

#### Spread Factor Setting Points

1. **Pool Creation**
   - For Stableswap: Set via `SwapFee` in pool parameters during pool creation
   - For Concentrated Liquidity: Set via `SpreadFactor` in pool initialization

2. **Genesis State**
   - Defined in genesis state for existing pools
   - Cannot be modified after pool creation

3. **Value Constraints**
   - Must be non-negative: `spreadFactor >= 0`
   - Must be less than 1: `spreadFactor < 1`
   - Typical values: 0.001 (0.1%) to 0.01 (1%)

4. **Usage in Swap Operations**
   ```go
   // Validation during swap
   poolSpreadFactor := pool.GetSpreadFactor(ctx)
   if spreadFactor.LT(poolSpreadFactor.QuoInt64(2)) {
       return osmomath.Int{}, fmt.Errorf(
           "given spread factor (%s) must be greater than or equal to half of the pool's spread factor (%s)", 
           spreadFactor, 
           poolSpreadFactor,
       )
   }
   ```

#### Implementation Notes

1. **Naming Convention**
   - Stableswap pools use `SwapFee`
   - Concentrated Liquidity pools use `SpreadFactor`
   - Both represent the same concept: trading fee percentage

2. **Storage**
   - Stored as `LegacyDec` type from Cosmos SDK
   - Allows for precise decimal calculations
   - Persisted in pool state

3. **Access Pattern**
   - Read-only after pool creation
   - Accessed through `GetSpreadFactor` interface method
   - Used in all swap calculations

4. **Fee Calculation Example**
   ```go
   effectivePrice = calculatedAmount * (1 - spreadFactor)
   feeAmount = calculatedAmount * spreadFactor
   ```

## Code Implementation Flow

![Untitled diagram-2025-01-09-160916](https://github.com/user-attachments/assets/1f06ea82-3cea-49bb-a73c-8a36bd5d083c)

The diagram above illustrates the detailed code flow from the implementation, showing:
1. Method calls and their parameters
2. Error handling and validation steps
3. Pool operations and state updates
4. Settlement process and event emission
5. Panic recovery and error wrapping