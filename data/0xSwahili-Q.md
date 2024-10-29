L-01 Incorrect use of block.timestamp when initializing a pool at Observe.initialize

The function's Natspecs at https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/libraries/Oracle.sol#L45 state that when initializing the oracle array by writing the first slot, the time should be set to **block.timestamp**:

```solidity
/// @notice Initialize the oracle array by writing the first slot. Called once for the lifecycle of the observations array
/// @param self The stored oracle array
/// @param time The time of the oracle initialization, via block.timestamp truncated to uint32
/// @return cardinality The number of populated elements in the oracle array
/// @return cardinalityNext The new length of the oracle array, independent of population
```
 However, this is not currently implemented as seen at [https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L159](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L159):

```solidity
(uint16 cardinality, uint16 cardinalityNext) = Oracle.initialize($.observations, 0);
```
Impact: Failure to adhere to code specifications 
For a POC, add this test to test/uniswapV3CoreTests/Oracle.spec.ts:

```solidity
it("sets first slot timestamp to zero", async () => {
            await oracle.initialize({ liquidity: 1, tick: 1, time: 0 });
            checkObservationEquals(await oracle.observations(0), {
                initialized: true,
                blockTimestamp: 0n,
                tickCumulative: 0n,
                secondsPerLiquidityCumulativeX128: 0,
            });
        });
```
Recommended fix: Modify https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L159 to:
```diff
-  (uint16 cardinality, uint16 cardinalityNext) = Oracle.initialize($.observations, 0);
+  (uint16 cardinality, uint16 cardinalityNext) = Oracle.initialize($.observations, _blockTimestamp()); 
```
 