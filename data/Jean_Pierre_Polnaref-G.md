---

**Title:** Optimization: Use `!= 0` Instead of `> 0` for Gas Efficiency

## Summary
The contract uses the `> 0` comparison for `uint256` variables, which is less gas-efficient compared to using `!= 0`. Optimizing these comparisons can slightly reduce gas costs, which can have a cumulative impact over multiple executions.

## Vulnerability Details
In the following lines of code, the contract checks if `amount0` and `amount1` are greater than 0 using `> 0`, which can be replaced by `!= 0` to save gas:
```solidity
if (amount0 > 0) balance0Before = balance0();
if (amount1 > 0) balance1Before = balance1();
if (amount0 > 0 && balance0Before + amount0 > balance0()) revert M0();
if (amount1 > 0 && balance1Before + amount1 > balance1()) revert M1();
```

## Impact
Although the impact on gas savings is minor, optimizing the use of comparisons in contracts that are frequently called can help reduce overall gas consumption.

## Tools Used
- Manual analysis
- https://vscode.dev/github/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/core/RamsesV3Pool.sol#L291-L296

## Recommendations
Replace `> 0` with `!= 0` for `uint256` comparisons to reduce gas usage:
```solidity
if (amount0 != 0) balance0Before = balance0();
if (amount1 != 0) balance1Before = balance1();
if (amount0 != 0 && balance0Before + amount0 > balance0()) revert M0();
if (amount1 != 0 && balance1Before + amount1 > balance1()) revert M1();
```

---