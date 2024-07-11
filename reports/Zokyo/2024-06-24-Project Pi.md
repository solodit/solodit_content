**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Incorrect Check For Early Stakers

**Severity** - High

**Status** - Resolved

**Description**

For the first 99 stakers the minimum staking amount is 8e6 ether tokens (compared to 16e6 ether tokens for non-early stakers) , the code which verifies if the staker is an early staker is →
```solidity
function canUseReducedStakingAmount(address stakerAddress) public view returns (bool eligible, uint256 requiredStakingAmount) {
    EarlyStaking earlyStaking = EarlyStaking(payable(getContractAddress("EarlyStaking")));
    int256 stakerIndex = earlyStaking.getIndexOf(stakerAddress);
    bool isEarlyStaker = stakerIndex >= 0; // True if stakerIndex is not -1, meaning the staker exists


    // Assuming the first early staker gets a special treatment and should use the reduced amount
    if (isEarlyStaker) {
        if (stakerIndex == 99) {
            // First 100 early stakers, eligible for reduced amount
            return (true, 8_000_000 ether);
        }
```
It checks if the stakerIndex == 99 , which is incorrect , this line would only be applicable for the 100th staker while it should be applicable to the first 100 stakers.

**Recommendation**:

Change the check to:
```solidity
If (stakerIndex <= 99) return (true , 8_000_000 ether);
```

### Incorrect Check To Verify startTime

**Severity** - High

**Status** - Resolved

**Description**

The start time for staking is assigned in the function `recordStakingStart()` , to verify if the `startTime` is a valid timestamp the following check is performed →
```solidity
if (startTime > block.timestamp) {
        revert("InvalidStartTime");
    }
```
But this would be incorrect in a general sense since startTime can be higher than the current timestamp but it should not be less than the current timestamp which would be problematic. If the staking begins in the next block it should be fine since the end timestamp would be calculated accordingly.

**Recommendation**

Change the check to
```solidity
if (startTime < block.timestamp) {
        revert("InvalidStartTime");
    }
```

### Incorrect check in the `createMinipool()` function

**Severity**: High

**Status**: Resolved

**Description**

MinipoolManager.sol L244
For the early stakers in the `createMinipool()` function, the function only accepts the combined contribution equal to the DAO’s minimum requirement and `plsAssignmentRequest` equal to the DAO’s max PLS assignment.

**Recommendation**: 
Change the if condition to like this.
```solidity
if (msg.value + plsAssignmentRequest < dao.getMinipoolMinPLSStakingAmt() || plsAssignmentRequest > dao.getMinipoolMaxPLSAssignment())
```
## Low Risk

### Lack of Checks-effects-interactions pattern 

**Severity**: Low

**Status**: Resolved

**Description**

The Checks-Effects-Interactions (CEI) pattern is not followed in some parts of the smart contract. The CEI pattern is a well-known best practice in smart contract development that helps to prevent reentrancy attacks.

While the `nonReentrant` modifier is used in the contract, which effectively prevents reentrancy, it is still highly recommended to adhere to the CEI pattern in functions like `withdrawPartialRewards()`. Following the CEI pattern can prevent potential issues that might arise from future changes.

**Recommendation**: 

Refactor the contract to implement the Checks-Effects-Interactions pattern.

### Missing Zero Address Checks in Constructor

**Severity**: Low

**Status**: Resolved

**Description**
The constructor of the contract does not perform checks to ensure that the provided addresses are not zero addresses. It is important to validate that the addresses passed to the constructor are valid and not zero.

**Recommendation**: 

Add checks in the constructor to ensure that the provided addresses are not zero addresses. This can be achieved using the require statement.

### Missing Event Emissions

**Severity**: Low

**Status**: Resolved

**Description**

The functions `depositFromRewards()` and `receiveWithdrawalPLS()` do not emit events upon successful execution. Emitting events in smart contracts is crucial for tracking state changes and ensuring transparency. Events provide a reliable way to notify off-chain applications of changes and facilitate easier debugging and auditing.

**Recommendation**: 

Consider emitting events in mentioned functions.

## Informational

### Unnecessary Check

**Severity** - Informational

**Status** - Resolved

**Description**

The check `“require(msg.sender == owner, "Only owner can withdraw");”` on L357 is not needed since the `onlyOwner()` check at L356 is sufficient and would revert if msg.sender is not the owner.

**Recommendation**:

L357 can be removed


### Public Functions Could Be Marked External

**Severity**: Informational

**Status**: Resolved

**Description**

The `distributeRewards()` and `depositFromRewards` functions of the `MiniPoolManager.sol` contract is marked as public but it could be marked as external instead. While both public and external functions can be called from outside the contract, marking a function as external is generally more gas-efficient when it is not intended to be called internally.

**Recommendation**: 

Consider changing the visibility of the claim function from public to external to optimize gas usage.


### Unused Variables

**Severity**: Informational

**Status**: Resolved

**Description**

The smart contract contains variables `minStakingDuration` and `locked` that are declared but not used anywhere in the code. Unused variables can lead to unnecessary gas costs and code bloat. They may also indicate incomplete functionality or leftover code from previous versions, which can be confusing and potentially error-prone.

**Recommendation**: 

Remove the unused variables from the contract to optimize gas usage and improve code clarity.


### Unnecessary converting
	
**Severity**: Informational

**Status**: Resolved

**Description**

In the `depositFromRewards()` function, `lastRewardsCycleEnd` local variable is set with a value converted `rewardsCycleEnd` to `uint32`. But `rewardsCycleEnd` is already a `uint32` type so no need to convert.

**Recommendation**: 

Assign the `lastRewardsCycleEnd` with `rewardsCycleEnd` directly without converting.

### Some variables could be made immutable

**Severity**: Informational

**Status**: Resolved

**Description**

The `nodeIDGenerator` and `validatorRegistration` variables are only set in the constructor.

**Recommendation**: 

Make the variables immutable.

### Unnecessary if condition

**Severity**: Informational

**Status**: Resolved

**Description**

In the `createMinipool()` function, if `eligibleForReducedStaking` is true, it checks if `msg.value` is greater than `8_000_000` ether. (L240)
But if `eligibleForReducedStaking` is true, then `requiredStakingAmount` will be `8_000_000` and `msg.value` is already checked with the `requiredStakingAmount` before.

**Recommendation**: 

Remove the if condition.


