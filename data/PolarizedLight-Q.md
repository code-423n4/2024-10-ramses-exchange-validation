## Ramses Exchanged - Team Polarized Light QA Report

## Low 1 - Lack of __gap Variable in Upgradeable Contract

### Overview: 

The upgradeable contract FeeCollector is missing a gap variable, which is crucial for future-proofing contract upgrades.

### Description: 

Upgradeable contracts should include a gap variable to reserve storage slots for future use. This practice ensures that new state variables can be added in future upgrades without disrupting the existing storage layout or causing potential storage collisions with child contracts.

### CodeLocation: 

https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/gauge/FeeCollector.sol#L18-L18

### Impact: 

Without a __gap variable, future upgrades to the contract may be limited or risky, potentially leading to storage layout conflicts or the need for complex migration strategies.

### Recommended mitigations:

1. Add a __gap variable at the end of the contract:
```solidity
uint256[50] private __gap;
```
2. Adjust the number of reserved slots (50 in this example) based on anticipated future needs.

The absence of a __gap variable in an upgradeable contract reduces its flexibility for future enhancements and increases the risk of storage-related issues during upgrades.

## Low 2 - Event Emission Order in` _getReward` Function

### Overview:

The `getReward` function in the `GaugeV3` contract emits events after external interactions, violating the check-effects-interaction pattern.

### Description: 

The current implementation performs an external interaction (safeTransfer) before emitting the ClaimRewards event. This order of operations could lead to events being emitted out of order if the transfer fails, inconsistent state representation in event logs, and potential vulnerabilities to reentrancy attacks.

### CodeLocation: 

https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/gauge/GaugeV3.sol#L534-L558

### Impact: 

The current order of operations could result in inaccurate event logs, potentially misleading off-chain systems or users about the state of reward claims.

### Recommended mitigations:
1. Reorder the operations to follow the check-effects-interaction pattern:
   - Perform all checks
   - Update the contract's state
   - Emit events
   - Perform external interactions

2. Implement the following code structure:
```solidity
if (_reward > 0) {
    periodClaimedAmount[period][_positionHash][token] += _reward;
    emit ClaimRewards(period, _positionHash, receiver, token, _reward);
    IERC20(token).safeTransfer(receiver, _reward);
}
```

Reordering the event emission before the external interaction ensures a more accurate representation of the contract's state changes and adheres to Solidity best practices.

## Low 3 - Missing Events in Critical Functions

### Overview:

Multiple setter and privileged functions in the contract lack event emissions, reducing transparency and complicating off-chain tracking.

### Description:

Functions such as `setFeeProtocol()`, `setFee(uint24 fee)`, and `setIsLive(bool status)` modify important contract states without emitting corresponding events. This omission reduces transparency, complicates auditing, and limits functionality for decentralized applications interacting with the contract.

### CodeLocation:
- setFeeProtocol(): https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/core/RamsesV3Pool.sol#L728-L728
- setFee(uint24 _fee): https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/core/RamsesV3Pool.sol#L744-L744
- setIsLive(bool status): https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/gauge/FeeCollector.sol#L67-L67

### Impact: 

The lack of events for these critical functions reduces transparency and makes it difficult for external systems to track important state changes in the contract.

### Recommended mitigations:

1. Implement appropriate event emissions in each of the affected functions:
```solidity
event FeeProtocolUpdated(/* relevant parameters */);
event FeeUpdated(uint24 newFee);
event TokenLiveStatusChanged(bool isLive);

function setFeeProtocol() external override lock {
    ProtocolActions.setFeeProtocol(factory);
    emit FeeProtocolUpdated(/* relevant parameters */);
}

function setFee(uint24 _fee) external override lock {
    ProtocolActions.setFee(_fee, factory);
    emit FeeUpdated(_fee);
}

function setIsLive(bool status) external onlyTreasury {
    isTokenLive = status;
    emit TokenLiveStatusChanged(status);
}
```

Adding event emissions to these critical functions will significantly improve the contract's transparency and make it easier for external systems to track important state changes.

## Low 4 - Read-Only Reentrancy Considerations in swap Function

### Overview: 
The swap function in `RamsesV3Pool.sol` contains external calls that could potentially expose read-only reentrancy patterns.

### Description:
The swap function makes external calls to Oracle.observeSingle() (read-only) and IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback() between state changes. While the Oracle call is read-only and the callback is protected by balance checks, this pattern could potentially be used to obtain outdated state readings in specific scenarios.

### CodeLocation:
https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/core/RamsesV3Pool.sol#L420-L562

### Impact:
The risk is limited since:
1. Oracle.observeSingle() is read-only
2. The callback is protected by balance checks
3. Any state manipulation would fail pool balance validations

However, there could be edge cases where reading outdated state through reentrancy might affect external contracts relying on these readings.

Recommended mitigations:
1. Consider adding nonReentrant modifier if integrating contracts need guaranteed fresh state readings
2. Consider moving Oracle.observeSingle() call earlier in the function if possible

## Low 5 -  Consideration for Network-Specific Token Behaviors in Reward Distribution

### Overview: 

The GaugeV3 contract uses a generic token transfer mechanism that may not account for potential network-specific token behaviors.

### Description: 

The contract uses `IERC20(token).safeTransferFrom()` for reward token transfers. This approach doesn't account for potential future network-specific token behaviors, especially if additional reward tokens are introduced. The main issue token being considered in this scenario is WETH9 on Blast Chain.

### CodeLocation: 

Functions 
`notifyRewardAmount` - https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/gauge/GaugeV3.sol#L147-L156
`notifyRewardAmountNextPeriod` - https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/gauge/GaugeV3.sol#L165-L172
`notifyRewardAmountForPeriod` - https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/contracts/CL/gauge/GaugeV3.sol#L182-L190

### Impact: 

Future introduction of reward tokens with unique implementations or network-specific behaviors could lead to inefficiencies or failures in reward distribution.

### Recommended mitigations:
1. Implement a general approval mechanism for reward tokens.
2. Add checks in reward notification functions for tokens requiring special handling.
3. Maintain a list of tokens that require special handling.

Implementing these suggestions would enhance the contract's ability to handle various token implementations across different networks, future-proofing it against potential issues with new reward tokens.

## Non Critical 1 - Missing Event Emission for Non-Immutable State Variable Assignment in Constructor

### Overview

The constructor of the contract sets non-immutable state variables without emitting corresponding events, potentially leading to reduced transparency and auditability.

### Description

It is considered a best practice to emit events when state variables are modified, including during contract initialization in the constructor. This practice enhances transparency and allows off-chain services to track important state changes from the moment of contract deployment.

In the given constructor, two non-immutable state variables (`_tokenDescriptor` and `voter`) are assigned values without emitting events. This omission may make it more difficult for external observers to track the initial state of these important variables.

### Code Location
```solidity
constructor(
    address _deployer,
    address _WETH9,
    address _tokenDescriptor_,
    address _voter
) ERC721('Ramses V3 Positions NFT', 'RAM-V3-NFP') PeripheryImmutableState(_deployer, _WETH9) {
    _tokenDescriptor = _tokenDescriptor_;
    voter = _voter;
}
```

### Impact

While it doesn't directly introduce vulnerabilities, it reduces the contract's transparency and may complicate off-chain monitoring and auditing processes. External systems relying on events to track the contract's state may miss these initial assignments, potentially leading to inconsistencies or difficulties in reconstructing the contract's full state history.

### Recommended Mitigations

1. Emit events for each non-immutable state variable assignment in the constructor. For example:

```solidity
constructor(
    address _deployer,
    address _WETH9,
    address _tokenDescriptor_,
    address _voter
) ERC721('Ramses V3 Positions NFT', 'RAM-V3-NFP') PeripheryImmutableState(_deployer, _WETH9) {
    _tokenDescriptor = _tokenDescriptor_;
    emit TokenDescriptorSet(_tokenDescriptor_);
    
    voter = _voter;
    emit VoterSet(_voter);
}
```

2. Ensure that appropriate events are defined in the contract, such as:

```solidity
event TokenDescriptorSet(address indexed tokenDescriptor);
event VoterSet(address indexed voter);
```

The constructor sets non-immutable state variables without emitting events, reducing transparency and potentially complicating off-chain monitoring of the contract's initial state.

