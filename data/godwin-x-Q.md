## Title: Unused Import of `IERC20Minimal` in `RamsesV3Pool` contract


## Links to Affected Code:
https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/core/RamsesV3Pool.sol#L21

## Description:
The contract includes an import statement for `IERC20Minimal` but does not use it within the contract. Unused imports increase the compiled bytecode size, potentially leading to slightly higher deployment costs. They can also introduce unnecessary dependencies, impacting code clarity and making the contract appear more complex than it actually is.

## Suggested Fix
Remove the unused import statement:
```solidity
// Remove this line if IERC20Minimal is not used in the contract
 import {IERC20Minimal} from './interfaces/IERC20Minimal.sol';

```

## Potential Impact
While this is not a security risk, it has minor negative effects:

 1. Slightly increases contract deployment costs due to larger bytecode size.
 2. Reduces code readability, as unused imports may confuse developers about dependencies.


**Recommendation**: Remove any unused imports to keep the contract clean, readable, and optimized.