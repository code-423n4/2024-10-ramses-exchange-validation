## QA Report

### QA-01
At the pool initialization, the first observation is [set to time 0](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L159C67-L159C93), which makes the `secondsPerLiquidityCumulativeX128` equal to the current timestamp (which is a big number) after the first `_advancePeriod`. Consider starting the first index at the time of initialization which.

### QA-02
In the [`removeRewards` function inside gauge](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L583-L585), you can just replace the token with the last token and then use `pop()` to remove the last index which saves gas.

### QA-03
The [`newPeriod`](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/libraries/Oracle.sol#L338-L360) function in the oracle is in charge of starting the new period, however, it uses the array of observations, and overwrites the last index in the observation array.
This means that the last entry of wether adding liquidity or swapping would be overwritten. This will cause any future `observe` calls where they are asking between [last_write_timestamp, end_of_period_timestamp) to be inaccurate in the future.