# Venus Protocol - Comprehensive Study Guide

> A complete technical breakdown for master's exam preparation covering the mathematics, core functions, and architecture of Venus Protocol.

> **✅ Verified against official Venus Protocol documentation** ([docs-v4.venus.io](https://docs-v4.venus.io/)) on January 2026. Formulas cross-referenced with:
> - [Interest Rate Model](https://docs-v4.venus.io/risk/interest-rate-model)
> - [Protocol Math](https://docs-v4.venus.io/guides/protocol-math)
> - [Liquidation Guide](https://docs-v4.venus.io/guides/liquidation)
> - [Two Kinks Interest Rate Curve](https://docs-v4.venus.io/technical-reference/reference-technical-articles/two-kinks-interest-rate-curve)

---

## Table of Contents
1. [Protocol Overview](#1-protocol-overview)
2. [Fixed-Point Arithmetic Deep Dive](#2-fixed-point-arithmetic-deep-dive)
3. [Interest Accrual: Discrete vs Continuous Compounding](#3-interest-accrual-discrete-vs-continuous-compounding)
4. [vToken System - Core Functions](#4-vtoken-system---core-functions)
5. [Interest Rate Models](#5-interest-rate-models)
6. [Account Liquidity & Risk Management](#6-account-liquidity--risk-management)
7. [Liquidation Mechanics](#7-liquidation-mechanics)
8. [Comparison with ERC-4626 Vault Standard](#8-comparison-with-erc-4626-vault-standard)
9. [Risk Parameter Framework](#9-risk-parameter-framework)
10. [Mathematical Derivations](#10-mathematical-derivations)
11. [Key Formulas Cheat Sheet](#11-key-formulas-cheat-sheet)
12. [Important State Variables](#12-important-state-variables)
13. [Flow Diagrams](#13-flow-diagrams)
14. [Academic Papers & Resources](#14-academic-papers--resources)

---

## 1. Protocol Overview

Venus Protocol is a **decentralized money market** (also called a Protocol for Loanable Funds or PLF) on BNB Chain. It enables:
- **Supplying assets** to earn interest (receive vTokens)
- **Borrowing assets** against collateral
- **Minting VAI** synthetic stablecoin
- **Liquidating** undercollateralized positions

### 1.1 Protocol Classification

Venus belongs to the family of **overcollateralized lending protocols** pioneered by Compound. Key characteristics:
- **Algorithmic interest rates** based on utilization
- **Pooled liquidity** (not peer-to-peer matching)
- **Overcollateralization** requirement (borrow < collateral value)
- **Permissionless liquidation** by any third party

### 1.2 Architecture Components

```
┌─────────────────────────────────────────────────────────────┐
│                    UNITROLLER (Proxy)                        │
│                 Manages Diamond upgrades                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│              DIAMOND COMPTROLLER (Risk Engine)               │
├─────────────────────────────────────────────────────────────┤
│  Facets:                                                     │
│  • MarketFacet    → Market entry/exit, delegation           │
│  • PolicyFacet    → Borrow/supply/liquidate validation      │
│  • RewardFacet    → XVS reward distribution                 │
│  • SetterFacet    → Parameter configuration                 │
└──────────┬──────────────┬──────────────┬────────────────────┘
           │              │              │
    ┌──────▼──────┐ ┌─────▼─────┐ ┌─────▼─────┐
    │   vTokens   │ │  Oracles  │ │    VAI    │
    │ (Markets)   │ │ (Prices)  │ │ (Stable)  │
    └─────────────┘ └───────────┘ └───────────┘
```

---

## 2. Fixed-Point Arithmetic Deep Dive

### 2.1 Why Fixed-Point in Solidity?

Solidity does not support floating-point arithmetic because:
1. **Determinism**: Floating-point operations can vary across machines, causing consensus failures
2. **Gas efficiency**: Integer operations are cheaper than floating-point
3. **Precision control**: Fixed-point gives predictable precision

**Reference**: [Fixed Point Arithmetic in Solidity - RareSkills](https://rareskills.io/post/solidity-fixed-point)

### 2.2 The WAD Convention (1e18)

**WAD** = "Wei as Decimal" = 10^18 scaling factor

```
Real Value    Mantissa Representation
─────────────────────────────────────
1.0           1e18 = 1000000000000000000
0.5           5e17 = 500000000000000000
0.75          7.5e17 = 750000000000000000
1.10          1.1e18 = 1100000000000000000
0.05          5e16 = 50000000000000000
```

### 2.3 Venus Exponential Math Library

From `ExponentialNoError.sol`:

```solidity
contract ExponentialNoError {
    uint internal constant expScale = 1e18;      // WAD
    uint internal constant doubleScale = 1e36;   // For intermediate calculations
    uint internal constant halfExpScale = 5e17;  // For rounding
    uint internal constant mantissaOne = 1e18;   // Represents 1.0

    struct Exp {
        uint mantissa;  // Value × 1e18
    }

    struct Double {
        uint mantissa;  // Value × 1e36 (for higher precision intermediates)
    }
}
```

### 2.4 Fixed-Point Operations

**Multiplication** (two mantissa values):
```
result = (a × b) / 1e18

Example: 0.5 × 0.8 = 0.4
Code:    (5e17 × 8e17) / 1e18 = 4e17 ✓
```

```solidity
function mul_(Exp memory a, Exp memory b) internal pure returns (Exp memory) {
    return Exp({ mantissa: mul_(a.mantissa, b.mantissa) / expScale });
}
```

**Division** (two mantissa values):
```
result = (a × 1e18) / b

Example: 0.6 / 0.8 = 0.75
Code:    (6e17 × 1e18) / 8e17 = 7.5e17 ✓
```

```solidity
function div_(Exp memory a, Exp memory b) internal pure returns (Exp memory) {
    return Exp({ mantissa: div_(mul_(a.mantissa, expScale), b.mantissa) });
}
```

**Scalar Multiplication** (mantissa × integer):
```
result = (mantissa × scalar) / 1e18

Example: 0.02 (exchange rate) × 1000 (vTokens) = 20 underlying
Code:    (2e16 × 1000) / 1e18 = 20 ✓
```

### 2.5 Precision Loss and Rounding

**Critical insight**: Division always rounds DOWN (truncates) in Solidity.

```solidity
function truncate(Exp memory exp) internal pure returns (uint) {
    return exp.mantissa / expScale;  // Loses fractional part
}
```

This has implications:
- When **minting shares**: User gets slightly fewer shares (favors protocol)
- When **redeeming**: User gets slightly less underlying (favors protocol)
- Consistent with ERC-4626 rounding rules: "round in favor of the vault"

---

## 3. Interest Accrual: Discrete vs Continuous Compounding

### 3.1 Mathematical Background

**Continuous Compounding** (theoretical):
```
A = P × e^(r×t)

Where:
  A = Final amount
  P = Principal
  r = Annual rate
  t = Time in years
  e = Euler's number (≈2.71828)
```

**Discrete Compounding** (what DeFi uses):
```
A = P × (1 + r/n)^(n×t)

Where:
  n = Number of compounding periods per year
```

**As n → ∞**, discrete approaches continuous:
```
lim(n→∞) (1 + r/n)^n = e^r
```

### 3.2 Why DeFi Uses Discrete (Per-Block) Compounding

1. **Gas costs**: Exponential calculations (e^x) are expensive
2. **Determinism**: Block-by-block is predictable
3. **Practicality**: Interest only matters when users interact

**Venus uses simple interest approximation per block:**
```
interestFactor = borrowRate × blockDelta
newBalance ≈ oldBalance × (1 + interestFactor)
```

This is valid because `interestFactor` is tiny (rate per block × few blocks).

### 3.3 The BorrowIndex Mechanism

Instead of updating every borrower's balance each block, Venus uses a **global index**:

```solidity
struct BorrowSnapshot {
    uint principal;      // Balance when last updated
    uint interestIndex;  // Global borrowIndex at that time
}
```

**Index Update** (each block with activity):
```
borrowIndex_new = borrowIndex_old × (1 + borrowRate × blockDelta)
```

**Individual Balance Calculation** (on-demand):
```
currentBalance = principal × (borrowIndex_current / borrowIndex_snapshot)
```

**Mathematical Proof of Equivalence:**

Let user borrow at time t₀ when index = I₀
After n periods with rates r₁, r₂, ..., rₙ:

```
I_n = I₀ × (1+r₁) × (1+r₂) × ... × (1+rₙ)

Balance_n = Principal × (I_n / I₀)
          = Principal × (1+r₁) × (1+r₂) × ... × (1+rₙ)
```

This equals compound interest! The index captures cumulative growth.

### 3.4 Interest Accrual Code

From `VToken.sol:562-681`:

```solidity
function accrueInterest() public returns (uint) {
    uint currentBlockNumber = block.number;
    uint accrualBlockNumberPrior = accrualBlockNumber;

    // Short-circuit if already accrued this block
    if (accrualBlockNumberPrior == currentBlockNumber) {
        return uint(Error.NO_ERROR);
    }

    // Read prior values
    uint cashPrior = _getCashPriorWithFlashLoan();
    uint borrowsPrior = totalBorrows;
    uint reservesPrior = totalReserves;
    uint borrowIndexPrior = borrowIndex;

    // Get current borrow rate from interest model
    uint borrowRateMantissa = interestRateModel.getBorrowRate(
        cashPrior, borrowsPrior, reservesPrior
    );

    // Calculate blocks elapsed
    uint blockDelta = currentBlockNumber - accrualBlockNumberPrior;

    /*
     * Calculate new values:
     *   simpleInterestFactor = borrowRate × blockDelta
     *   interestAccumulated = simpleInterestFactor × totalBorrows
     *   totalBorrowsNew = totalBorrows + interestAccumulated
     *   totalReservesNew = totalReserves + interestAccumulated × reserveFactor
     *   borrowIndexNew = borrowIndex × (1 + simpleInterestFactor)
     */

    Exp memory simpleInterestFactor = mul_(
        Exp({ mantissa: borrowRateMantissa }),
        blockDelta
    );

    uint interestAccumulated = mul_ScalarTruncate(
        simpleInterestFactor,
        borrowsPrior
    );

    uint totalBorrowsNew = borrowsPrior + interestAccumulated;

    uint totalReservesNew = mul_ScalarTruncateAddUInt(
        Exp({ mantissa: reserveFactorMantissa }),
        interestAccumulated,
        reservesPrior
    );

    uint borrowIndexNew = mul_ScalarTruncateAddUInt(
        simpleInterestFactor,
        borrowIndexPrior,
        borrowIndexPrior
    );

    // Write to storage
    accrualBlockNumber = currentBlockNumber;
    borrowIndex = borrowIndexNew;
    totalBorrows = totalBorrowsNew;
    totalReserves = totalReservesNew;
}
```

---

## 4. vToken System - Core Functions

### 4.1 Exchange Rate

**The fundamental equation** connecting vTokens to underlying assets:

```
                 totalCash + totalBorrows - totalReserves
exchangeRate = ─────────────────────────────────────────────
                            totalSupply
```

**Key insight**: `totalBorrows` is included because those assets are still "owned" by the pool—they're just lent out.

**When totalSupply = 0:**
```
exchangeRate = initialExchangeRateMantissa  (typically 0.02e18 or 1e18)
```

#### 4.1.1 Decimal Conversion (from Official Docs)

> **Important**: All vTokens have **8 decimal places**, regardless of the underlying token's decimals.

**Converting exchange rate to actual value:**
```
mantissa = 18 + underlyingDecimals - vTokenDecimals
         = 18 + underlyingDecimals - 8

oneVTokenInUnderlying = exchangeRateCurrent / 10^mantissa
```

**Calculating underlying from vTokens:**
```
underlyingTokens = vTokenAmount × oneVTokenInUnderlying
```

**Example with USDC (6 decimals):**
```
mantissa = 18 + 6 - 8 = 16
exchangeRateMantissa = 0.02 × 1e18 = 2e16

oneVTokenInUnderlying = 2e16 / 1e16 = 2

So 1 vUSDC (with 8 decimals) = 2 USDC (with 6 decimals)?
→ Actually: 1e8 vUSDC smallest units = 2e6 USDC smallest units
→ Ratio per base unit = 2e6 / 1e8 = 0.02 USDC per vUSDC ✓
```

**Example with ETH/BNB (18 decimals):**
```
mantissa = 18 + 18 - 8 = 28
exchangeRateMantissa = 0.02 × 1e18 = 2e16

oneVTokenInUnderlying = 2e16 / 1e28 = 2e-12

So 1 vETH base unit = 2e-12 ETH base units?
→ Since vETH has 8 decimals and ETH has 18 decimals:
→ 1 vETH (1e8 base) = 0.02 ETH (0.02e18 base) = 2e16 wei ✓
```

> **Note**: For vBNB, use `underlyingDecimals = 18` since BNB doesn't have a separate token contract.

**Why exchange rate increases:**
```
Time 0: cash=1000, borrows=0, reserves=0, supply=50000
        rate = 1000/50000 = 0.02

Time 1: cash=900, borrows=110 (100 borrowed + 10 interest), reserves=2
        rate = (900 + 110 - 2)/50000 = 1008/50000 = 0.02016

Suppliers earned: (0.02016 - 0.02) / 0.02 = 0.8% return
```

### 4.2 Mint (Supply)

User deposits underlying → receives vTokens

**Formula:**
```
                 underlyingAmount
vTokensMinted = ──────────────────
                   exchangeRate
```

**Example:**
```
User deposits 100 USDC
Exchange rate = 0.02 (1 vUSDC = 0.02 USDC)
vTokens minted = 100 / 0.02 = 5000 vUSDC
```

**Code** (`VToken.sol:888-954`):
```solidity
function mintFresh(address minter, uint mintAmount) internal returns (uint, uint) {
    // 1. Comptroller permission check
    uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);

    // 2. Get exchange rate (interest already accrued)
    (vars.mathErr, vars.exchangeRateMantissa) = exchangeRateStoredInternal();

    // 3. Transfer underlying from user
    vars.actualMintAmount = doTransferIn(minter, mintAmount);

    // 4. Calculate vTokens: mintTokens = amount / exchangeRate
    (vars.mathErr, vars.mintTokens) = divScalarByExpTruncate(
        vars.actualMintAmount,
        Exp({ mantissa: vars.exchangeRateMantissa })
    );

    // 5. Update state
    totalSupply = totalSupply + vars.mintTokens;
    accountTokens[minter] = accountTokens[minter] + vars.mintTokens;
}
```

### 4.3 Redeem (Withdraw)

**Two entry points:**

1. **Redeem by vToken amount:**
   ```
   underlyingReceived = vTokens × exchangeRate
   ```

2. **Redeem by underlying amount:**
   ```
   vTokensBurned = underlyingAmount / exchangeRate
   ```

**Code** (`VToken.sol:1111-1230`):
```solidity
function redeemFresh(
    address redeemer,
    address payable receiver,
    uint redeemTokensIn,    // vTokens to redeem (or 0)
    uint redeemAmountIn     // Underlying to receive (or 0)
) internal returns (uint) {
    // Only one can be non-zero
    require(redeemTokensIn == 0 || redeemAmountIn == 0);

    (vars.mathErr, vars.exchangeRateMantissa) = exchangeRateStoredInternal();

    if (redeemTokensIn > 0) {
        // User specifies vTokens
        vars.redeemTokens = redeemTokensIn;
        vars.redeemAmount = mul_ScalarTruncate(
            Exp({ mantissa: vars.exchangeRateMantissa }),
            redeemTokensIn
        );
    } else {
        // User specifies underlying amount
        vars.redeemTokens = divScalarByExpTruncate(
            redeemAmountIn,
            Exp({ mantissa: vars.exchangeRateMantissa })
        );
        vars.redeemAmount = redeemAmountIn;
    }

    // Comptroller checks if redeem would cause shortfall
    uint allowed = comptroller.redeemAllowed(address(this), redeemer, vars.redeemTokens);

    // Update state
    totalSupply = totalSupply - vars.redeemTokens;
    accountTokens[redeemer] = accountTokens[redeemer] - vars.redeemTokens;

    // Transfer underlying
    doTransferOut(receiver, vars.redeemAmount);
}
```

### 4.4 Borrow

**User borrows underlying against their collateral**

```solidity
function borrowFresh(
    address borrower,
    address payable receiver,
    uint borrowAmount,
    bool shouldTransfer
) internal returns (uint) {
    // 1. Comptroller checks liquidity
    uint allowed = comptroller.borrowAllowed(address(this), borrower, borrowAmount);

    // 2. Get current borrow balance (includes accrued interest)
    (vars.mathErr, vars.accountBorrows) = borrowBalanceStoredInternal(borrower);

    // 3. Calculate new balances
    vars.accountBorrowsNew = vars.accountBorrows + borrowAmount;
    vars.totalBorrowsNew = totalBorrows + borrowAmount;

    // 4. Update state with NEW index snapshot
    accountBorrows[borrower].principal = vars.accountBorrowsNew;
    accountBorrows[borrower].interestIndex = borrowIndex;
    totalBorrows = vars.totalBorrowsNew;

    // 5. Transfer if requested
    if (shouldTransfer) {
        doTransferOut(receiver, borrowAmount);
    }
}
```

### 4.5 Borrow Balance Calculation

**The index-based compound interest formula:**

```
                    principal × currentBorrowIndex
currentBalance = ─────────────────────────────────────
                       snapshotInterestIndex
```

```solidity
function borrowBalanceStoredInternal(address account) internal view returns (MathError, uint) {
    BorrowSnapshot storage borrowSnapshot = accountBorrows[account];

    // No debt = 0 balance
    if (borrowSnapshot.principal == 0) {
        return (MathError.NO_ERROR, 0);
    }

    // Calculate: principal × borrowIndex / interestIndex
    uint principalTimesIndex = borrowSnapshot.principal * borrowIndex;
    uint result = principalTimesIndex / borrowSnapshot.interestIndex;

    return (MathError.NO_ERROR, result);
}
```

**Numerical Example:**
```
Time 0: User borrows 1000 USDC
        borrowIndex = 1.0e18
        Snapshot: {principal: 1000e18, interestIndex: 1.0e18}

Time 1: 5% interest accrued globally
        borrowIndex = 1.05e18

User's balance = 1000e18 × 1.05e18 / 1.0e18 = 1050e18 = 1050 USDC
```

### 4.6 Repay Borrow

```solidity
function repayBorrowFresh(address payer, address borrower, uint repayAmount) internal {
    // 1. Get current balance with interest
    vars.accountBorrows = borrowBalanceStoredInternal(borrower);

    // 2. Handle "repay max" case
    if (repayAmount == type(uint256).max) {
        vars.repayAmount = vars.accountBorrows;
    } else {
        vars.repayAmount = repayAmount;
    }

    // 3. Transfer from payer
    vars.actualRepayAmount = doTransferIn(payer, vars.repayAmount);

    // 4. Calculate new balances
    vars.accountBorrowsNew = vars.accountBorrows - vars.actualRepayAmount;
    vars.totalBorrowsNew = totalBorrows - vars.actualRepayAmount;

    // 5. Update state with fresh index
    accountBorrows[borrower].principal = vars.accountBorrowsNew;
    accountBorrows[borrower].interestIndex = borrowIndex;
    totalBorrows = vars.totalBorrowsNew;
}
```

---

## 5. Interest Rate Models

### 5.1 Utilization Rate

**The demand/supply indicator:**

Venus uses **two different utilization calculations** (important distinction from official docs):

**1. Utilization Rate `u` (for borrow rate calculation):**
```
              borrows + badDebt
u = ─────────────────────────────────────────
     cash + borrows + badDebt - reserves
```

**2. Supply Utilization `us` (for supply rate calculation):**
```
              borrows
us = ─────────────────────────────────────────
      cash + borrows - reserves + badDebt
```

> **Note**: `badDebt` represents unrecoverable debt from accounts with insufficient collateral. It's included in utilization calculations to accurately reflect protocol health.

**Range**: [0, 1] where:
- 0 = No borrowing (all funds idle)
- 1 = 100% utilization (no available liquidity)

```solidity
function utilizationRate(uint cash, uint borrows, uint reserves, uint badDebt) public pure returns (uint) {
    if (borrows + badDebt == 0) {
        return 0;
    }
    return (borrows + badDebt) * 1e18 / (cash + borrows + badDebt - reserves);
}
```

### 5.2 JumpRateModel (Kinked Model)

**Piecewise linear function with two slopes:**

```
         ┌ baseRate + U × multiplier                       if U ≤ kink
R_borrow = │
         └ baseRate + kink × multiplier + (U - kink) × jumpMultiplier   if U > kink
```

**Visual Representation:**
```
Borrow
Rate │
     │                              ╱
     │                            ╱  (jumpMultiplier slope)
     │                          ╱
     │                        ╱
     │         ─────────────•  ← kink (typically 80%)
     │        ╱  (multiplier slope)
     │      ╱
     │    ╱
     │  ╱
     │╱ baseRate
     └──────────────────────────────── Utilization
     0%              kink           100%
```

**Implementation** (`JumpRateModel.sol`):
```solidity
function getBorrowRate(uint cash, uint borrows, uint reserves) public view returns (uint) {
    uint util = utilizationRate(cash, borrows, reserves);

    if (util <= kink) {
        // Linear: rate = base + util × multiplier
        return util * multiplierPerBlock / 1e18 + baseRatePerBlock;
    } else {
        // Above kink: add steep jump component
        uint normalRate = kink * multiplierPerBlock / 1e18 + baseRatePerBlock;
        uint excessUtil = util - kink;
        return excessUtil * jumpMultiplierPerBlock / 1e18 + normalRate;
    }
}
```

**Parameter Conversion (Annual → Per Block):**
```solidity
blocksPerYear = 10512000;  // ~3 seconds per block on BSC
baseRatePerBlock = baseRatePerYear / blocksPerYear;
multiplierPerBlock = multiplierPerYear / blocksPerYear;
jumpMultiplierPerBlock = jumpMultiplierPerYear / blocksPerYear;
```

### 5.3 TwoKinksInterestRateModel (Three-Slope)

**More flexible model with two kink points:**

```
         ┌ BASE_RATE + U × MULTIPLIER                                    if U < KINK_1
         │
R_borrow = │ RATE_1 + BASE_RATE_2 + (U - KINK_1) × MULTIPLIER_2         if KINK_1 ≤ U < KINK_2
         │
         └ RATE_1 + RATE_2 + (U - KINK_2) × JUMP_MULTIPLIER             if U ≥ KINK_2

Where:
  RATE_1 = BASE_RATE + KINK_1 × MULTIPLIER
  RATE_2 = BASE_RATE_2 + (KINK_2 - KINK_1) × MULTIPLIER_2
```

### 5.4 Supply Rate

**Suppliers earn a portion of borrower interest:**

```
R_supply = us × R_borrow × (1 - reserveFactor)
```

> **Important**: The supply rate uses `us` (supply utilization), NOT the standard utilization `u`.
> This is because only actual `borrows` (not `badDebt`) generate interest income for suppliers.

**Official formula from Venus docs:**
```
              borrows
us = ─────────────────────────────────────────
      cash + borrows - reserves + badDebt

supply_rate = borrow_rate × us × (1 - reserve_factor)
```

**Derivation:**
```
Total interest paid by borrowers = totalBorrows × R_borrow
Protocol takes = interest × reserveFactor
Remaining for suppliers = interest × (1 - reserveFactor)
Per unit supplied = (interest × (1 - reserveFactor)) / totalSupply
                  = (borrows × R_borrow × (1 - reserveFactor)) / (cash + borrows - reserves + badDebt)
                  = us × R_borrow × (1 - reserveFactor)
```

```solidity
function getSupplyRate(
    uint cash, uint borrows, uint reserves, uint badDebt, uint reserveFactorMantissa
) public view returns (uint) {
    uint oneMinusReserveFactor = 1e18 - reserveFactorMantissa;
    uint borrowRate = getBorrowRate(cash, borrows, reserves, badDebt);
    uint rateToPool = borrowRate * oneMinusReserveFactor / 1e18;
    // Note: supply utilization uses borrows (not borrows+badDebt) in numerator
    uint supplyUtilization = borrows * 1e18 / (cash + borrows - reserves + badDebt);
    return supplyUtilization * rateToPool / 1e18;
}
```

---

## 6. Account Liquidity & Risk Management

### 6.1 Key Risk Parameters

| Parameter | Symbol | Typical Value | Purpose |
|-----------|--------|---------------|---------|
| **Collateral Factor** | CF | 75% (0.75e18) | Max borrowing power |
| **Liquidation Threshold** | LT | 80% (0.80e18) | When liquidation triggers |
| **Liquidation Incentive** | LI | 110% (1.10e18) | Liquidator bonus |
| **Close Factor** | CLF | 50% (0.50e18) | Max repayable per liquidation |
| **Reserve Factor** | RF | 20% (0.20e18) | Protocol's interest share |

### 6.2 Why Collateral Factor ≠ Liquidation Threshold

**The Buffer Zone Concept:**

```
                    CF (75%)              LT (80%)
                       │                     │
 ◄────── Safe ────────►│◄──── Buffer ──────►│◄──── Liquidatable ────►
                       │      Zone           │
 Can borrow more       │  Cannot borrow      │  Position at risk
                       │  but not liquidated │
```

This design:
1. Prevents instant liquidation after max borrow
2. Gives users time to react to price movements
3. Creates a safety margin for the protocol

### 6.3 Health Factor

**Common DeFi metric** (used by Aave, similar concept in Venus):

```
                 Σ(collateral_i × price_i × LT_i)
Health Factor = ────────────────────────────────────
                 Σ(borrow_j × price_j)

HF > 1  →  Safe
HF = 1  →  At liquidation threshold
HF < 1  →  Liquidatable
```

### 6.4 Account Liquidity Calculation

**Venus approach** (from `ComptrollerLens.sol:218-295`):

```solidity
function _calculateAccountPosition(
    address account,
    VToken vTokenModify,
    uint redeemTokens,
    uint borrowAmount,
    WeightFunction weightingStrategy  // COLLATERAL_FACTOR or LIQUIDATION_THRESHOLD
) internal view returns (uint oErr, AccountLiquidityLocalVars memory vars) {

    vars.sumCollateral = 0;
    vars.sumBorrowPlusEffects = 0;

    // Iterate through all markets user has entered
    VToken[] memory assets = ComptrollerInterface(comptroller).getAssetsIn(account);

    for (uint i = 0; i < assets.length; ++i) {
        VToken asset = assets[i];

        // Get user's position in this market
        (oErr, vars.vTokenBalance, vars.borrowBalance, vars.exchangeRateMantissa) =
            asset.getAccountSnapshot(account);

        // Get weighting factor (CF or LT based on strategy)
        vars.weightedFactor = Exp({
            mantissa: getEffectiveLtvFactor(account, address(asset), weightingStrategy)
        });

        // Get oracle price
        vars.oraclePriceMantissa = oracle.getUnderlyingPrice(address(asset));
        vars.oraclePrice = Exp({ mantissa: vars.oraclePriceMantissa });
        vars.exchangeRate = Exp({ mantissa: vars.exchangeRateMantissa });

        // tokensToDenom = weightedFactor × exchangeRate × oraclePrice
        vars.tokensToDenom = mul_(mul_(vars.weightedFactor, vars.exchangeRate), vars.oraclePrice);

        // sumCollateral += tokensToDenom × vTokenBalance
        vars.sumCollateral = mul_ScalarTruncateAddUInt(
            vars.tokensToDenom, vars.vTokenBalance, vars.sumCollateral
        );

        // sumBorrowPlusEffects += oraclePrice × borrowBalance
        vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(
            vars.oraclePrice, vars.borrowBalance, vars.sumBorrowPlusEffects
        );

        // If modifying this market, account for the change
        if (asset == vTokenModify) {
            // Redeem reduces collateral
            vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(
                vars.tokensToDenom, redeemTokens, vars.sumBorrowPlusEffects
            );
            // Borrow increases debt
            vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(
                vars.oraclePrice, borrowAmount, vars.sumBorrowPlusEffects
            );
        }
    }

    // Add VAI debt if any
    if (address(vaiController) != address(0)) {
        vars.sumBorrowPlusEffects = add_(
            vars.sumBorrowPlusEffects,
            vaiController.getVAIRepayAmount(account)
        );
    }
}
```

**Mathematical Summary:**

```
Collateral Value = Σᵢ (vTokenBalance_i × exchangeRate_i × price_i × CF_i)

Borrow Value = Σⱼ (borrowBalance_j × price_j) + VAI_debt

Liquidity = max(0, Collateral Value - Borrow Value)
Shortfall = max(0, Borrow Value - Collateral Value)
```

---

## 7. Liquidation Mechanics

### 7.1 Liquidation Conditions

An account is **liquidatable** when:
```
shortfall > 0  (calculated using LIQUIDATION_THRESHOLD)
```

Equivalently:
```
Σ(borrow × price) > Σ(collateral × price × LT)
```

### 7.2 Close Factor Limitation

**Maximum repayable in one transaction:**
```
maxRepay = borrowBalance × closeFactor

Example:
  Borrow = 1000 USDC
  Close Factor = 50%
  Max Repay = 500 USDC per liquidation
```

**Rationale**: Prevents one liquidator from seizing all collateral, allows competition.

### 7.3 Seize Token Calculation

**From `ComptrollerLens.sol:306-323`:**

```
                actualRepayAmount × liquidationIncentive × priceBorrowed
seizeTokens = ────────────────────────────────────────────────────────────
                         priceCollateral × exchangeRate
```

**Derivation:**

1. Value repaid (in USD): `repayAmount × priceBorrowed`
2. Value to seize (with bonus): `repaid_value × liquidationIncentive`
3. Underlying collateral to seize: `seize_value / priceCollateral`
4. vTokens to seize: `underlying_seize / exchangeRate`

### 7.3.1 Protocol Share (from Official Docs)

**The protocol takes a cut of the liquidation incentive:**

```
Protocol Shares = (Collateral Seized ÷ Liquidation Incentive) × Protocol Share %
Liquidator Receives = Collateral Seized - Protocol Shares
```

**Example:**
```
Collateral Seized: $1,100 (with 10% liquidation incentive on $1,000 repay)
Protocol Share %: 5%

Protocol Shares = ($1,100 ÷ 1.10) × 0.05 = $1,000 × 0.05 = $50
Liquidator Receives = $1,100 - $50 = $1,050
Liquidator Profit = $1,050 - $1,000 = $50 (5% instead of 10%)
```

**State variables:**
- Core Pool: `treasuryPercentMantissa`
- Isolated Pools: `protocolSeizeShareMantissa`

```solidity
function _calculateSeizeTokens(
    uint actualRepayAmount,
    uint liquidationIncentiveMantissa,
    uint priceBorrowedMantissa,
    uint priceCollateralMantissa,
    uint exchangeRateMantissa
) internal pure returns (uint seizeTokens) {
    // numerator = liquidationIncentive × priceBorrowed
    Exp memory numerator = mul_(
        Exp({ mantissa: liquidationIncentiveMantissa }),
        Exp({ mantissa: priceBorrowedMantissa })
    );

    // denominator = priceCollateral × exchangeRate
    Exp memory denominator = mul_(
        Exp({ mantissa: priceCollateralMantissa }),
        Exp({ mantissa: exchangeRateMantissa })
    );

    // seizeTokens = actualRepayAmount × (numerator / denominator)
    seizeTokens = mul_ScalarTruncate(div_(numerator, denominator), actualRepayAmount);
}
```

### 7.4 Complete Liquidation Example

```
Initial State:
─────────────
Borrower:
  - Collateral: 1 ETH @ $2000 = $2000 value
  - Borrowed: 1500 USDC
  - Liquidation Threshold: 80%
  - Effective collateral: $2000 × 0.80 = $1600
  - Status: SAFE (1600 > 1500)

ETH Drops to $1800:
───────────────────
  - Collateral value: $1800
  - Effective collateral: $1800 × 0.80 = $1440
  - Borrowed: $1500
  - Status: LIQUIDATABLE (shortfall = $60)

Liquidation (50% close factor):
───────────────────────────────
Liquidator repays: 750 USDC (50% of 1500)

seizeTokens calculation:
  liquidationIncentive = 1.10 (10% bonus)
  priceBorrowed = $1 (USDC)
  priceCollateral = $1800 (ETH)
  exchangeRate = 0.02 (1 vETH = 0.02 ETH)

  seizeAmount_USD = 750 × 1.10 × 1 = $825
  seizeAmount_ETH = 825 / 1800 = 0.4583 ETH
  seizeTokens = 0.4583 / 0.02 = 22.917 vETH

Result:
  - Liquidator pays: 750 USDC
  - Liquidator receives: 22.917 vETH (worth ~$825)
  - Profit: ~$75 (10% of 750)
  - Borrower's remaining debt: 750 USDC
  - Borrower's remaining collateral: 0.5417 ETH
```

### 7.5 Venus Liquidation Types

From [Venus Docs](https://docs-v4.venus.io/guides/liquidation):

| Type | Function | When Used |
|------|----------|-----------|
| **Regular** | `liquidateBorrow()` | Collateral > minLiquidatableCollateral |
| **Batch** | `liquidateAccount()` | Small collateral, still solvent |
| **Debt Recovery** | `healAccount()` | Insolvent, minimal collateral |

---

## 8. Comparison with ERC-4626 Vault Standard

### 8.1 Overview

| Feature | ERC-4626 | Venus vToken |
|---------|----------|--------------|
| **Standard** | EIP-4626 | Custom (Compound-style) |
| **Purpose** | Tokenized yield vaults | Lending/borrowing markets |
| **Share Token** | Generic vault shares | vTokens |
| **Interest Source** | External yield | Borrower payments |
| **Borrowing** | Not supported | Core feature |
| **Collateral** | Not supported | Core feature |
| **Preview Functions** | Native (`previewX`) | Via `exchangeRateCurrent()` |

### 8.2 Function Mapping

| ERC-4626 | Venus Equivalent | Key Difference |
|----------|------------------|----------------|
| `deposit(assets, receiver)` | `mint(mintAmount)` | Venus takes amount, not receiver |
| `mint(shares, receiver)` | N/A | Venus can't mint exact shares |
| `withdraw(assets, receiver, owner)` | `redeemUnderlying()` | Venus uses `redeem` terminology |
| `redeem(shares, receiver, owner)` | `redeem(vTokens)` | Similar concept |
| `convertToShares(assets)` | `assets / exchangeRate` | Manual calculation |
| `convertToAssets(shares)` | `shares × exchangeRate` | Manual calculation |
| `totalAssets()` | `cash + borrows - reserves` | Venus includes lent assets |
| `previewDeposit(assets)` | `assets / exchangeRateCurrent()` | Venus requires tx |
| `maxDeposit(receiver)` | Supply cap check | Venus has caps |
| `maxWithdraw(owner)` | Liquidity check | Venus checks health |

### 8.3 Share Price Mechanisms

**ERC-4626:**
```solidity
function convertToAssets(uint256 shares) public view returns (uint256) {
    uint256 supply = totalSupply();
    return supply == 0 ? shares : shares * totalAssets() / supply;
}

// Note: totalAssets() only counts assets IN the vault
```

**Venus vToken:**
```solidity
function exchangeRateStoredInternal() internal view returns (MathError, uint) {
    if (totalSupply == 0) {
        return (MathError.NO_ERROR, initialExchangeRateMantissa);
    }
    // Includes borrowed assets (not in contract but still "owned")
    return (
        MathError.NO_ERROR,
        (getCash() + totalBorrows - totalReserves) * 1e18 / totalSupply
    );
}
```

### 8.4 Critical Differences

1. **Interest accrual timing**:
   - ERC-4626: Yield typically auto-compounds or is external
   - Venus: Must call `accrueInterest()` before exchange rate reflects reality

2. **Initial exchange rate**:
   - ERC-4626: Usually 1:1 (shares = assets)
   - Venus: Typically 0.02 (50 vTokens per 1 underlying)

3. **Rounding direction**:
   - Both: Round in favor of protocol (down on mint, down on redeem)

4. **Inflation attack mitigation**:
   - ERC-4626 (OpenZeppelin): Virtual shares/assets offset
   - Venus: Non-zero initial exchange rate + first deposit protection

### 8.5 References

- [EIP-4626 Specification](https://eips.ethereum.org/EIPS/eip-4626)
- [OpenZeppelin ERC-4626 Docs](https://docs.openzeppelin.com/contracts/5.x/erc4626)
- [ERC-4626 Interface Explained - RareSkills](https://rareskills.io/post/erc4626)

---

## 9. Risk Parameter Framework

### 9.1 Gauntlet's Methodology

[Gauntlet](https://www.gauntlet.xyz/) is the leading DeFi risk management firm. Their approach:

**Agent-Based Simulation:**
- Model different participant types (borrowers, lenders, liquidators)
- Run thousands of scenarios including tail risks
- Quantify Value-at-Risk (VaR) and failure modes

**Optimization Objective:**
```
Maximize: Protocol Revenue (borrow usage)
Subject to: VaR < threshold
            Liquidations at Risk < threshold
```

**Reference**: [Gauntlet's Parameter Recommendation Methodology](https://www.gauntlet.xyz/resources/gauntlets-parameter-recommendation-methodology)

### 9.2 Collateral Factor Selection

**Factors considered:**
1. **Asset volatility**: Higher volatility → lower CF
2. **Liquidity depth**: Low liquidity → lower CF (harder to liquidate)
3. **Oracle reliability**: Less reliable → lower CF
4. **Historical drawdowns**: Larger drops → lower CF

**Academic insight** from Berkeley DeFi research:
> "The existing liquidation designs well incentivize liquidators but sell excessive amounts of discounted collateral at the borrowers' expenses."

### 9.3 Liquidation Incentive Calibration

**Trade-offs:**
```
Higher Incentive:
  ✓ More liquidator participation
  ✓ Faster liquidations
  ✗ More value extracted from borrowers
  ✗ Higher bad debt risk if too slow

Lower Incentive:
  ✓ Less borrower value extraction
  ✗ Fewer liquidators willing to act
  ✗ Risk of cascade failures
```

### 9.4 Dynamic Risk Considerations

From BIS Working Paper:
> "Higher borrower leverage generally undermines lending resilience, particularly increasing the share of outstanding debt close to being liquidated."

Key metrics to monitor:
- Debt-at-risk (close to liquidation threshold)
- Liquidator activity and profitability
- Oracle latency and accuracy
- Gas price impact on liquidation profitability

---

## 10. Mathematical Derivations

### 10.1 Exchange Rate Invariant

**Claim**: For a supplier who deposits at time t₀ and withdraws at time t₁:

```
underlying_received / underlying_deposited = exchangeRate(t₁) / exchangeRate(t₀)
```

**Proof:**
```
At t₀:
  vTokens_minted = underlying_deposited / exchangeRate(t₀)

At t₁:
  underlying_received = vTokens × exchangeRate(t₁)
                      = (underlying_deposited / exchangeRate(t₀)) × exchangeRate(t₁)
                      = underlying_deposited × (exchangeRate(t₁) / exchangeRate(t₀))
```

### 10.2 Interest Rate Index Equivalence

**Claim**: The index-based system produces identical results to direct compound interest.

**Proof by induction:**

Base case (n=1):
```
After 1 period with rate r₁:
  Direct: Balance = P × (1 + r₁)
  Index:  I₁ = I₀ × (1 + r₁)
          Balance = P × I₁/I₀ = P × (1 + r₁) ✓
```

Inductive step (n → n+1):
```
Assume true for n periods.
After period n+1 with rate r_{n+1}:
  Direct: Balance = P × (1+r₁)×...×(1+rₙ) × (1+r_{n+1})
  Index:  I_{n+1} = Iₙ × (1 + r_{n+1})
                  = I₀ × (1+r₁)×...×(1+rₙ) × (1+r_{n+1})
          Balance = P × I_{n+1}/I₀ = P × (1+r₁)×...×(1+r_{n+1}) ✓
```

### 10.3 Utilization-Interest Relationship

**Economic intuition** behind the kink model:

Below kink (normal operations):
```
Supply ≈ Demand → Moderate rates → Market equilibrium
```

Above kink (liquidity stress):
```
Demand >> Supply → Spike rates → Incentivize deposits, discourage borrows
```

**Mathematical form** (JumpRateModel):
```
R(U) = {
  R₀ + m₁U           if U ≤ U*
  R₀ + m₁U* + m₂(U-U*)  if U > U*
}

Where: m₂ >> m₁ (jump multiplier much larger)
```

---

## 11. Key Formulas Cheat Sheet

### Exchange Rate
```
exchangeRate = (cash + totalBorrows - totalReserves) / totalSupply
```

### Mint/Redeem
```
vTokens = underlyingAmount / exchangeRate
underlying = vTokens × exchangeRate
```

### Borrow Balance
```
currentBalance = principal × (currentBorrowIndex / snapshotBorrowIndex)
```

### Interest Accrual (per block)
```
simpleInterestFactor = borrowRatePerBlock × blockDelta
interestAccumulated = simpleInterestFactor × totalBorrows
newBorrowIndex = oldBorrowIndex × (1 + simpleInterestFactor)
newTotalBorrows = oldTotalBorrows + interestAccumulated
newReserves = oldReserves + interestAccumulated × reserveFactor
```

### Utilization Rate
```
u = (borrows + badDebt) / (cash + borrows + badDebt - reserves)   // for borrow rate
us = borrows / (cash + borrows - reserves + badDebt)              // for supply rate
```

### Borrow Rate (JumpRateModel)
```
If U ≤ kink:  R = baseRate + U × multiplier
If U > kink:  R = baseRate + kink × multiplier + (U - kink) × jumpMultiplier
```

### Supply Rate
```
supplyRate = us × borrowRate × (1 - reserveFactor)
```
(Note: Uses supply utilization `us`, not standard utilization `u`)

### Account Liquidity
```
collateralValue = Σ(vTokenBalance × exchangeRate × price × CF)
borrowValue = Σ(borrowBalance × price) + vaiDebt
liquidity = max(0, collateralValue - borrowValue)
shortfall = max(0, borrowValue - collateralValue)
```

### Health Factor
```
HF = collateralValue (using LT) / borrowValue
Liquidatable when HF < 1
```

### Liquidation Seize
```
seizeTokens = (repayAmount × liquidationIncentive × priceBorrowed) / (priceCollateral × exchangeRate)
```

### Protocol Share (Liquidation)
```
protocolShare = (collateralSeized / liquidationIncentive) × protocolSharePercent
liquidatorReceives = collateralSeized - protocolShare
```

### Max Repay
```
maxRepay = totalBorrowBalance × closeFactor
```

---

## 12. Important State Variables

### vToken State

```solidity
// Supply side
uint public totalSupply;                      // Total vTokens in existence
mapping(address => uint) accountTokens;       // User vToken balances

// Borrow side
uint public totalBorrows;                     // Total borrowed (with interest)
uint public borrowIndex;                      // Cumulative interest index (starts at 1e18)
mapping(address => BorrowSnapshot) accountBorrows;

// Interest tracking
uint public accrualBlockNumber;               // Last block interest was accrued
uint public totalReserves;                    // Protocol's accumulated reserves
uint public reserveFactorMantissa;            // % of interest to reserves (e.g., 0.2e18)

// Configuration
uint public initialExchangeRateMantissa;      // Starting rate (e.g., 0.02e18)
InterestRateModel public interestRateModel;   // Rate calculation contract
ComptrollerInterface public comptroller;      // Risk management
```

### Comptroller State

```solidity
// Per-market configuration
struct Market {
    bool isListed;
    uint collateralFactorMantissa;         // e.g., 0.75e18
    uint liquidationThresholdMantissa;     // e.g., 0.80e18
    uint liquidationIncentiveMantissa;     // e.g., 1.10e18
}

// Global parameters
uint public closeFactorMantissa;           // e.g., 0.50e18
mapping(address => uint) public supplyCaps;
mapping(address => uint) public borrowCaps;

// User tracking
mapping(address => VToken[]) public accountAssets;  // Markets user has entered
```

---

## 13. Flow Diagrams

### Supply Flow
```
User                    vToken                  Comptroller
  │                       │                          │
  │──── mint(amount) ────►│                          │
  │                       │──── mintAllowed() ──────►│
  │                       │◄───────── OK ───────────│
  │                       │                          │
  │                       │ accrueInterest()         │
  │                       │ rate = exchangeRate()    │
  │                       │ tokens = amount/rate     │
  │◄── doTransferIn() ───│                          │
  │                       │                          │
  │                       │ totalSupply += tokens    │
  │                       │ accountTokens += tokens  │
  │◄── vTokens minted ───│                          │
```

### Borrow Flow
```
User                    vToken                  Comptroller
  │                       │                          │
  │──── borrow(amount) ──►│                          │
  │                       │──── borrowAllowed() ────►│
  │                       │   (liquidity check)      │
  │                       │◄───────── OK ───────────│
  │                       │                          │
  │                       │ accrueInterest()         │
  │                       │ oldBal = borrowBalance() │
  │                       │                          │
  │                       │ accountBorrows[user] =   │
  │                       │   {principal: old+amt,   │
  │                       │    interestIndex: idx}   │
  │                       │ totalBorrows += amount   │
  │                       │                          │
  │◄── doTransferOut() ──│                          │
```

### Liquidation Flow
```
Liquidator             vToken(Borrow)          vToken(Collateral)     Comptroller
    │                       │                        │                     │
    │── liquidateBorrow() ─►│                        │                     │
    │   (borrower,amount,   │                        │                     │
    │    collateralVToken)  │                        │                     │
    │                       │                        │                     │
    │                       │─── liquidateBorrowAllowed() ────────────────►│
    │                       │    (shortfall > 0?)    │                     │
    │                       │◄────────── OK ─────────────────────────────│
    │                       │                        │                     │
    │                       │ repayBorrowFresh()     │                     │
    │── doTransferIn() ────►│ (borrower debt -= amt) │                     │
    │                       │                        │                     │
    │                       │─ calculateSeizeTokens ►│                     │
    │                       │                        │                     │
    │                       │───── seize() ─────────►│                     │
    │                       │                        │ borrower -= tokens  │
    │                       │                        │ liquidator += tokens│
    │◄────────────── vTokens (collateral) ──────────│                     │
```

---

## 14. Academic Papers & Resources

### 14.1 Core Academic Papers

#### Interest Rate Models & DeFi Lending

1. **"DeFi Protocols for Loanable Funds: Interest Rates, Liquidity and Market Efficiency"**
   - [arXiv](https://arxiv.org/pdf/2006.13922) | [ACM](https://dl.acm.org/doi/10.1145/3419614.3423254)
   - Reviews methodologies for setting interest rates in Compound, Aave, dYdX
   - Examines market efficiency and Uncovered Interest Parity

2. **"Agents' Behavior and Interest Rate Model Optimization in DeFi Lending"**
   - [Wiley Mathematical Finance](https://onlinelibrary.wiley.com/doi/abs/10.1111/mafi.70002)
   - Theoretical framework for optimal interest rate models
   - Shows optimal control model generates kink-like shapes

3. **"Optimal Risk-Aware Interest Rates for Decentralized Lending Protocols"**
   - [arXiv (2025)](https://arxiv.org/html/2502.19862v1)
   - Mathematical model using point processes
   - Calibrated on AAVE v3 USDT pool data

#### Liquidation Mechanics & Risk

4. **"An Empirical Study of DeFi Liquidations: Incentives, Risks, and Instabilities"**
   - [arXiv](https://arxiv.org/abs/2106.06389) | [ACM IMC 2021](https://dl.acm.org/doi/10.1145/3487552.3487811) | [Berkeley PDF](https://berkeley-defi.github.io/assets/material/A%20Study%20of%20DeFi%20Liquidations.pdf)
   - First comprehensive study of DeFi liquidations
   - Covers Aave, Compound, MakerDAO, dYdX

5. **"DeFi Leverage" (BIS Working Paper No. 1171)**
   - [Bank for International Settlements](https://www.bis.org/publ/work1171.pdf)
   - Documents individual wallet leverage in DeFi
   - Analyzes impact of leverage on lending resilience

6. **"DeFi Liquidations: Volatility and Liquidity" (OECD)**
   - [OECD PDF](https://www.oecd.org/content/dam/oecd/en/publications/reports/2023/07/defi-liquidations_89cba79d/0524faaf-en.pdf)
   - On-chain analysis of major lending protocols
   - Investigates systemic risk from liquidations

7. **"On the Fragility of DeFi Lending"**
   - [LSE Paper](https://personal.lse.ac.uk/yuan/papers/defi.pdf)
   - Bank of Canada & LSE researchers
   - Identifies unique fragility from self-fulfilling cycles

8. **"Locked in, Levered Up: Risk, Return, and Ruin in DeFi Lending"**
   - [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0890838925001416)
   - University of Sydney research
   - Trade-offs in permissionless architecture

### 14.2 Protocol Documentation

#### Compound (Venus is based on Compound)
- [Compound Whitepaper](https://compound.finance/documents/Compound.Whitepaper.pdf)
- [Compound v2 Documentation](https://docs.compound.finance/v2/)
- [Compound cTokens Reference](https://docs.compound.finance/v2/ctokens/)
- [Compound GitHub](https://github.com/compound-finance/compound-protocol)

#### Venus Protocol
- [Venus Documentation](https://docs-v4.venus.io/)
- [Venus Liquidation Guide](https://docs-v4.venus.io/guides/liquidation)
- [Venus GitHub](https://github.com/VenusProtocol/venus-protocol)

#### ERC-4626
- [EIP-4626 Specification](https://eips.ethereum.org/EIPS/eip-4626)
- [OpenZeppelin ERC-4626 Docs](https://docs.openzeppelin.com/contracts/5.x/erc4626)
- [Ethereum.org ERC-4626 Guide](https://ethereum.org/developers/docs/standards/tokens/erc-4626/)

### 14.3 Technical Deep Dives

#### Fixed-Point Arithmetic
- [Fixed Point Arithmetic in Solidity - RareSkills](https://rareskills.io/post/solidity-fixed-point)
- [PRBMath Library - GitHub](https://github.com/PaulRBerg/prb-math)
- [Math in Solidity (Part 1) - Medium](https://medium.com/coinmonks/math-in-solidity-part-1-numbers-384c8377f26d)

#### Interest Rate Models
- [Aave/Compound Interest Rate Model - RareSkills](https://rareskills.io/post/aave-interest-rate-model)
- [Continuous Compounding in DeFi - DeFinomist](https://definomist.com/post/continuous-compounding-strategies-for-defi-borrowing-and-lending/)

#### Collateral & Liquidation
- [DeFi Liquidations and Collateral - RareSkills](https://rareskills.io/post/defi-liquidations-collateral)
- [GARP Whitepaper: Risk Management in DeFi](https://www.garp.org/hubfs/Whitepapers/a2r5d0000065tecAAA_RiskIntel.Whitepaper.DeFi.2.9.23.pdf)

### 14.4 Risk Assessment Resources

- [Gauntlet's Parameter Recommendation Methodology](https://www.gauntlet.xyz/resources/gauntlets-parameter-recommendation-methodology)
- [Gauntlet's Model Methodology](https://medium.com/gauntlet-networks/gauntlets-model-methodology-ea150ff0bafd)
- [How to Measure Risk Models - Gauntlet](https://www.gauntlet.xyz/resources/how-to-measure-risk-models-part-i)
- [Aave Market Risk Assessment - Gauntlet](https://www.gauntlet.xyz/resources/aave-market-risk-assessment)

### 14.5 AMM & Vault Mathematics

- [Axioms for Automated Market Makers - Operations Research](https://pubsonline.informs.org/doi/10.1287/opre.2022.0520) | [arXiv](https://arxiv.org/html/2210.01227v4)
- [Automated Market Making and Loss-Versus-Rebalancing](https://anthonyleezhang.github.io/pdfs/lvr.pdf)
- [CMU: Liquidity-Sensitive AMM](https://www.cs.cmu.edu/~sandholm/liquidity-sensitive%20automated%20market%20maker.teac.pdf)
- [Homogeneous Properties of AMMs - Copenhagen](https://arxiv.org/pdf/2105.02782)

---

## Quick Reference Card

### Constants to Remember
| Constant | Value | Meaning |
|----------|-------|---------|
| `1e18` | 10^18 | 100% / 1.0 in mantissa |
| `0.75e18` | 7.5×10^17 | Typical collateral factor (75%) |
| `0.80e18` | 8×10^17 | Typical liquidation threshold (80%) |
| `1.10e18` | 1.1×10^18 | Typical liquidation incentive (110%) |
| `0.50e18` | 5×10^17 | Typical close factor (50%) |
| `0.20e18` | 2×10^17 | Typical reserve factor (20%) |
| `0.02e18` | 2×10^16 | Common initial exchange rate |

### Critical Safety Checks
1. **Before Borrow**: `liquidity > 0` (using collateral factor)
2. **Before Liquidation**: `shortfall > 0` (using liquidation threshold)
3. **Before Redeem**: Won't cause `shortfall > 0`
4. **Interest Accrual**: Called before EVERY state change
5. **Reentrancy**: `nonReentrant` modifier on external functions

### Common Exam Questions
1. Derive the exchange rate formula and explain why it increases
2. Explain the borrow index mechanism and prove its equivalence to compound interest
3. Compare ERC-4626 share pricing with vToken exchange rate
4. Calculate liquidation scenarios with given parameters
5. Explain why CF < LT and the buffer zone concept
6. Derive supply rate from borrow rate and explain the reserve factor

---

*Good luck on your master's exam!*
