# L1. Unnecessary Payable Modifier on `NonfungiblePositionManager.decreaseLiquidity` Function

## Links to affected code
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/periphery/NonfungiblePositionManager.sol#L268

## Vulnerability Details

### Summary
The `decreaseLiquidity` function in the `NonfungiblePositionManager` contract is unnecessarily marked as `payable`. This function does not require or utilize any Ether, as it calls `pool.burn`, which itself does not involve Ether transfers. This issue results in an unnecessary modifier, creating potential confusion and an increased attack surface.

### Impact
The presence of the `payable` modifier may lead users or developers to mistakenly send Ether to the `decreaseLiquidity` function, under the incorrect assumption that it is required. This could lead to unintentional Ether accumulation in the contract and introduces ambiguity in how the function should be used. Additionally, an unnecessary `payable` modifier slightly increases the attack surface of the contract.

### Proof of Concept
The following code snippet highlights the `decreaseLiquidity` function's current structure:

```solidity
function decreaseLiquidity(
    DecreaseLiquidityParams calldata params
)
    external
@>  payable
    override
    isAuthorizedForToken(params.tokenId)
    checkDeadline(params.deadline)
    returns (uint256 amount0, uint256 amount1)
{
    // Function logic
}
```

The `payable` modifier is present, but neither this function nor the underlying `pool.burn` method involves Ether.

### Recommended Mitigation
Remove the `payable` modifier from the `decreaseLiquidity` function to align its behavior with intended usage and eliminate any potential for unintentional Ether transfers.


# L2. Missing Input Validation for Desired Amounts in `NonfungiblePositionManager.mint` Function

## Links to affected code
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/periphery/NonfungiblePositionManager.sol#L139

## Vulnerability Details

### Summary
The `mint` function in the `NonfungiblePositionManager` contract lacks a validation check to ensure that either `params.amount0Desired` or `params.amount1Desired` is greater than zero. This allows users to unintentionally attempt to mint positions with zero desired liquidity, leading to inefficient execution and unnecessary gas costs.

### Impact
Without validation on the input amounts, users may incur additional gas costs when unintentionally attempting to mint positions with zero tokens. This results in unnecessary function execution and transaction reverts that could have been avoided with a fast-fail validation. Adding this check would improve user experience and optimize gas costs by preventing attempts to mint positions with no liquidity.

### Proof of Concept
In the current implementation, a user could call `mint` with zero values for both `params.amount0Desired` and `params.amount1Desired`. Since the function does not validate these values at the start, it proceeds with minting logic, leading to wasted gas in cases where no liquidity can be added.

```solidity
function mint(
    MintParams calldata params
)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 tokenId, uint128 liquidity, uint256 amount0, uint256 amount1)
{
    // Function logic proceeds without validation on desired amounts
}
```

### Recommended Mitigation
Add a check at the beginning of the `mint` function to ensure that at least one of `params.amount0Desired` or `params.amount1Desired` is greater than zero:

```solidity
require(params.amount0Desired > 0 || params.amount1Desired > 0, "Invalid amounts");
```

This validation will allow the function to fail quickly if both amounts are zero, saving gas costs and improving efficiency.


# L3. Missing Input Validation for Desired Amounts in `NonfungiblePositionManager.increaseLiquidity` Function

## Links to affected code
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/periphery/NonfungiblePositionManager.sol#L205

## Vulnerability Details

### Summary
The `increaseLiquidity` function in the `NonfungiblePositionManager` contract lacks a validation check to ensure that either `params.amount0Desired` or `params.amount1Desired` is greater than zero. This oversight allows users to unintentionally attempt to increase liquidity with zero token amounts, leading to inefficient execution and unnecessary gas costs.

### Impact
Without input validation on the desired token amounts, users may incur additional gas costs when unintentionally attempting to increase liquidity with zero amounts. This results in wasted gas and unnecessary function execution. Adding this check would improve user experience and save gas by preventing calls that do not meaningfully increase liquidity.

### Proof of Concept
In the current implementation, a user could call `increaseLiquidity` with zero values for both `params.amount0Desired` and `params.amount1Desired`. Since the function does not validate these values at the start, it proceeds with liquidity increase logic, which could lead to wasted gas if no liquidity is actually added.

```solidity
function increaseLiquidity(
    IncreaseLiquidityParams calldata params
)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint128 liquidity, uint256 amount0, uint256 amount1)
{
    // Function logic proceeds without validation on desired amounts
}
```

### Recommended Mitigation
Add a check at the beginning of the `increaseLiquidity` function to ensure that at least one of `params.amount0Desired` or `params.amount1Desired` is greater than zero:

```solidity
require(params.amount0Desired > 0 || params.amount1Desired > 0, "Invalid amounts");
```

This validation will allow the function to fail early if both token amounts are zero, saving gas costs and improving function efficiency.


# L4. Inconsistent Timestamp Usage in `GaugeV3.left` Function

## Links to affected code
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L115-L116

## Vulnerability Details

### Summary
The `left` function in the `GaugeV3` contract calls `_blockTimestamp()` twice, relying on `block.timestamp`. Because `block.timestamp` can vary slightly between calls within the same transaction, there is a possibility of minor inconsistencies in the `remainingTime` calculation. This may lead to small inaccuracies in reward distribution calculations.

### Impact
The use of multiple calls to `_blockTimestamp()` can result in slight discrepancies in the calculated remaining rewards due to differences in the two timestamps. While this discrepancy is small, it can impact the precision of the reward calculations. Additionally, calling `_blockTimestamp()` twice introduces a minor gas overhead, which could be avoided by using a single call.

### Proof of Concept
In the current implementation, `left` calls `_blockTimestamp()` twice, potentially leading to different values each time due to `block.timestamp` changes:

```solidity
function left(address token) external view override returns (uint256) {
@>  uint256 period = _blockTimestamp() / WEEK;
@>  uint256 remainingTime = ((period + 1) * WEEK) - _blockTimestamp();
    return (tokenTotalSupplyByPeriod[period][token] * remainingTime) / WEEK;
}
```

### Recommended Mitigation
To ensure consistency, call `_blockTimestamp()` only once and store its value in a local variable:

```solidity
function left(address token) external view override returns (uint256) {
    uint256 currentTimestamp = _blockTimestamp();
    uint256 period = currentTimestamp / WEEK;
    uint256 remainingTime = ((period + 1) * WEEK) - currentTimestamp;
    return (tokenTotalSupplyByPeriod[period][token] * remainingTime) / WEEK;
}
```

This change ensures that only one timestamp is used throughout the function, improving accuracy and gas efficiency.
