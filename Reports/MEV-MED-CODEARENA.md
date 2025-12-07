# MagicLpAggregator Vulnerability: Incorrect Pricing Leading to Arbitrage Loss

**Source:** [MagicLpAggregator.sol](https://github.com/code-423n4/2024-03-abracadabra-money/blob/1f4693fdbf33e9ad28132643e2d6f7635834c6c6/src/oracles/aggregators/MagicLpAggregator.sol#L37)

## Overview

This document provides a comprehensive analysis of a critical vulnerability in the `MagicLpAggregator` contract. It includes:
- Vulnerability summary and impact
- Step-by-step mathematical walkthrough
- Exploit derivation and profit formulas
- Additional implementation bugs
- Recommended fixes and best practices
- A safe replacement contract with `mulDiv`

---

## Table of Contents

1. [Summary](#1-summary)
2. [Vulnerability Details](#2-vulnerability-details)
3. [Detailed Arithmetic Walkthrough](#3-detailed-arithmetic-walkthrough)
4. [General Profit Formula](#4-general-profit-formula)
5. [Additional Implementation Bugs](#5-additional-implementation-bugs)
6. [Recommended Fixes](#6-recommended-fixes--best-practices)
7. [Safe Replacement Contract](#7-safe-replacement-contract)

---

## Vulnerable Code

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.0;

import {FixedPointMathLib} from "solady/utils/FixedPointMathLib.sol";
import {IERC20Metadata} from "openzeppelin-contracts/interfaces/IERC20Metadata.sol";
import {IAggregator} from "interfaces/IAggregator.sol";
import {IMagicLP} from "/mimswap/interfaces/IMagicLP.sol";

contract MagicLpAggregator is IAggregator {
    IMagicLP public immutable pair;
    IAggregator public immutable baseOracle;
    IAggregator public immutable quoteOracle;
    uint8 public immutable baseDecimals;
    uint8 public immutable quoteDecimals;

    uint256 public constant WAD = 18;

    /// @param pair_ The MagicLP pair address
    /// @param baseOracle_ The base oracle
    /// @param quoteOracle_ The quote oracle
    constructor(IMagicLP pair_, IAggregator baseOracle_, IAggregator quoteOracle_) {
        pair = pair_;
        baseOracle = baseOracle_;
        quoteOracle = quoteOracle_;
        baseDecimals = IERC20Metadata(pair_._BASE_TOKEN_()).decimals();
        quoteDecimals = IERC20Metadata(pair_._QUOTE_TOKEN_()).decimals();
    }

    function decimals() external pure override returns (uint8) {
        return 18;
    }

    function _getReserves() internal view virtual returns (uint256, uint256) {
        (uint256 baseReserve, uint256 quoteReserve) = pair.getReserves();
    }

    function latestAnswer() public view override returns (int256) {
        uint256 baseAnswerNomalized = uint256(baseOracle.latestAnswer()) * (10 ** (WAD - baseOracle.decimals()));
        uint256 quoteAnswerNormalized = uint256(quoteOracle.latestAnswer()) * (10 ** (WAD - quoteOracle.decimals()));
        uint256 minAnswer = baseAnswerNomalized < quoteAnswerNormalized ? baseAnswerNomalized : quoteAnswerNormalized;

        (uint256 baseReserve, uint256 quoteReserve) = _getReserves();
        baseReserve = baseReserve * (10 ** (WAD - baseDecimals));
        quoteReserve = quoteReserve * (10 ** (WAD - quoteDecimals));
        return int256(minAnswer * (baseReserve + quoteReserve) / pair.totalSupply());
    }

    function latestRoundData() external view returns (uint80, int256, uint256, uint256, uint80) {
        return (0, latestAnswer(), 0, 0, 0);
    }
}
```

---

## 1. Summary

The `MagicLpAggregator` incorrectly values every reserve unit at the **minimum** oracle price between the two underlying tokens. 

### Incorrect Calculation (Current Implementation)

```
perLP = min(priceA, priceB) × (reserveA + reserveB) / totalSupply
```

### Correct Calculation (What It Should Be)

```
perLP = (priceA × reserveA + priceB × reserveB) / totalSupply
```

### Attack Vector

This vulnerability allows a simple arbitrage attack:
1. Buy LP tokens priced by the aggregator (undervalued)
2. Redeem underlying tokens
3. Sell tokens at real market prices
4. Pocket the difference

The attack extracts value from LP holders or any system trusting the aggregator.

---

## 2. Vulnerability Details

- **Type:** MEV / Pricing Oracle Vulnerability
- **Severity:** High
- **Root Cause:** Using `minPrice` across both token reserves rather than valuing each reserve by its **own** oracle price
- **Impact:** 
  - Protocols using this aggregator to price LPs (for trading, lending, liquidation) can be exploited for profit
  - LP holders lose value through arbitrage attacks

---

## 3. Detailed Arithmetic Walkthrough

### Example Pool State

- `totalSupply = 1,000,000` LP tokens
- `reserveA = 1,000,000` units of token A
- `reserveB = 1,000,000` units of token B

### Example Prices

- `priceA = $0.99`
- `priceB = $1.00`

### True Per-LP Value (Correct Calculation)

```
perLP_true = (priceA × reserveA + priceB × reserveB) / totalSupply
           = (0.99 × 1,000,000 + 1.00 × 1,000,000) / 1,000,000
           = 1,990,000 / 1,000,000
           = $1.99
```

### Buggy Aggregator Per-LP Value

```
minPrice = min(0.99, 1.00) = $0.99

perLP_agg = minPrice × (reserveA + reserveB) / totalSupply
          = 0.99 × (1,000,000 + 1,000,000) / 1,000,000
          = 1,980,000 / 1,000,000
          = $1.98
```

### Difference

**$0.01 per LP token** - This creates an arbitrage opportunity.

### Attack Example

An attacker buys `Δ = 100,000` LP tokens at the aggregator price and redeems:

- **Paid:** `Δ × perLP_agg = 100,000 × $1.98 = $198,000`
- **Redeemed underlying:** `100,000` token A + `100,000` token B
- **Sold at true prices:** `100,000 × $0.99 + 100,000 × $1.00 = $199,000`
- **Profit:** `$1,000`

This profit is realized immediately at the expense of LP holders or the platform that allowed the LP to be purchased at the aggregator price.

---

## 4. General Profit Formula

### Variable Definitions

- `S` = totalSupply (total LP tokens)
- `Δ` = LP tokens attacker buys & redeems
- `R_b, R_q` = reserves of base and quote tokens (in units)
- `p_b, p_q` = oracle prices for base and quote tokens
- `p_min = min(p_b, p_q)`

### Paid by Attacker (Aggregator Price)

```
paid = Δ × (p_min × (R_b + R_q)) / S
```

### Redeemed True Value

```
redeemed = Δ × (p_b × R_b + p_q × R_q) / S
```

### Profit Formula

```
profit = (Δ / S) × (p_b × R_b + p_q × R_q - p_min × (R_b + R_q))
       = (Δ / S) × ((p_b - p_min) × R_b + (p_q - p_min) × R_q)
```

### Simplified Profit Formula

Because one of `(p_b - p_min)` or `(p_q - p_min)` equals zero, profit simplifies to the spread times the reserve of the higher-priced token, scaled by `Δ/S`.

**If `p_b = p_min` (base is cheaper):**

```
profit = (Δ / S) × (p_q - p_b) × R_q
```

This formula shows that profit scales linearly with:
- `Δ` (amount of LP tokens purchased)
- Price spread `(p_q - p_b)`
- Reserve size `R_q` on the expensive side

---

## 5. Additional Implementation Bugs

1. **`_getReserves()` does not return values**
   - Missing `return (baseReserve, quoteReserve);` statement
   - Function declares return values but doesn't return them

2. **Potential overflow / incorrect arithmetic ordering**
   - `minAnswer × (baseReserve + quoteReserve)` may overflow `uint256` before division by `totalSupply`
   - Should use safe multiplication/division operations

3. **Unsafe conversion to `int256`**
   - No bounds check before casting `uint256` → `int256`
   - Could revert or misbehave if value exceeds `int256.max`

4. **Normalization typos and naming**
   - Variable name typo: `baseAnswerNomalized` (should be `baseAnswerNormalized`)

5. **No TWAP or smoothing mechanism**
   - Single-block oracle spikiness can enable flash attacks
   - No protection against oracle manipulation

---

## 6. Recommended Fixes & Best Practices

### Core Fixes

1. **Correct per-LP calculation:**
   ```
   perLP = (p_b × R_b + p_q × R_q) / S
   ```

2. **Use safe fixed-point and `mulDiv` operations**
   - Prevents overflow and precision loss
   - Use OpenZeppelin's `Math.mulDiv` or similar

3. **Add bounds checking**
   - Check bounds before casting `uint256` → `int256`
   - Ensure value doesn't exceed `type(int256).max`

4. **Implement TWAP options**
   - Use time-weighted average prices/reserves to avoid flash arbitrage and MEV
   - Smooth out single-block oracle manipulation

5. **Provide conservative bounds**
   - Compute lower/upper bounds per-token rather than applying a single `minPrice` across the pool

6. **Add interaction limits**
   - Implement size/slippage limits where third-party systems can buy LPs priced by this aggregator

---

## 7. Safe Replacement Contract

Below is a production-oriented replacement contract that:

- Computes per-LP value correctly as `(p_b × R_b + p_q × R_q) / totalSupply`
- Uses `Math.mulDiv` for exact multiplication/division guarded against overflow
- Normalizes decimals safely
- Includes guards for `int256` conversion
- Exposes an optional `useTwap` flag for TWAP integration

> **Note:** This is a draft and must be tested and adapted to your project's libraries and style. Imports assume OpenZeppelin (`Math.sol`) is available.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "openzeppelin-contracts/interfaces/IERC20Metadata.sol";
import "openzeppelin-contracts/utils/math/Math.sol"; // for mulDiv
import {IAggregator} from "interfaces/IAggregator.sol";
import {IMagicLP} from "/mimswap/interfaces/IMagicLP.sol";

contract SafeMagicLpAggregator is IAggregator {
    IMagicLP public immutable pair;
    IAggregator public immutable baseOracle;
    IAggregator public immutable quoteOracle;
    uint8 public immutable baseDecimals;
    uint8 public immutable quoteDecimals;

    uint8 public constant OUT_DECIMALS = 18; // WAD

    bool public useTwap; // if true, prefer TWAP data (if implemented/offered)

    constructor(IMagicLP pair_, IAggregator baseOracle_, IAggregator quoteOracle_, bool useTwap_) {
        pair = pair_;
        baseOracle = baseOracle_;
        quoteOracle = quoteOracle_;
        baseDecimals = IERC20Metadata(pair_._BASE_TOKEN_()).decimals();
        quoteDecimals = IERC20Metadata(pair_._QUOTE_TOKEN_()).decimals();
        useTwap = useTwap_;
    }

    function decimals() external pure override returns (uint8) {
        return OUT_DECIMALS;
    }

    function _getReserves() internal view virtual returns (uint256 baseReserve, uint256 quoteReserve) {
        (baseReserve, quoteReserve) = pair.getReserves();
    }

    function latestAnswer() public view override returns (int256) {
        // Fetch and normalize prices to OUT_DECIMALS
        uint256 pB = uint256(baseOracle.latestAnswer()) * (10 ** (OUT_DECIMALS - baseOracle.decimals()));
        uint256 pQ = uint256(quoteOracle.latestAnswer()) * (10 ** (OUT_DECIMALS - quoteOracle.decimals()));

        (uint256 rB, uint256 rQ) = _getReserves();
        // Normalize reserves to OUT_DECIMALS
        uint256 rBW = rB * (10 ** (OUT_DECIMALS - baseDecimals));
        uint256 rQW = rQ * (10 ** (OUT_DECIMALS - quoteDecimals));

        // numerator = pB × rBW + pQ × rQW
        // Use Math.mulDiv to compute each product safely and sum, then divide by totalSupply safely.
        uint256 term1 = Math.mulDiv(pB, rBW, 1);
        uint256 term2 = Math.mulDiv(pQ, rQW, 1);
        uint256 numerator = term1 + term2;

        uint256 supply = pair.totalSupply();
        require(supply > 0, "ZERO_SUPPLY");

        uint256 perLP = Math.mulDiv(numerator, 1, supply); // exact division

        require(perLP <= uint256(type(int256).max), "OVERFLOW_INT");
        return int256(perLP);
    }

    function latestRoundData() external view returns (uint80, int256, uint256, uint256, uint80) {
        return (0, latestAnswer(), 0, 0, 0);
    }
}
```

### Implementation Notes

- `Math.mulDiv` is used for clarity; depending on your environment, you may prefer `FixedPointMathLib`'s `mulWadDown`/`mulWadUp` to keep consistent fixed-point semantics
- This contract doesn't implement TWAP itself; if enabling `useTwap`, you should integrate a reliable TWAP provider for prices and/or reserves

---

## Conclusion

The `MagicLpAggregator` vulnerability demonstrates the critical importance of correct pricing calculations in DeFi protocols. The use of `minPrice` across both reserves creates a systematic undervaluation that can be exploited for profit. The recommended fixes address both the core pricing issue and additional implementation bugs to create a more robust and secure aggregator.
