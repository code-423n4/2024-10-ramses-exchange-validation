# QA for Ramses-Exchange

## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-incorrect-period-tracking-in-reward-claims-allows-double-claiming-of-current-period-rewards) | Incorrect period tracking in reward claims allows double-claiming of current period rewards |
| [QA-02](#qa-02-unintended-fee-protocol-persistence-in-pool-specific-settings) | Unintended fee protocol persistence in pool-specific settings |
| [QA-03](#qa-03-inaccurate-tick-selection-in-oraclesperiodcumulativesinside-function) | Inaccurate tick selection in `Oracle's::periodCumulativesInside` function |
| [QA-04](#qa-04-no-input-validation-in-periodcumulativesinside) | No input validation in `periodCumulativesInside` |
| [QA-05](#qa-05-unchangeable-voter-address-creates-single-point-of-failure) | Unchangeable voter address creates single point of failure |
| [QA-06](#qa-06-data-gaps-in-period-tracking-because-of-non-contiguous-period-advancement) | Data gaps in period tracking because of non-contiguous period advancement |
| [QA-07](#qa-07-nonfungiblepositionmanager-may-cause-losses-if-tokensowed0-overflow) | `NonfungiblePositionManager` may cause losses if `tokensOwed0` overflow |
| [QA-08](#qa-08-no-token-recovery-mechanism-in-ramsesv3poolsol) | No token recovery mechanism in `RamsesV3Pool.sol` |
| [QA-09](#qa-09-limited-traceability-of-pool-fee-protocol-updates) | Limited traceability of pool fee protocol updates |




## [QA-01] Incorrect period tracking in reward claims allows double-claiming of current period rewards

The issue is in the `GaugeV3.sol` contract where the `lastClaimByToken` mapping is incorrectly updated after processing reward claims. This allows users to double-claim rewards for the current period due to improper period tracking.

First take a look at the [`_getAllRewards()` function:](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L493-L532)

```solidity
function _getAllRewards(
    address owner,
    uint256 index,
    int24 tickLower,
    int24 tickUpper,
    address[] memory tokens,
    address receiver
) internal {
    bytes32 _positionHash = positionHash(
        owner,
        index,
        tickLower,
        tickUpper
    );
    uint256 currentPeriod = _blockTimestamp() / WEEK;
    uint256 lastClaim;
    for (uint256 i = 0; i < tokens.length; ++i) {
        lastClaim = Math.max(
            lastClaimByToken[tokens[i]][_positionHash],
            firstPeriod
        );
        // Loop processes rewards including currentPeriod
        for (uint256 period = lastClaim; period <= currentPeriod; ++period) {
            _getReward(
                period,
                tokens[i],
                owner,
                index,
                tickLower,
                tickUpper,
                _positionHash,
                receiver
            );
        }
        // Incorrectly sets lastClaimByToken to currentPeriod - 1
        lastClaimByToken[tokens[i]][_positionHash] = currentPeriod - 1;
    }
}
```

This could be exploited as follows:

Initially, 

```solidity
currentPeriod = 10
lastClaim = 8
```

2. First claim occurs here:
```solidity
- User calls getReward()
- Processes periods 8, 9, and 10
- Sets lastClaimByToken to 9 (currentPeriod - 1)
```

3. Second claim:
```solidity
- User immediately calls getReward() again
- lastClaim is now 9
- Can claim period 10 again
- Sets lastClaimByToken to 9 again
```

This creates a cycle where period 10 can be claimed multiple times because:
1. The loop includes the current period (`period <= currentPeriod`)
2. [`lastClaimByToken` is set](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L530) to one period behind (`currentPeriod - 1`)
3. The next claim starts from this previous period

Additionally this issue is compounded by the period amount tracking in [`cachePeriodEarned()`](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L311-L315):

```solidity
if (period < _blockTimestamp() / WEEK && caching) {
    periodAmountsWritten[period][_positionHash] = true;
    periodNfpSecondsX96[period][_positionHash] = periodSecondsInsideX96;
}
```

This condition prevents proper recording of the current period's amounts and further adds to the inconsistent state

### Recommendations

Consider updating `lastClaimByToken` to the current period instead of `currentPeriod - 1`. Also, consider implementing a cooldown period between claims to prevent rapid consecutive claims.

```diff
-lastClaimByToken[tokens[i]][_positionHash] = currentPeriod - 1;
+lastClaimByToken[tokens[i]][_positionHash] = currentPeriod;
```


## [QA-02] Unintended fee protocol persistence in pool-specific settings

`setPoolFeeProtocol` and `setPoolFeeProtocolBatch` functions allow setting a pool-specific fee protocol, but they don't properly treat the case when the `_feeProtocol` parameter is set to 0.

Take a look at the `setPoolFeeProtocol` function: https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/RamsesV3Factory.sol#L127-L133

```solidity
function setPoolFeeProtocol(address pool, uint8 _feeProtocol) external restricted {
    require(_feeProtocol <= 100, FTL());
    uint8 feeProtocolOld = poolFeeProtocol(pool);
    _poolFeeProtocol[pool] = _feeProtocol;
    emit SetPoolFeeProtocol(pool, feeProtocolOld, _feeProtocol == 0 ? feeProtocol : _feeProtocol);

    IRamsesV3Pool(pool).setFeeProtocol();
}
```

The issue is that when `_feeProtocol` is 0, it's still stored in `_poolFeeProtocol[pool]`. However, the intention seems to be that a value of 0 should reset the pool to use the global `feeProtocol` instead of a pool-specific one.

This becomes apparent when looking at the [`poolFeeProtocol`](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/RamsesV3Factory.sol#L164-L166) function:

```solidity
function poolFeeProtocol(address pool) public view override returns (uint8 __poolFeeProtocol) {
    return (_poolFeeProtocol[pool] == 0 ? feeProtocol : _poolFeeProtocol[pool]);
}
```

This function is designed to return the global `feeProtocol` when `_poolFeeProtocol[pool]` is 0. But `setPoolFeeProtocol` function allows explicitly setting `_poolFeeProtocol[pool]` to 0, which doesn't achieve the intended reset behavior.


### Recommendation

`setPoolFeeProtocol` function should remove the pool-specific fee protocol when `_feeProtocol` is 0, rather than setting it to 0. Could be done by using the `delete` keyword:

```diff
function setPoolFeeProtocol(address pool, uint8 _feeProtocol) external restricted {
    require(_feeProtocol <= 100, FTL());
    uint8 feeProtocolOld = poolFeeProtocol(pool);
+    if (_feeProtocol == 0) {
+        delete _poolFeeProtocol[pool];
+    } else {
        _poolFeeProtocol[pool] = _feeProtocol;
    }
    emit SetPoolFeeProtocol(pool, feeProtocolOld, _feeProtocol == 0 ? feeProtocol : _feeProtocol);

    IRamsesV3Pool(pool).setFeeProtocol();
}
```

Same fix should also apply to the `setPoolFeeProtocolBatch` functions as well






## [QA-03] Inaccurate tick selection in `Oracle's::periodCumulativesInside` function

`periodCumulativesInside` function in the Oracle library, which is essential for calculating liquidity data for specific time intervals within a tick range may cause incorrect data retrieval and usage.

This can cause incorrect calculations of seconds per liquidity data for tick ranges, particularly when querying data for the current, ongoing period. 

Take a look at [`periodCumulativesInside` function](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/libraries/Oracle.sol#L507-L512): 

```solidity
uint256 currentPeriod = $.lastPeriod;

if (currentPeriod > period) {
    lastTick = $.periods[period].lastTick;
} else {
    lastTick = $.slot0.tick;
}
```

The function intends to use the period's last tick for finalized periods and the current tick for the ongoing period. But it fails to handle the case where we're still in the current period correctly.

Consider a scenario where `periodCumulativesInside` is called from another function, such as [`_updatePosition`](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/libraries/Position.sol#L298-L384):

```solidity
function _updatePosition(UpdatePositionParams memory params, uint256 counter) external returns (PositionInfo storage position) {
    uint256 period = params._blockTimestamp / 1 weeks;
   //...SNIP
    uint160 secondsPerLiquidityPeriodX128 = Oracle.periodCumulativesInside(
        uint32(period),
        params.tickLower,
        params.tickUpper,
        params._blockTimestamp
    );
    // ...
}
```

If `currentPeriod` and `period` are equal (e.g., both 3000), indicating we're still in the current period, the function incorrectly uses the current tick (`$.slot0.tick`) instead of the period's last tick. This leads to inaccurate data being retrieved and used for the tick range.

### Proof of Concept
1. Assume `currentPeriod` ($.lastPeriod) = 3000
2. A call is made to `periodCumulativesInside` with `period` = 3000
3. The condition `currentPeriod > period` is false
4. The function uses `$.slot0.tick` instead of `$.periods[period].lastTick`
5. This results in potentially inaccurate data for the current period

### Recommended Mitigation
Modify the condition in the `periodCumulativesInside` function as follows:

```solidity
--- if (currentPeriod > period) {
+++ if (currentPeriod >= period) {
                lastTick = $.periods[period].lastTick;
            } else {
                lastTick = $.slot0.tick;
            }
        }
```









## [QA-04]No input Validation in `periodCumulativesInside`

https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/libraries/Oracle.sol#L470-L486

```solidity
function periodCumulativesInside(
    uint32 period,
    int24 tickLower,
    int24 tickUpper,
    uint32 _blockTimestamp
) external view returns (uint160 secondsPerLiquidityInsideX128) {
    PoolStorage.PoolState storage $ = PoolStorage.getStorage();

    TickInfo storage lower = $._ticks[tickLower];
    TickInfo storage upper = $._ticks[tickUpper];

    // ... SNIP
}
```

The function directly accesses tick information from storage without verifying:
1. If the ticks are within the valid range for the pool
2. If the ticks are initialized
3. If `tickLower` is actually lower than `tickUpper`

### Recommendations

Consider adding checks for tick range validity, tick initialization and correct ordering of lower and upper ticks











## [QA-05] Unchangeable voter address creates single point of failure

The concern in this [`createGauge` function](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/gauge/ClGaugeFactory.sol#L37-L46) is related to the ownership and access control.

```solidity
function createGauge(
    address pool
) external override returns (address gauge) {
    require(msg.sender == voter, "AUTH");
    require(getGauge[pool] == address(0), "GE");
    gauge = address(new GaugeV3(voter, nfpManager, feeCollector, pool));
    getGauge[pool] = gauge;
    emit GaugeCreated(pool, gauge);
}
```

While the function checks if the caller is the `voter` address, there's no way to change the `voter` address after the contract is deployed. As a result,

- If the `voter` address is compromised or lost, there's no way to create new gauges which will effectively lock the functionality.
- There's no way to transfer ownership or update the `voter` address if needed for upgrades.
 
Additionally, the contract emits an [`OwnerChanged` event](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/gauge/ClGaugeFactory.sol#L33) in the constructor:

```solidity
emit OwnerChanged(address(0), msg.sender);
```

This event actually suggests that there should be an owner of the contract, but there's no actual owner functionality implemented. There're no functions to change the owner or transfer ownership.

### Recommendations
Consider implementing an owner role with the ability to update the `voter` address.












## [QA-06] Data gaps in period tracking because of non-contiguous period advancement

https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/RamsesV3Pool.sol#L749-L778

```solidity
function _advancePeriod() public {
    PoolStorage.PoolState storage $ = PoolStorage.getStorage();

    /// @dev if in new week, record lastTick for previous period
    /// @dev also record secondsPerLiquidityCumulativeX128 for the start of the new period
    uint256 _lastPeriod = $.lastPeriod;
    if ((_blockTimestamp() / 1 weeks) != _lastPeriod) {
        Slot0 memory _slot0 = $.slot0;
        uint256 period = _blockTimestamp() / 1 weeks;
        $.lastPeriod = period;

        /// @dev start new period in observations
        uint160 secondsPerLiquidityCumulativeX128 = Oracle.newPeriod(
            $.observations,
            _slot0.observationIndex,
            period
        );

        /// @dev record last tick and secondsPerLiquidityCumulativeX128 for old period
        $.periods[_lastPeriod].lastTick = _slot0.tick;
        $.periods[_lastPeriod].endSecondsPerLiquidityPeriodX128 = secondsPerLiquidityCumulativeX128;

        /// @dev record start tick and secondsPerLiquidityCumulativeX128 for new period
        PeriodInfo memory _newPeriod;

        _newPeriod.previousPeriod = uint32(_lastPeriod);
        _newPeriod.startTick = _slot0.tick;
        $.periods[period] = _newPeriod;
    }
}
```

The potential issue here is that the function unintentionally assumes that the periods are always contiguous and that there are no gaps between them. Albeit, if the `_advancePeriod` function is not called for a while (e.g., more than a week), there could be missing periods in between.

For e.g, let's say the last recorded period is `10`, and the current block timestamp corresponds to period `15`. In this case, the function will update the storage for period `10` and create a new entry for period `15`, but it will not create entries for periods `11, 12, 13, and 14`.

It will not create or record any data for the  intermediate periods that were skipped. This could lead to gaps in the historical data

### Recommendation
Consider modifying the `_advancePeriod` function to handle missing periods and ensure that all periods between the last recorded period and the current period are properly initialized and updated in the storage.











## [QA-07] `NonfungiblePositionManager` may cause losses if `tokensOwed0` overflow

The casting of amounts from uint256 to uint128 and the addition operations in tokensOwed0/tokensOwed1 
can cause a loss to users of the `NonfungiblePositionManager` for tokens that have very high decimals or large amounts.

This appears in multiple functions:

[In decreaseLiquidity():](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/periphery/NonfungiblePositionManager.sol#L296-L324)
```solidity
position.tokensOwed0 +=
    uint128(amount0) +
    uint128(
        FullMath.mulDiv(
            feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
            positionLiquidity,
            FixedPoint128.Q128
        )
    );
```

[In increaseLiquidity():](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/periphery/NonfungiblePositionManager.sol#L237-L258)
```solidity
position.tokensOwed0 += uint128(
    FullMath.mulDiv(
        feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
        position.liquidity,
        FixedPoint128.Q128
    )
);
```

The amount of tokens necessary for the loss is `3.4028237e+38 (2^128 - 1)`. This is equivalent to 1e20 value with 18 decimals
As a result, loss of funds due to overflow when collecting fees or decreasing liquidity

### Recommendation
This bug doesn't require a fix but consider documenting this risk to the end users











## [QA-08] No token recovery mechanism in `RamsesV3Pool.sol`

RamsesV3Pool doesnt have a way to recover tokens that are accidentally sent directly to the contract address. This could lead to permanent loss of tokens for users who mistakenly transfer ERC20 tokens to the pool contract outside of its intended functions.

>This is user mistake


The contract includes various functions for managing the liquidity pool, such as:

```solidity
function swap(
    address recipient,
    bool zeroForOne,
    int256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) external override returns (int256 amount0, int256 amount1) {
    // ... SNIP ...
}

function mint(
    address recipient,
    uint256 index,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount,
    bytes calldata data
) external override lock advancePeriod returns (uint256 amount0, uint256 amount1) {
    // ... SNIP ...
}

function collectProtocol(
    address recipient,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock returns (uint128 amount0, uint128 amount1) {
    // ... SNIP ...
}
```

However, there is no function that allows for the recovery of arbitrary ERC20 tokens sent to the contract address. This means that any tokens accidentally transferred to the contract become permanently locked, as there is no mechanism to retrieve them.

As a result, users may lose tokens permanently if they mistakenly send them directly to the contract address and the contract could accumulate "stuck" tokens over time, which cannot be utilized or recovered either.

### Recommendation

Implement a token recovery function in that allows an authorized party (e.g., contract owner or designated admin) to recover accidentally sent tokens.












## [QA-09] Limited traceability of pool fee protocol updates

The problem here is in how pool-specific fee protocols interact with the global fee protocol. Specifically, in the `poolFeeProtocol` function: 

https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/RamsesV3Factory.sol#L164-L166

```solidity
function poolFeeProtocol(address pool) public view override returns (uint8 __poolFeeProtocol) {
    return (_poolFeeProtocol[pool] == 0 ? feeProtocol : _poolFeeProtocol[pool]);
}
```

The issue arises when the global `feeProtocol` is changed using `setFeeProtocol`. If a pool has its `_poolFeeProtocol` set to 0 (which means it should use the global fee protocol), and then the global `feeProtocol` is changed, the pool's fee protocol will silently change without emitting any events for that specific pool.

This creates a situation where:
1. A pool can be set up with `_poolFeeProtocol[pool] = 0` (using global fee)
2. The global `feeProtocol` is changed from, say, 80 to 50
3. The pool's effective fee protocol changes from 80 to 50 without any event being emitted for that pool
4. There's no way to track this change for individual pools through events

This could lead to unexpected situations and make it difficult to track fee protocol changes at the pool level. While the `SetFeeProtocol` event is emitted for the global change, there's no indication of which pools were affected by this change.

### Recommendation
Consider storing the actual fee protocol value for each pool instead of using 0 as a special case to reference the global value OR emit events for all affected pools when the global fee protocol changes(though this will costs extra gas)
