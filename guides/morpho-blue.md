# Morpho Blue Protocol - Comprehensive Study Guide

## Table of Contents
1. [Protocol Philosophy & Design Principles](#1-protocol-philosophy--design-principles)
2. [Architecture Overview](#2-architecture-overview)
3. [Core Data Structures](#3-core-data-structures)
4. [The Shares System - Critical Math](#4-the-shares-system---critical-math)
5. [Comparison with EIP-4626 Vault Standard](#5-comparison-with-eip-4626-vault-standard)
6. [Interest Accrual Mechanism](#6-interest-accrual-mechanism)
7. [Health Factor & Liquidations](#7-health-factor--liquidations)
8. [Key Protocol Functions](#8-key-protocol-functions)
9. [Callbacks & Flash Operations](#9-callbacks--flash-operations)
10. [Authorization & EIP-712 Signatures](#10-authorization--eip-712-signatures)
11. [Security Considerations](#11-security-considerations)
12. [Key Algorithms to Remember](#12-key-algorithms-to-remember)
13. [Potential Exam Questions](#13-potential-exam-questions)
14. [References & Further Reading](#14-references--further-reading)

---

## 1. Protocol Philosophy & Design Principles

### Why Morpho Blue Exists
Morpho Blue is a **governance-minimized, immutable lending protocol** that launched on Ethereum mainnet in February 2024. Unlike Aave or Compound, it:
- Has **no governance** for market parameters once deployed
- Allows **permissionless market creation** with any oracle/IRM combination
- Uses a **singleton pattern** (one contract manages ALL markets)
- Provides **free flash loans** (no fees)
- Is extremely minimal: **~650 lines of Solidity code**

### The Two-Layer Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                      MORPHO VAULTS (Layer 2)                     │
│  - Managed by third-party curators                               │
│  - Aggregates multiple Morpho Blue markets                       │
│  - Implements ERC-4626 standard                                  │
│  - Provides automated yield strategies                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MORPHO BLUE (Layer 1)                        │
│  - Primitive lending protocol                                    │
│  - Isolated markets with immutable parameters                    │
│  - NO share tokens (positions tracked internally)                │
│  - Direct interaction for sophisticated users                    │
└─────────────────────────────────────────────────────────────────┘
```

### The Singleton Pattern
```
Traditional (Aave/Compound): 1 Pool = 1 Contract (or proxy)
Morpho Blue:                 1 Contract = ALL Markets
```

**Why singleton?**
- Gas efficiency (no proxy calls between contracts)
- Simpler integrations (single entry point)
- Unified liquidity view
- Flash loans can access ALL assets in one call

### Key Differentiators from Compound/Aave
| Feature | Compound/Aave | Morpho Blue |
|---------|--------------|-------------|
| Market Creation | Governance-controlled | Permissionless |
| Parameters | Can be changed | Immutable once created |
| Token Tracking | cTokens/aTokens | Internal shares (no ERC20) |
| Markets | Multi-asset pools | Isolated pairs |
| Flash Loan Fee | 0.09% (Aave) | 0% (Free) |
| Code Complexity | ~10,000+ lines | ~650 lines |

---

## 2. Architecture Overview

### Contract Structure
```
Morpho.sol (555 lines) ─── Single entry point for ALL operations
    │
    ├── Libraries (internal, used by Morpho.sol):
    │   ├── MathLib.sol        → Fixed-point arithmetic (WAD math)
    │   ├── SharesMathLib.sol  → Asset ↔ Share conversions
    │   ├── MarketParamsLib.sol → Market ID computation (keccak256)
    │   ├── UtilsLib.sol       → Helper functions (min, exactlyOneZero)
    │   ├── SafeTransferLib.sol → Safe ERC20 transfers
    │   ├── ConstantsLib.sol   → Protocol constants
    │   ├── ErrorsLib.sol      → Error definitions
    │   └── EventsLib.sol      → Event definitions
    │
    ├── Periphery Libraries (for integrators, NOT used by core):
    │   ├── MorphoLib.sol        → Storage access helpers
    │   ├── MorphoBalancesLib.sol → Interest-accrued balance queries
    │   └── MorphoStorageLib.sol  → Direct storage slot access
    │
    └── Pluggable Components (external contracts):
        ├── IOracle   → Price feeds (1e36 scaling)
        └── IIrm      → Interest Rate Models
```

### Market Identification
Each market is uniquely identified by `Id` (bytes32), computed as:
```solidity
// MarketParamsLib.sol
function id(MarketParams memory marketParams) internal pure returns (Id) {
    return Id.wrap(keccak256(abi.encode(marketParams)));
}

// Which is equivalent to:
Id = keccak256(abi.encode(loanToken, collateralToken, oracle, irm, lltv))
```

**Critical Insight**: Same tokens + different oracle = DIFFERENT market

---

## 3. Core Data Structures

### MarketParams (Defines a Market)
```solidity
struct MarketParams {
    address loanToken;        // Token being lent/borrowed (e.g., USDC)
    address collateralToken;  // Token used as collateral (e.g., WETH)
    address oracle;           // Price oracle for collateral/loan conversion
    address irm;              // Interest Rate Model contract
    uint256 lltv;             // Liquidation Loan-To-Value (e.g., 0.86e18 = 86%)
}
```

### Position (User's State in a Market)
```solidity
struct Position {
    uint256 supplyShares;   // NOT assets! Shares of supply pool
    uint128 borrowShares;   // NOT assets! Shares of borrow pool
    uint128 collateral;     // Actual collateral ASSETS (not shares)
}
```

**Critical insight**: Supply and borrow are tracked in SHARES, collateral is tracked in ASSETS. This asymmetry exists because collateral doesn't accrue interest.

### Market (Global Market State)
```solidity
struct Market {
    uint128 totalSupplyAssets;   // Total assets supplied (increases with interest)
    uint128 totalSupplyShares;   // Total supply shares issued (mostly constant)
    uint128 totalBorrowAssets;   // Total assets borrowed (increases with interest)
    uint128 totalBorrowShares;   // Total borrow shares issued (mostly constant)
    uint128 lastUpdate;          // Timestamp of last interest accrual
    uint128 fee;                 // Protocol fee (max 25%, in WAD)
}
```

---

## 4. The Shares System - Critical Math

### Why Shares Instead of Assets?
Interest accrual problem: If Alice supplies 100 USDC, then interest accrues, how do we track her new balance?

**Option 1 (Bad)**: Update every user's balance → O(n) gas cost
**Option 2 (Morpho)**: Use shares → O(1) gas cost

### The Core Conversion Formulas

```solidity
// SharesMathLib.sol - The heart of Morpho's accounting

uint256 constant VIRTUAL_SHARES = 1e6;  // Anti-inflation attack
uint256 constant VIRTUAL_ASSETS = 1;    // Anti-inflation attack

// Assets → Shares (rounding DOWN - protocol favored)
function toSharesDown(uint256 assets, uint256 totalAssets, uint256 totalShares)
    internal pure returns (uint256)
{
    return assets.mulDivDown(totalShares + VIRTUAL_SHARES, totalAssets + VIRTUAL_ASSETS);
}

// Shares → Assets (rounding DOWN - protocol favored)
function toAssetsDown(uint256 shares, uint256 totalAssets, uint256 totalShares)
    internal pure returns (uint256)
{
    return shares.mulDivDown(totalAssets + VIRTUAL_ASSETS, totalShares + VIRTUAL_SHARES);
}

// Assets → Shares (rounding UP - protocol favored for debt)
function toSharesUp(uint256 assets, uint256 totalAssets, uint256 totalShares)
    internal pure returns (uint256)
{
    return assets.mulDivUp(totalShares + VIRTUAL_SHARES, totalAssets + VIRTUAL_ASSETS);
}

// Shares → Assets (rounding UP - protocol favored for withdrawals)
function toAssetsUp(uint256 shares, uint256 totalAssets, uint256 totalShares)
    internal pure returns (uint256)
{
    return shares.mulDivUp(totalAssets + VIRTUAL_ASSETS, totalShares + VIRTUAL_SHARES);
}
```

### The Share Price Concept
```
Share Price = (totalAssets + VIRTUAL_ASSETS) / (totalShares + VIRTUAL_SHARES)

Initial state (empty market):
  sharePrice = (0 + 1) / (0 + 1e6) = 1e-6

After 1000 assets supplied with 1000 shares:
  sharePrice = (1000 + 1) / (1000 + 1e6) ≈ 0.001

After interest makes totalAssets = 1100:
  sharePrice = (1100 + 1) / (1000 + 1e6) ≈ 0.0011
```

### Virtual Shares: Inflation Attack Prevention

**The Attack (without virtual shares)**:
1. Attacker is first depositor, deposits 1 wei
2. Attacker donates 1e18 tokens directly to contract
3. Share price = 1e18 assets / 1 share = 1e18
4. Victim deposits 0.9e18 → gets 0 shares (rounded down)
5. Attacker withdraws, stealing victim's deposit

**The Defense (with virtual shares)**:
```
Initial state: totalAssets=0, totalShares=0
With virtual: effectiveAssets=1, effectiveShares=1e6

If attacker donates 1e18:
effectiveAssets = 1e18 + 1 ≈ 1e18
effectiveShares = 0 + 1e6 = 1e6
Share price = 1e18 / 1e6 = 1e12 (much lower!)

Victim deposits 0.9e18:
shares = 0.9e18 * 1e6 / 1e18 = 0.9e6 shares ✓ (victim gets shares!)
```

### Rounding Direction (CRITICAL for Security)

| Operation | Calculation | Rounding | Rationale |
|-----------|-------------|----------|-----------|
| Supply | assets → shares | **Down** | User gets fewer shares (protocol protected) |
| Withdraw | shares → assets | **Down** | User gets fewer assets (protocol protected) |
| Withdraw | assets → shares | **Up** | User burns more shares (protocol protected) |
| Borrow | assets → shares | **Up** | User gets more debt shares (protocol protected) |
| Borrow | shares → assets | **Down** | User gets fewer assets (protocol protected) |
| Repay | assets → shares | **Down** | User burns fewer shares (protocol protected) |
| Repay | shares → assets | **Up** | User pays more assets (protocol protected) |

**Golden Rule**: Always round in favor of the protocol to prevent economic exploits.

---

## 5. Comparison with EIP-4626 Vault Standard

### EIP-4626 Overview
EIP-4626 is the "Tokenized Vault Standard" - a standard API for yield-bearing vaults that hold a single underlying ERC20 token.

### Key EIP-4626 Functions vs Morpho Blue

| EIP-4626 Function | Morpho Blue Equivalent | Key Difference |
|-------------------|----------------------|----------------|
| `deposit(assets, receiver)` | `supply(marketParams, assets, 0, onBehalf, data)` | Morpho has no share token |
| `mint(shares, receiver)` | `supply(marketParams, 0, shares, onBehalf, data)` | Morpho allows either input |
| `withdraw(assets, receiver, owner)` | `withdraw(marketParams, assets, 0, onBehalf, receiver)` | Morpho has `onBehalf` pattern |
| `redeem(shares, receiver, owner)` | `withdraw(marketParams, 0, shares, onBehalf, receiver)` | Same flexibility |
| `convertToShares(assets)` | `SharesMathLib.toSharesDown()` | View function |
| `convertToAssets(shares)` | `SharesMathLib.toAssetsDown()` | View function |
| `totalAssets()` | `market[id].totalSupplyAssets` | Per-market storage |
| `totalSupply()` | `market[id].totalSupplyShares` | Internal tracking |

### Share Calculation Formula Comparison

**EIP-4626 Standard Formula**:
```solidity
shares = assets * totalSupply / totalAssets
assets = shares * totalAssets / totalSupply
```

**Morpho Blue Formula (with virtual offset)**:
```solidity
shares = assets * (totalSupply + VIRTUAL_SHARES) / (totalAssets + VIRTUAL_ASSETS)
assets = shares * (totalAssets + VIRTUAL_ASSETS) / (totalSupply + VIRTUAL_SHARES)

// Where VIRTUAL_SHARES = 1e6, VIRTUAL_ASSETS = 1
```

### Key Architectural Differences

| Aspect | EIP-4626 | Morpho Blue |
|--------|----------|-------------|
| **Share Representation** | ERC20 token | Internal uint256 |
| **Transferability** | Shares are transferable | Positions are not transferable |
| **Multi-asset** | Single underlying asset | Each market has loan + collateral |
| **Virtual Offset** | Optional (OpenZeppelin uses it) | Mandatory (1e6 shares, 1 asset) |
| **Interest Model** | Implementation-specific | Pluggable IRM contracts |
| **Rounding** | Must favor vault | Must favor protocol |

### OpenZeppelin's ERC4626 Inflation Attack Defense
```solidity
// OpenZeppelin uses decimalsOffset to inflate virtual shares
function _decimalsOffset() internal view virtual returns (uint8) {
    return 0; // Can be overridden to add virtual shares
}

// With offset = 3, virtual shares = 10^3 = 1000
```

**Morpho's approach** is more aggressive: `VIRTUAL_SHARES = 1e6` provides stronger protection.

---

## 6. Interest Accrual Mechanism

### When Does Interest Accrue?
Interest accrues **lazily** - only when `_accrueInterest()` is called by:
- `supply()`, `withdraw()`, `borrow()`, `repay()`
- `withdrawCollateral()` (needs health check)
- `liquidate()`
- `setFee()` (before fee change)
- `accrueInterest()` (explicit public call)

**NOT called on**: `supplyCollateral()` (gas optimization - collateral doesn't earn interest)

### The Interest Accrual Algorithm
```solidity
function _accrueInterest(MarketParams memory marketParams, Id id) internal {
    uint256 elapsed = block.timestamp - market[id].lastUpdate;
    if (elapsed == 0) return;  // No time passed, no interest

    if (marketParams.irm != address(0)) {
        // 1. Get per-second borrow rate from IRM (WAD-scaled)
        uint256 borrowRate = IIrm(marketParams.irm).borrowRate(marketParams, market[id]);

        // 2. Calculate interest using Taylor approximation
        uint256 interest = market[id].totalBorrowAssets.wMulDown(
            borrowRate.wTaylorCompounded(elapsed)
        );

        // 3. CRITICAL: Interest increases BOTH supply and borrow assets
        market[id].totalBorrowAssets += interest;
        market[id].totalSupplyAssets += interest;

        // 4. Protocol fee handling (if fee > 0)
        if (market[id].fee != 0) {
            uint256 feeAmount = interest.wMulDown(market[id].fee);
            // Fee recipient gets supply shares for fee amount
            uint256 feeShares = feeAmount.toSharesDown(
                market[id].totalSupplyAssets - feeAmount,
                market[id].totalSupplyShares
            );
            position[id][feeRecipient].supplyShares += feeShares;
            market[id].totalSupplyShares += feeShares;
        }
    }
    market[id].lastUpdate = uint128(block.timestamp);
}
```

### Taylor Series Approximation

For continuous compounding: `FinalValue = Principal * e^(rate * time)`
Interest factor: `e^(rate * time) - 1`

Taylor expansion of `e^x - 1`:
```
e^x - 1 = x + x²/2! + x³/3! + x⁴/4! + ...
        ≈ x + x²/2 + x³/6   (first 3 terms)
```

```solidity
// MathLib.sol
function wTaylorCompounded(uint256 x, uint256 n) internal pure returns (uint256) {
    uint256 firstTerm = x * n;                                      // rate * time
    uint256 secondTerm = mulDivDown(firstTerm, firstTerm, 2 * WAD); // (rate*time)²/2
    uint256 thirdTerm = mulDivDown(secondTerm, firstTerm, 3 * WAD); // (rate*time)³/6
    return firstTerm + secondTerm + thirdTerm;
}
```

**Why Taylor approximation instead of exact `e^x`?**
1. EVM has no native exponentiation for decimals
2. Exact calculation requires infinite precision or lookup tables
3. Taylor is accurate for small x (rate * elapsed is typically small)
4. Gas efficient (3 multiplications)

### Interest Distribution Visualization
```
Before interest (t=0):
├── totalSupplyAssets: 10,000
├── totalSupplyShares: 10,000  → Supply share price: 1.0
├── totalBorrowAssets: 5,000
└── totalBorrowShares: 5,000   → Borrow share price: 1.0

After 100 interest accrues (t=1):
├── totalSupplyAssets: 10,100  (+100)
├── totalSupplyShares: 10,000  (unchanged!) → Supply share price: 1.01
├── totalBorrowAssets: 5,100   (+100)
└── totalBorrowShares: 5,000   (unchanged!) → Borrow share price: 1.02

Alice's 1000 supply shares now worth: 1000 * 1.01 = 1010 assets
Bob's 500 borrow shares now owe:     500 * 1.02 = 510 assets
```

---

## 7. Health Factor & Liquidations

### Health Check Formula
```solidity
function _isHealthy(
    MarketParams memory marketParams,
    Id id,
    address borrower,
    uint256 collateralPrice
) internal view returns (bool) {
    // Calculate debt in asset terms (rounded UP against borrower)
    uint256 borrowed = uint256(position[id][borrower].borrowShares)
        .toAssetsUp(market[id].totalBorrowAssets, market[id].totalBorrowShares);

    // Calculate max borrowable (collateral value * LLTV)
    uint256 maxBorrow = uint256(position[id][borrower].collateral)
        .mulDivDown(collateralPrice, ORACLE_PRICE_SCALE)  // collateral in loan terms
        .wMulDown(marketParams.lltv);                      // apply LLTV

    return maxBorrow >= borrowed;
}
```

### Oracle Price Scaling
```solidity
// IOracle.sol
/// @notice Returns the price of 1 asset of collateral token quoted in 1 asset of loan token,
/// scaled by 1e36.
function price() external view returns (uint256);
```

**Example**: ETH/USDC where 1 ETH = $2000, ETH has 18 decimals, USDC has 6:
```
price = (1e18 wei ETH value in USDC) / 1e18 * 1e36
      = 2000e6 * 1e36 / 1e18
      = 2000e24

To convert 1 ETH to USDC value:
collateralValue = 1e18 * 2000e24 / 1e36 = 2000e6 ✓
```

### Liquidation Incentive Factor (LIF)

```solidity
// In liquidate() function
uint256 liquidationIncentiveFactor = UtilsLib.min(
    MAX_LIQUIDATION_INCENTIVE_FACTOR,  // 1.15e18 (15% max bonus)
    WAD.wDivDown(WAD - LIQUIDATION_CURSOR.wMulDown(WAD - marketParams.lltv))
);

// Formula: LIF = 1 / (1 - cursor * (1 - LLTV))
// Where: cursor = 0.3 (30%), capped at 1.15 (15% bonus)
```

**Worked Example**:
```
LLTV = 86% (0.86e18)
cursor = 30% (0.3e18)

denominator = 1 - 0.3 * (1 - 0.86)
            = 1 - 0.3 * 0.14
            = 1 - 0.042
            = 0.958

LIF = 1 / 0.958 = 1.0438 (4.38% liquidation bonus)

If LLTV = 50%:
denominator = 1 - 0.3 * 0.5 = 0.85
LIF = 1 / 0.85 = 1.176 → capped at 1.15 (15% max)
```

### Bad Debt Socialization
When a borrower's collateral is depleted but debt remains:
```solidity
if (position[id][borrower].collateral == 0) {
    badDebtShares = position[id][borrower].borrowShares;
    badDebtAssets = UtilsLib.min(
        market[id].totalBorrowAssets,
        badDebtShares.toAssetsUp(market[id].totalBorrowAssets, market[id].totalBorrowShares)
    );

    // Socialize the loss across ALL suppliers
    market[id].totalBorrowAssets -= badDebtAssets;
    market[id].totalSupplyAssets -= badDebtAssets;  // <-- Suppliers take the hit!
    market[id].totalBorrowShares -= badDebtShares;
    position[id][borrower].borrowShares = 0;
}
```

**Effect**: Supply share price decreases proportionally to the bad debt, spreading the loss among all suppliers.

---

## 8. Key Protocol Functions

### Supply Flow
```solidity
function supply(
    MarketParams memory marketParams,
    uint256 assets,      // Amount of loan token to supply
    uint256 shares,      // OR amount of shares to mint (exactly one must be 0)
    address onBehalf,    // Who receives the supply position
    bytes calldata data  // Callback data (empty if no callback)
) external returns (uint256 assetsSupplied, uint256 sharesSupplied);
```

**Algorithm**:
1. Verify market exists (`lastUpdate != 0`)
2. Verify exactly one of assets/shares is non-zero
3. Accrue interest (updates share prices)
4. Convert: `assets → shares` (toSharesDown) OR `shares → assets` (toAssetsUp)
5. Update user's `supplyShares`
6. Update market totals
7. Execute callback (if data provided) - **before** token transfer!
8. Transfer tokens FROM caller TO Morpho

### Borrow Flow
```solidity
function borrow(
    MarketParams memory marketParams,
    uint256 assets,
    uint256 shares,
    address onBehalf,    // Who owns the collateral and takes on debt
    address receiver     // Who receives the borrowed tokens
) external returns (uint256 assetsBorrowed, uint256 sharesBorrowed);
```

**Algorithm**:
1. Verify market exists, caller is authorized for `onBehalf`
2. Accrue interest
3. Convert: `assets → shares` (toSharesUp) OR `shares → assets` (toAssetsDown)
4. Update user's `borrowShares`
5. Update market totals
6. **Check health** (CRITICAL - after state change)
7. **Check liquidity** (`totalBorrow ≤ totalSupply`)
8. Transfer tokens FROM Morpho TO receiver

### Liquidate Flow
```solidity
function liquidate(
    MarketParams memory marketParams,
    address borrower,
    uint256 seizedAssets,  // Amount of collateral to seize
    uint256 repaidShares,  // OR amount of debt shares to repay
    bytes calldata data
) external returns (uint256 seized, uint256 repaid);
```

**Algorithm**:
1. Verify position is unhealthy
2. Calculate LIF based on market LLTV
3. Convert between seizedAssets and repaidShares using LIF
4. Update borrower's position (reduce debt and collateral)
5. Handle bad debt if collateral fully depleted
6. Transfer collateral TO liquidator
7. Execute callback (enables flash liquidation)
8. Transfer loan tokens FROM liquidator TO Morpho

---

## 9. Callbacks & Flash Operations

### Callback Interfaces
```solidity
interface IMorphoSupplyCallback {
    function onMorphoSupply(uint256 assets, bytes calldata data) external;
}

interface IMorphoRepayCallback {
    function onMorphoRepay(uint256 assets, bytes calldata data) external;
}

interface IMorphoSupplyCollateralCallback {
    function onMorphoSupplyCollateral(uint256 assets, bytes calldata data) external;
}

interface IMorphoLiquidateCallback {
    function onMorphoLiquidate(uint256 repaidAssets, bytes calldata data) external;
}

interface IMorphoFlashLoanCallback {
    function onMorphoFlashLoan(uint256 assets, bytes calldata data) external;
}
```

### The Callback Pattern (Flash Mint)
```solidity
// In supply():
// 1. State is already updated (user has supply shares)
if (data.length > 0) IMorphoSupplyCallback(msg.sender).onMorphoSupply(assets, data);
// 2. Callback executes - user can do arbitrage, leverage, etc.
// 3. User doesn't need to have tokens yet!
IERC20(marketParams.loanToken).safeTransferFrom(msg.sender, address(this), assets);
// 4. Tokens transferred LAST
```

**Use Cases**:
- Flash mint: Get shares → use in callback → provide tokens
- Flash liquidation: Get collateral → sell in callback → repay loan token
- Leveraged positions: Borrow → swap → supply as collateral → borrow more

### Flash Loans
```solidity
function flashLoan(address token, uint256 assets, bytes calldata data) external {
    require(assets != 0, ErrorsLib.ZERO_ASSETS);

    emit EventsLib.FlashLoan(msg.sender, token, assets);

    // 1. Send tokens first
    IERC20(token).safeTransfer(msg.sender, assets);

    // 2. Callback - borrower uses the tokens
    IMorphoFlashLoanCallback(msg.sender).onMorphoFlashLoan(assets, data);

    // 3. Tokens must be returned (no fee!)
    IERC20(token).safeTransferFrom(msg.sender, address(this), assets);
}
```

**Key Features**:
- **Zero fees** (unlike Aave's 0.09%)
- Can borrow ANY token held by Morpho (liquidity + collateral from all markets)
- Simple implementation (no ERC-3156 compliance, but easily wrapped)

---

## 10. Authorization & EIP-712 Signatures

### Authorization System
```solidity
// Direct authorization
function setAuthorization(address authorized, bool newIsAuthorized) external;

// Signature-based authorization (gasless)
function setAuthorizationWithSig(
    Authorization memory authorization,
    Signature calldata signature
) external;
```

### EIP-712 Domain
```solidity
bytes32 constant DOMAIN_TYPEHASH =
    keccak256("EIP712Domain(uint256 chainId,address verifyingContract)");

bytes32 constant AUTHORIZATION_TYPEHASH = keccak256(
    "Authorization(address authorizer,address authorized,bool isAuthorized,uint256 nonce,uint256 deadline)"
);

// In constructor:
DOMAIN_SEPARATOR = keccak256(abi.encode(DOMAIN_TYPEHASH, block.chainid, address(this)));
```

### Signature Verification
```solidity
function setAuthorizationWithSig(Authorization memory authorization, Signature calldata signature) external {
    require(block.timestamp <= authorization.deadline, ErrorsLib.SIGNATURE_EXPIRED);
    require(authorization.nonce == nonce[authorization.authorizer]++, ErrorsLib.INVALID_NONCE);

    bytes32 hashStruct = keccak256(abi.encode(AUTHORIZATION_TYPEHASH, authorization));
    bytes32 digest = keccak256(bytes.concat("\x19\x01", DOMAIN_SEPARATOR, hashStruct));
    address signatory = ecrecover(digest, signature.v, signature.r, signature.s);

    require(signatory != address(0) && authorization.authorizer == signatory, ErrorsLib.INVALID_SIGNATURE);

    isAuthorized[authorization.authorizer][authorization.authorized] = authorization.isAuthorized;
}
```

---

## 11. Security Considerations

### Reentrancy Protection
Morpho uses **checks-effects-interactions** pattern with a twist:
- State updates happen BEFORE external calls
- Callbacks happen BEFORE final token transfer
- Token transfers are the LAST operation

```solidity
// supply() pattern:
position[id][onBehalf].supplyShares += shares;     // Effect
market[id].totalSupplyShares += shares;            // Effect
if (data.length > 0) callback();                   // Interaction (callback)
IERC20.safeTransferFrom(msg.sender, ...);          // Interaction (transfer)
```

### Token Assumptions (from `IMorpho.sol:createMarket()`)
```
✓ ERC-20 compliant (can omit return values on transfer/transferFrom)
✓ Balance only decreases on transfer/transferFrom (no burn functions)
✓ No reentrant tokens
✓ No fee-on-transfer tokens
✓ No rebasing tokens
✗ Tokens that change balance unexpectedly
```

### Oracle Assumptions
```
✓ Returns price with correct 1e36 scaling
✓ Price cannot change faster than LLTV * LIF in one transaction
✗ Vault-based oracles using AUM (vulnerable to donation attacks)
```

### Front-Running Risks
```
Scenario: User wants to repay exact debt
1. Attacker front-runs with small repay (1 wei)
2. User's repay reverts (trying to repay more shares than exist)

Mitigation: Use `shares` parameter for full repayment:
repay(marketParams, 0, borrowShares, onBehalf, "")
```

### Share Price Manipulation Limits
```
- Markets with < 1e4 borrowed assets can have share price manipulated
- totalBorrowShares could overflow if price is manipulated too high
- Mitigation: Virtual shares provide base protection
```

---

## 12. Key Algorithms to Remember

### 1. Share Conversion with Virtual Offset
```
shares = assets × (totalShares + 1e6) / (totalAssets + 1)
assets = shares × (totalAssets + 1) / (totalShares + 1e6)
```

### 2. Taylor Series for Compound Interest
```
e^x - 1 ≈ x + x²/2 + x³/6
Where x = borrowRate × elapsedTime
```

### 3. Liquidation Incentive Factor
```
LIF = min(1.15, 1 / (1 - 0.3 × (1 - LLTV)))
```

### 4. Health Check
```
healthy = (collateral × price / 1e36 × LLTV) ≥ debt
```

### 5. Market ID
```
Id = keccak256(abi.encode(loanToken, collateralToken, oracle, irm, lltv))
```

### 6. Rounding Rules
```
Supply:    assets → shares (DOWN)
Withdraw:  shares → assets (DOWN), assets → shares (UP)
Borrow:    assets → shares (UP), shares → assets (DOWN)
Repay:     assets → shares (DOWN), shares → assets (UP)
```

---

## 13. Potential Exam Questions

### Conceptual Questions

**Q: Why does Morpho use shares instead of directly tracking assets?**
A: Gas efficiency for interest accrual. Instead of updating every user's balance O(n), only update totalAssets O(1). Users' share counts stay constant; share value increases.

**Q: Explain the virtual shares mechanism and why VIRTUAL_SHARES = 1e6.**
A: Prevents inflation attacks. The asymmetry (1e6 shares vs 1 asset) means share price starts very low (1e-6), making donation attacks expensive. Attacker must donate millions to significantly affect share price.

**Q: Why are Morpho Blue flash loans free while Aave charges 0.09%?**
A: Design philosophy - Morpho aims to be a minimal primitive. Free flash loans increase capital efficiency and composability. Revenue comes from protocol fees on interest, not flash loan fees.

**Q: What happens to suppliers when bad debt occurs?**
A: Bad debt is socialized. `totalSupplyAssets` decreases, reducing share price for all suppliers proportionally. This eliminates the need for insurance funds or bad debt auctions.

### Calculation Questions

**Q: Alice supplies 1000 USDC when totalSupplyAssets=10000, totalSupplyShares=9000. How many shares?**
```
shares = 1000 × (9000 + 1e6) / (10000 + 1)
       = 1000 × 1,009,000 / 10,001
       = 1,008,899.01 → 1,008,899 shares (rounded down)
```

**Q: Borrow rate is 5% APY. Interest after 1 day on 1,000,000 borrowed?**
```
perSecondRate = 0.05 / 31,536,000 ≈ 1.585 × 10⁻⁹ (WAD: 1.585e9)
elapsed = 86,400 seconds
x = 1.585e9 × 86400 = 1.369e14

firstTerm = 1.369e14
secondTerm = (1.369e14)² / (2 × 1e18) ≈ 9.37e9
thirdTerm ≈ negligible

interest_factor ≈ 1.369e14
interest = 1,000,000 × 1.369e14 / 1e18 ≈ 136.9 USDC
```

**Q: LLTV=80%, collateral=$1000. Max borrow?**
```
maxBorrow = $1000 × 0.80 = $800
```

**Q: 100 ETH collateral at $2000/ETH, 150,000 USDC debt, LLTV=85%. Healthy?**
```
collateralValue = 100 × 2000 = $200,000
maxBorrow = 200,000 × 0.85 = $170,000
debt = $150,000
170,000 ≥ 150,000 ✓ HEALTHY
```

### Code Analysis Questions

**Q: Why does repay use `zeroFloorSub` for totalBorrowAssets?**
A: Rounding can cause repaid assets to exceed totalBorrowAssets by 1 wei. `zeroFloorSub` returns `max(0, x-y)` preventing underflow.

**Q: Why does supplyCollateral NOT call _accrueInterest?**
A: Gas optimization. Collateral doesn't earn interest. Health checks use oracle prices, not interest-affected values.

**Q: In liquidate, why is collateral transferred BEFORE the callback?**
A: Enables flash liquidation. Liquidator receives collateral first, can sell it in callback, then pay loan token. No upfront capital needed.

---

## 14. References & Further Reading

### Official Resources
- [Morpho Blue GitHub Repository](https://github.com/morpho-org/morpho-blue)
- [Morpho Blue Whitepaper (PDF)](https://github.com/morpho-org/morpho-blue/blob/main/morpho-blue-whitepaper.pdf)
- [Morpho Official Blog: Morpho Blue Vision](https://morpho.org/blog/morpho-blue-and-how-it-enables-our-vision-for-defi-lending/)
- [Morpho Documentation](https://docs.morpho.org/)

### Technical Deep Dives
- [MixBytes: Modern DeFi Lending Protocols - Morpho Blue](https://mixbytes.io/blog/modern-defi-lending-protocols-how-its-made-morpho-blue) - Excellent technical breakdown
- [RareSkills: ERC4626 Interface Explained](https://rareskills.io/post/erc4626) - Understanding vault standards
- [OpenZeppelin ERC4626 Documentation](https://docs.openzeppelin.com/contracts/5.x/erc4626) - Inflation attack defense

### Standards
- [EIP-4626: Tokenized Vaults](https://eips.ethereum.org/EIPS/eip-4626) - The vault standard Morpho draws from
- [EIP-712: Typed Structured Data Hashing](https://eips.ethereum.org/EIPS/eip-712) - For signature authorization

### Analytics & Market Data
- [DefiLlama - Morpho](https://defillama.com/protocol/morpho) - TVL and market statistics
- [Dune Analytics](https://dune.com/morpho) - On-chain data dashboards

### Security
- [Morpho Blue Audits](https://github.com/morpho-org/morpho-blue/tree/main/audits) - Security audit reports
- [Certora Formal Verification](https://github.com/morpho-org/morpho-blue/tree/main/certora) - Formal specs

---

## Quick Reference: Key Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `WAD` | 1e18 | Standard precision unit (18 decimals) |
| `MAX_FEE` | 0.25e18 | 25% max protocol fee on interest |
| `ORACLE_PRICE_SCALE` | 1e36 | Oracle price denominator |
| `LIQUIDATION_CURSOR` | 0.3e18 | 30% - affects LIF calculation |
| `MAX_LIQUIDATION_INCENTIVE_FACTOR` | 1.15e18 | 15% max liquidation bonus |
| `VIRTUAL_SHARES` | 1e6 | Anti-inflation attack: virtual share count |
| `VIRTUAL_ASSETS` | 1 | Anti-inflation attack: virtual asset count |

---

## Mental Model Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MORPHO BLUE                                     │
│                                                                              │
│  Market = keccak256(loanToken, collateralToken, oracle, irm, lltv)          │
│                                    │                                         │
│                                    ▼                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                          SUPPLY SIDE                                    │ │
│  │  • User deposits loan tokens → receives internal shares                 │ │
│  │  • Interest accrues → totalSupplyAssets increases (shares unchanged)   │ │
│  │  • Share price = (totalAssets + 1) / (totalShares + 1e6)               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                          Loan tokens flow                                    │
│                                    │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                          BORROW SIDE                                    │ │
│  │  • User posts collateral (tracked as assets, not shares)               │ │
│  │  • Can borrow up to: collateral × price × LLTV                         │ │
│  │  • Borrows loan tokens → receives debt shares                          │ │
│  │  • Interest accrues → totalBorrowAssets increases (debt grows)         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                       If unhealthy (debt > max borrow)                       │
│                                    ▼                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         LIQUIDATION                                     │ │
│  │  • Liquidator repays debt → seizes collateral + LIF bonus              │ │
│  │  • LIF = min(1.15, 1/(1 - 0.3×(1-LLTV)))                               │ │
│  │  • If collateral = 0 but debt remains → bad debt socialized            │ │
│  │  • Bad debt reduces totalSupplyAssets (suppliers share loss)           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Good luck with your exam! Remember: understand the WHY behind each design decision, not just the WHAT. The simplicity of Morpho Blue (650 lines) means every line has a purpose.*
