# QA Report

## L-01 Prevent Governance Errors in `enableTickSpacing`

### `enableTickSpacing` allows setting the initial fee for a tickSpacing to zero, ruining the tickSpacing

[RamsesV3Factory.sol#L23](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L107)

We should `require` that `initialFee` is greater than zero. It is not obvious to governance that zero-fee pools are not allowed by [RamsesV3Factory.sol#L81](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L81) in `createPool`.

Given that the default fee cannot be changed once set, the governance error would permanently ruin a `tickSpacing`.

## L-02 Event Poisoning

### `FeeCollector.collectProtocolFees` can be made to emit false `FeeCollected` events

[FeeCollector.sol#L72](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/FeeCollector.sol#L72)

The `pool` variable is not verified as existing, so a malicious one can be supplied by anyone. This fake pool can be built with real `token0` and `token1` contracts and reporting any amount of them being provided as fees, while providing nothing in reality.

As a result, the `FeeCollector` will emit `FeeCollected` events referring to bogus pools for any amount.

## L-03 Inefficient Code in GaugeV3.removeRewards

### The loop to maintain reward order is not necessary

[RamsesV3Pool.sol#L427](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L427)

The `removeRewards` function loops through the `rewards` array, writing again each entry after the index.

```
for (uint256 i = idx; i < rewards.length - 1; ++i) {
    rewards[i] = rewards[i + 1];
}
rewards.pop();
```

Since the order is not important, we can replace the removed reward by the last entry, and then `pop` the array.

## L-04 Mixed Precision in Parameters Should be avoided

### Protocol Split is an FP2, while Swap Fee is an FP6

[RamsesV3Factory.sol#L23](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L23)

`feeProtocol` should be in a [0, 1_000_000] range.

Having different precision in configuration parameters can easily lead to governance errors and should generally be avoided.

In this case, making `feeProtocol` have the appropriate range would imply making it a `uint24`, which would mean `unlocked` [wouldn’t fit anymore](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/libraries/PoolStorage.sol#L17) in `Slot0` with traditional struct packing. A possible way of solving this would be to store `unlocked` in the upper bits of a `uint24` `feeProtocol`, since those would always be free. Easier solutions might exist.

## L-05 Use Arrays of Structs

### `setPoolFeeProtocolBatch` should not take two arrays

[RamsesV3Factory.sol#L149](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L149)

Parameters that include multiple arrays, which must be the same length, are confusing for users and error prone.

The `setPoolFeeProtocolBatch(address[], uint8[])` should take an array of `PoolFeeProtocol` structs as a parameter.

## L-06 Code Could Be Simplified in Position

### `positionHash` could be replaced by the index

[Position.sol#L28](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/libraries/Position.sol#L28)

An `index` has been introduced to positions to allow the same `owner` to have different positions with the same tick ranges.

Given that the `index` is [unique](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/periphery/NonfungiblePositionManager.sol#L141) we don’t need to generate a hash out of the owner, index, tickLower and tickUpper, we can just use the `index`. The `index` is also always required to be known by the user, so there is not downside to UX.

## L-07 Code Could Be Simplified in Oracle

### `Oracle.newPeriod` could be a one-liner

[Oracle.sol#L338](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/libraries/Oracle.sol#L338)

The `newPeriod` function creates a new `Observation` for the first second of a period. This could be achieved as follows:

```
function newPeriod(
    Observation[65535] storage self,
    uint16 index,
    uint256 period
) external returns (uint160 secondsPerLiquidityCumulativeX128) {
    self[index] = transform(uint32(period) * 1 weeks - 1, $.slot0.tick, $.liquidity);
    return self[index].secondsPerLiquidityCumulativeX128;
}
```

## L-08 Duplicated Code in RamsesV3Pool

### `_advancePeriod` is reimplemented inside `swap`.

[RamsesV3Pool.sol#L427](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L427)

Duplicated code in smart contracts increases the risk of inconsistencies during updates, where changing one instance but missing others can create dangerous discrepancies in contract behavior and security.

With regards to `RamsesV3Pool`, we are also constrained in how much bytecode we can fit in the contract, so removing duplicated code will allow for some library functions to be declared `internal`, saving execution gas.

The code from `_advancePeriod` is reimplemented inside `swap` to avoid loading `$` again. Since `$` is a `storage` pointer, we could have an internal `_advancePeriod` function that takes it as a parameter, and both an `external` `advancePeriod` and `swap` would pass `$` to `_advancePeriod`.

## NC-01 Duplicated Code in RamsesV3Factory

### `setPoolFeeProtocol` is implemented three times with minor variations.

[RamsesV3Factory.sol#L127](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L127)

Duplicated code in smart contracts increases the risk of inconsistencies during updates, where changing one instance but missing others can create dangerous discrepancies in contract behavior and security.

The `setPoolFeeProtocol`, `setPoolFeeProtocolBatch(address[], uint8)` and `setPoolFeeProtocolBatch(address[], uint8[])` all duplicate the same code to set the protocol fee for a pool:

```
    require(_feeProtocol <= 100, FTL());
    uint8 feeProtocolOld = poolFeeProtocol(pool);
    _poolFeeProtocol[pool] = _feeProtocol;
    emit SetPoolFeeProtocol(pool, feeProtocolOld, _feeProtocol == 0 ? feeProtocol : _feeProtocol);

    IRamsesV3Pool(pool).setFeeProtocol();

```

This code should be extracted into an internal function, which is then called by the external functions that provide UX features.

## NC-02 Duplicated Code For Current Period

### `_blockTimestamp / WEEK` should be replaced by `_currentPeriod`

[GaugeV3.sol#L87](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L87)

The `_blockTimestamp / WEEK` snippet is used 10 times in GaugeV3, it should be replaced by an appropriately named function.

## NC-03 Math Could Be Simplified

### Adding Liquidity to Delta can use a single sum

[RamsesV3Pool.sol#L251](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L251)

The function below comes from Uniswap v3, but it could still be simpler. Simpler code helps finding bugs.

```
uint128 liquidityBefore = $.liquidity; 
...
$.liquidity = params.liquidityDelta < 0
    ? liquidityBefore - uint128(-params.liquidityDelta)
    : liquidityBefore + uint128(params.liquidityDelta);
```

This code could be rewritten as:

```
int256 liquidityBefore = int256(uint256($.liquidity))
...
$.liquidity = (liquidityBefore + params.liquidityDelta).toUint128();
```

## NC-04 Useless parameters in getReward and getPeriodReward

### The `owner` parameter could be removed

[GaugeV3.sol#L394](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L394)
[GaugeV3.sol#L489](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L489)

Requiring the user to enter their own address as a parameter is bad UX. Both `getReward` and `getPeriodReward` should shed the `owner` parameter and do the underlying call with `msg.sender` instead.

## NC-05 Missing receiver in getReward

### `getReward(uint256, address[])` is missing a `receiver` parameter.

[GaugeV3.sol#L454](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L454)

All `getReward` functions allow to collect rewards to an arbitrary `receiver`, except this one. For consistency and future development reasons, it would be convenient to add it here as well.

## NC-06 Use Leading Underscore for Internal Functions

### `_advancePeriod` is not an `internal` function

[RamsesV3Pool.sol#L749](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L749).

`_advancePeriod` should be renamed to `advancePeriod`

Not following code conventions on function visibility can lead to developers misunderstanding the visibility of functions, leading to errors.

## NC-07 Use camelCase for State Variables

### `RamsesV3PoolDeployer.RamsesV3Factory` should be `RamsesV3PoolDeployer.ramsesV3Factory`

[RamsesV3PoolDeployer.sol#L10](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3PoolDeployer.sol#L10)

Not following code conventions on variable  can lead to developers misunderstanding the meaning of variables, leading to errors.

## NC-08 Missing visibility keyword

### Missing visibility keyword

[RamsesV3Factory.sol#L20](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L20).

`_poolFeeProtocol` should be explicitly set to `internal`.

Smart contract state variables should have explicit visibility modifiers (public, private, internal) to prevent unintended access patterns and make the code's security boundaries clear to both developers and auditors.

## NC-09 Unused Named Returns

[RamsesV3Factory.sol#L164](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L164), [RamsesV3Pool.sol#L911](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L911)

Unused named returns use space in the variable stack, and can lead to errors.