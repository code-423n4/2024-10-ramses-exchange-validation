## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01] | Incorrect Historical Liquidity Usage in Period Calculations |
| [QA-02] | Incorrect Function Visibility Modifier in Position Library |
| [QA-03] | Unrestricted Pool Creation Vulnerability in RamsesV3Factory |
| [QA-04] | Inconsistent Function Signatures for `setPoolFeeProtocolBatch` |
| [QA-05] | Unclear Error Messages Reduce Contract Usability and Maintainability |
| [QA-06] | GaugeV3 Contract Lacks Protection Against Rebasing Token Accounting Issues |

## [QA-01] Incorrect Historical Liquidity Usage in Period Calculations

### Proof of Concept
The `Oracle.sol` contract incorrectly uses current liquidity values when calculating historical period metrics, leading to inaccurate results for non-finalized periods.

https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/libraries/Oracle.sol#L470-L471
```
function periodCumulativesInside(
    uint32 period,
    int24 tickLower,
    int24 tickUpper,
    uint32 _blockTimestamp
) external view returns (uint160 secondsPerLiquidityInsideX128) {
    // ...
    if (currentPeriod <= period) {
        cache.time = _blockTimestamp;
        // BUG: Uses current liquidity for historical calculations
        (, cache.secondsPerLiquidityCumulativeX128) = observeSingle(
            $.observations,
            cache.time,
            0,
            _slot0.tick,
            _slot0.observationIndex,
            $.liquidity,  // Current liquidity used instead of historical
            _slot0.observationCardinality
        );
    }
}
```
Impact:

- Historical period calculations use current liquidity values instead of period-specific liquidity
- Results in incorrect secondsPerLiquidity calculations when liquidity changes significantly
Affects:
- Liquidity mining rewards calculations
- Historical protocol metrics
- Any dependent protocols using this data
Example scenario:
```
// Period 1 (Historical):
Initial liquidity: 1000 ETH
Time elapsed: 1 week

// Current state:
Current liquidity: 100 ETH (90% decrease)

// Calculation using current code:
secondsPerLiquidity = (1 week) / 100 ETH  // INCORRECT
// Should be:
secondsPerLiquidity = (1 week) / 1000 ETH // CORRECT

// Result: 10x inflation of secondsPerLiquidity value
```

### Recommended Mitigation Steps
Add period-specific liquidity tracking

## [QA-02] Incorrect Function Visibility Modifier in Position Library

### Proof of Concept
The `_updatePosition` function in the Position library is incorrectly marked as `external`, which is inconsistent with library usage patterns and Solidity conventions. Libraries are meant to be used internally by other contracts, and functions prefixed with underscore conventionally indicate internal visibility.

Direct link to the code: https://github.com/Ramsesv3Project/2024-10-ramses-exchange/blob/main/contracts/CL/core/libraries/Position.sol#L298
```
/// @dev Gets and updates a position with the given liquidity delta
/// @param params the position details and the change to the position's liquidity to effect
function _updatePosition(UpdatePositionParams memory params) external returns (PositionInfo storage position) {
    // ... implementation
}
```
Impact:

- Inconsistent with Solidity library patterns
- Violates naming conventions (underscore prefix suggests internal use)
- Could cause confusion for developers maintaining or integrating with the code
- May lead to unexpected behavior when trying to use the library
Additional Context:

- Libraries in Solidity are designed for internal code reuse
- The external visibility modifier on library functions is unusual and potentially problematic
- The function handles critical position management logic that should be restricted to internal usagage
### Recommended Mitigation Steps
Change the function visibility from `external` to `internal`:
## [QA-03] Unrestricted Pool Creation Vulnerability in RamsesV3Factory
### Proof of Concept
The `RamsesV3Factory` contract allows unrestricted creation of liquidity pools, which could lead to potential griefing attacks and state bloat. This vulnerability exists in the `createPool` function, which is externally accessible without any access controls or creation fees.

Direct link to the relevant code: https://github.com/Ramsesv3Project/2024-10-ramses-exchange/blob/main/contracts/CL/core/RamsesV3Factory.sol#L71-L103
```
function createPool(
    address tokenA,
    address tokenB,
    int24 tickSpacing,
    uint160 sqrtPriceX96
) external override returns (address pool) {
    require(tokenA != tokenB, IT());
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
    require(token0 != address(0), A0());
    uint24 fee = tickSpacingInitialFee[tickSpacing];
    require(fee != 0, F0());
    require(getPool[token0][token1][tickSpacing] == address(0), PE());

    // ... (pool creation logic)
}
```
Potential impacts:

- Malicious actors could create a large number of unnecessary pools, leading to state bloat.
- Excessive gas consumption and network congestion due to spam pool creation.
- Difficulty for users in identifying legitimate pools among a flood of irrelevant ones.
- Potential denial of service (DoS) if the number of pools becomes extremely large.

### Recommended Mitigation Steps
- Implement access control for pool creation
- Implement a whitelist for allowed token pairs
- Introduce a pool creation fee to discourage spam

## [QA-04] Inconsistent Function Signatures for `setPoolFeeProtocolBatch`

### Proof of Concept
The `RamsesV3Factory` contract contains two overloaded versions of the `setPoolFeeProtocolBatch` function with different parameter lists. This can lead to confusion and potential misuse.

Direct links to the referenced code in GitHub:

[setPoolFeeProtocolBatch with single feeProtocol](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/RamsesV3Factory.sol#L137-L138)
[setPoolFeeProtocolBatch with array of feeProtocols](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/RamsesV3Factory.sol#L149-L150)

The two function signatures are:
```
function setPoolFeeProtocolBatch(address[] calldata pools, uint8 _feeProtocol) external restricted { ... }

function setPoolFeeProtocolBatch(address[] calldata pools, uint8[] calldata _feeProtocols) external restricted { ... }
```
While Solidity supports function overloading, having two functions with the same name but different behaviors can be confusing for users and auditors. The first function sets the same fee protocol for all pools, while the second allows setting different fee protocols for each pool.

This design could lead to accidental misuse, where a caller might intend to set custom fee protocols for each pool but accidentally calls the version that sets a uniform fee protocol for all pools, or vice versa.

### Recommended Mitigation Steps
- Rename the functions to be more descriptive of their specific behaviors
- Alternatively, use a common prefix with distinct suffixes

## [QA-05] Unclear Error Messages Reduce Contract Usability and Maintainability

### Proof of Concept
The RamsesV3Factory contract uses cryptic, abbreviated error messages that significantly reduce code readability, complicate debugging, and make integration more difficult. This affects multiple critical functions in the contract.
```
require(tokenA != tokenB, IT());
require(token0 != address(0), A0());
require(fee != 0, F0());
require(getPool[token0][token1][tickSpacing] == address(0), PE());
```
Impact:

- Debugging becomes more difficult as error messages don't clearly indicate the failure reason
- Integration with other protocols is complicated due to unclear error handling
- User experience suffers when transactions fail with cryptic error codes
- Increased maintenance overhead as developers need to maintain separate error code documentation
Additional examples of unclear error messages:
```
// In enableTickSpacing function
require(tickSpacing > 0 && tickSpacing < 16384, 'TS');
require(tickSpacingInitialFee[tickSpacing] == 0, 'TS!0');

// In setPoolFeeProtocolBatch function
require(pools.length == _feeProtocols.length, 'AL');
```
### Recommended Mitigation Steps
Replace current error messages with custom errors that clearly describe the issue

## [QA-06] GaugeV3 Contract Lacks Protection Against Rebasing Token Accounting Issues
### Proof of Concept
The GaugeV3 contract's reward accounting system assumes static token balances between operations, making it vulnerable to accounting errors when handling rebasing tokens (e.g., AMPL, aTokens, stETH) where token balances can change without transfers.

Direct link to vulnerable code: https://github.com/Ramsesv3Project/2024-10-ramses-exchange/blob/main/contracts/CL/gauge/GaugeV3.sol

Key vulnerable areas:

Reward Amount Recording:
```
function notifyRewardAmount(address token, uint256 amount) external override pushFees lock {
    // ...
    uint256 balanceBefore = IERC20(token).balanceOf(address(this));
    IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
    uint256 balanceAfter = IERC20(token).balanceOf(address(this));

    amount = balanceAfter - balanceBefore;
    tokenTotalSupplyByPeriod[period][token] += amount; // Vulnerable to rebases
    emit NotifyReward(msg.sender, token, amount, period);
}
```
Reward Calculation:
```
function cachePeriodEarned(...) public override returns (uint256 amount) {
    // ...
    amount = FullMath.mulDiv(
        tokenTotalSupplyByPeriod[period][token], // Uses potentially outdated amounts
        periodSecondsInsideX96,
        WEEK << 96
    );
}
```
Reward Distribution:
```
function _getReward(...) internal {
    uint256 _reward = cachePeriodEarned(...);
    if (_reward > 0) {
        periodClaimedAmount[period][_positionHash][token] += _reward;
        IERC20(token).safeTransfer(receiver, _reward); // Actual transferred amount may differ
        emit ClaimRewards(period, _positionHash, receiver, token, _reward);
    }
}
```
Impact:

- Incorrect reward calculations due to outdated balance records
- Users might receive more/less rewards than intended
- Total distributed rewards may not match actual token balances
- Potential for reward distribution inequalities

### Recommended Mitigation Steps
- Add Rebasing Token Detection and Tracking
- Modify Reward Notification for Rebasing Tokens