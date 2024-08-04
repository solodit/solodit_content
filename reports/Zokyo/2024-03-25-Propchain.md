**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Incorrect Loop Condition.

**Severity**: Critical

**Status**: Resolved

**description**

1. PROPCStakingV2.sol: redeem().

The function contains a critical flaw in its loop construct that results in the function not accounting for the very first staking position (stake ID 0) when calculating the rewards for redemption. As a result, users cannot claim rewards for their initial stake. The issue arises from the loop's condition that prevents the iteration from reaching the index 0:
```solidity
for(uint256 stakeId = _stakes.length; stakeId > 0; stakeId--) {
Stake storage_stake = _stakes[stakeId];
}
```
Since the condition checks 'stakeId > 0, and stake IDs are zero-indexed, the first element (stakeId = 0) is never processed, resulting in rewards associated with the first stake never being claimed.

2. PROPCStakingV2.sol: _userPoolMetrics()
   
The function contains a critical flaw in its for-loop condition. In case when `stakeld` becomes zero, the index in `array_stakes` will be less than zero, so there will be underflow in the loop. As a result, users cannot leave the pool or make emergency withdrawals.

**Recommendation:**

Adjust the loop to iterate over all stakes by changing the loop condition to allow stakeId to reach 0.

**Post audit.**

Loop iterates over the stake with index 0 now.

### Assignment missed.

**Severity**: Critical

**Status**: Resolved

**description**

1. PROPCStakingV2.sol: rewardsForPool().

The `rewardsForPool` getter function is intended to return the total accumulated rewards for a user's stakes in a given pool. However, due to an issue with how the add operation is used within the function, it will always return zero. The result of the add operation is not being stored back into the accumulating variable, thus the total calculated rewards are not preserved.

3. PROPCStakingV2.sol: leavePool()
   
The `leavePool` function has loop 'for' calculating all staking IDs to withdraw all stakes from the pool. On line 352, where `amount should update the total amount to withdraw is not updating, just adding 'amount from the stake id and not assigning the value of amount variable. This causes the number of withdrawals to always be 0, so that the user will not be able to withdraw their funds.

**Recommendation:**

Modify the loop in the `rewardsForPool` function to properly accumulate rewards by assigning the result of the add operation to the rewards variable. Regarding `leavePool` function, modify assigning the result of the add operation to the amount variable in loop.

**Post audit.**

Result of addition is stored in the variable now.

### Unpredicted reward amount necessary to pay out in the pool.


**Severity**: High

**Status**: Resolved

**Description**

Usually the reward distribution system has a fixed amount of reward token which is distributed linearly among participants based on the weight of their stake.
However, the current system pays out a fixed APY per token (which only changes based on time periods set by admin). Since the pool has no maximum cap on total deposits, it might be difficult to predict how many reward tokens the protocol has to allocate to pay out all the users. In a worst-case scenario, the maximum which can be deposited in a single pool is restricted by
circulating supply of the PROP token. Thus, the protocol potentially has to allocate an amount of rewards to cover such scenario. Also, the issue is marked as high since the withdrawal functions will revert if the contract can't pay all the rewards to the user, preventing users from withdrawing funds until admin replenishes the rewards for pool.

**Recommendation:**

Add a mechanism to predict the necessary amount of reward tokens, for example, a maximum cap for the total deposit amount per pool in order to have the ability to calculate the necessary amount of rewards more precisely. Add an emergency withdrawal function which will allow users to withdraw without caring about rewards.

**Post audit.**

The Propchain team has added the parameter `maxTotalStake` in every pool which checks that users can't stake more than the limit. Additionally, function emergency `Withdraw()` was implemented which will pay out only available rewards (based on allowance of the `rewardsWallet`), thus not preventing the withdrawal due to the absence of rewards.

### Incorrect loop condition.

**Severity**: High

**Status**: Resolved

**Description**

PROPCStakingV2.sol: redeem(), redeemAll().
The function redeem uses the loop to go through the array of all the user's stakes. Since the array is walked from the last to the first stake, the loop sooner or later comes to a point when `stakeId` will be equal to 0, which leads to underflow, because at the end of the loop, 'stakeId with value 0 will subtract one. This leads to the fact that user will not be able to output his rewards as a whole.

**Recommendation:**

The best way to fix the problem is to go through an array from the first to the last stake of the user
```solidity
uint256 stakeId=0; stakeId < stakeId.length;stakeId++
```
On the other hand, fixing the issue possible by adding an if statement to check if `stakeId` is equal to zero, then break from the loop in a case when the direction is going from the last to the first stake.

### Claiming time.

**Severity**: High

**Status**: Resolved

**Description**

PROPCStakingV2.sol: _pending RewardsForStake().

The function for counting rewards uses logic to check how long a user has been in the pool since their last output in the loop. A comment in the code describes that if the user has not spent enough time in the pool then the loop is skipped and a new iteration is started, but the logic does the opposite, which results in the user's rewards being if they have spent enough time in the pool. If the user has made a deposit and `claimTimeLimit` is 1 day, then after this day, his rewards will be skipped in the loop, which equals 0. This problem also leads to incorrect operation of such functions as redeem where a similar check comes from the reverse logic.

**Recommendation:**

Confirm that the logic in the `_pendingRewardsForStake` function is true to the conditions of the contract business logic and, in this case, change the check in the redeem function in the loop.

**Post audit.**

The logic of the function was fixed thus comment and logic in the loop correspond to each other.

## Medium Risk

### Potential Performance Issue with Unbounded Loops.

**Severity**: Medium

**Status**: Resolved

**Description**

PROPCStakingV2.sol: leavePool(), redeem().

Several functions iterate over arrays without explicit bounds, specifically when looping through user stakes. As a user's number of stakes increases, functions like `leavePool` and `redeem` require more gas to execute, potentially reaching block gas limit constraints. This situation could lead to denial of service if the functions become unexecutable due to exceeding gas limits.

**Recommendation:**

Implement a mechanism to limit the number of active stakes per user.

**Post audit.**

A limit of 10 stakes per user was introduced.

## Low Risk

### SafeMath usage can be omitted.

**Severity**: Low

**Status**: Resolved

**Description**


Contracts utilize Solidity 0.8.x, which has built-in support for overflow and underflow warnings. Thus, the SafeMath library can be omitted for gas savings and code simplification.

**Recommendation:**

Remove SafeMath library.

**Post audit.**

SafeMath was removed.

## Informational

### Inaccurate version pragma.

**Severity**: Informational

**Status**: Resolved

**Description**


`pragma solidity ^0.8.0;` is used in contract. The contract should be deployed with the same compiler version and options with which it has been most tested. Locking the pragma version helps ensure that the contract is not accidentally deployed using a different version. In addition, older versions may contain bugs and vulnerabilities, and be less optimized in terms of gas. It is recommended to use the latest version of Solidity and specify the exact pragma.

**Recommendation:**

Specify the latest version of Solidity in the pragma statement.

**Post audit.**

The Propchain team has specified a more recent version of Solidity, 0.8.19.

### Violations of Solidity Style Guide Conventions - magic numbers.

**Severity**: Informational

**Status**: Resolved

**Description**

Magic numbers usage. Hard-coded constants like:
a) _pendingRewardsForStake() → line 263: 365 days, 10000;
b) penalty() → line 278: 100; line 280: 10_000_000; line 282: 10000;
are not self-explanatory or defined as named constants, making it difficult to understand their meaning or how they were derived.

**Recommendation:**

Replace magic numbers with named constants that provide context and make the code more maintainable and understandable.

### Violations of Solidity Style Guide Conventions - file name and contract name mismatch.

**Severity**: Informational

**Status**: Resolved

**Description**


File PlatformStakingV2 → contract PROPCStakingV2.sol.

Solidity style guidelines advise that the contract name should match its filename for clarity and to prevent confusion during contract interaction or deployment.

**Recommendation:**

Name contract file and contract itself with the same name.

### Unused Function.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: _min(uint256 , uint256).
Within the PROPCStakingV2 contract, the internal function `_min` is defined to return the minimum of two given uint256 values. Upon review of the contract, it appears that this function is not invoked anywhere within the code. Unused functions increase the contract size unnecessarily, which can lead to extra gas costs during deployment.

**Recommendation:**

If the function is not required for current or future anticipated contract logic, consider removing it.

**Post audit.**

Unused function was removed.

### Non-updatable State Variable Not Declared immutable.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: propcToken.
The `propcToken` state variable is set during contract deployment and is not intended to be updated thereafter. However, the variable is currently not marked as immutable. Marking state variables as immutable can save on gas costs because these variables are written directly into the contract runtime bytecode, eliminating the need for storage access after contract deployment.

**Recommendation:**

If there are no use cases where the `propcToken` would need to change post-deployment, consider declaring propcToken as immutable to optimize gas usage during contract interactions.

**Post audit.**

Variables was marked as immutable.

### Lack of events.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: updateRewardsWallet(), setPool(), addVIPAddress(), addVIPAddresses(), removeVIPAddress(), removeVIPAddresses(), deleteStake(). In order to keep track of historical changes of storage variables, it is recommended to emit events on every change in the functions that modify the storage.

**Recommendation:**

Consider emitting events in these functions.

### Lack of Validation.

**Severity**: Informational

**Status**: Resolved

**Description**

```solidity
PROPCStakingV2.sol: constructor()→_propc, _rewardsWallet;
setPool() → _penaltyWallet;
addVIPAddress()→_vipAddress;
addVIPAddresses() → _vipAddresses;
removeVIPAddress()→_vipAddress;
removeVIPAddresses()→_vipAddresses;
stake() → amount;
unstake() → amount.
```
All parameters described above do not have validation. Addresses are not checked for zero addresses, and amounts are not checked for being zero amounts.

**Recommendation:**

Consider adding the necessary validation.

### Lack of events.

**Severity**: Informational

**Status**: Resolved

**Description**

Functions With Public Visibility That Could Be Marked External. (Resolved) PROPCStakingV2.sol: setPool(), pendingRewardsForStake(), getUserInfo(), getStakesInfo(),
getPoolInfo(). Several functions are currently declared as public. However, these functions do not appear to be called internally by the contract and are typical interaction points for users or front-end applications. Changing these to external can result in gas savings because external functions
can read arguments directly from calldata, which is cheaper compared to copying arguments to memory as with public functions.

**Recommendation:**

Assess whether these functions are strictly used for external calls and if so, consider updating their visibility modifier from public to external to achieve gas savings.

### Similar Naming for Variables.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: setPool()→_pid and pid.
The setPool function utilizes parameters and local variables with similar names (pid and pid). These naming choices may lead to confusion and difficulty in differentiating between them, especially for new developers or during code maintenance.

**Recommendation:**

Employ more descriptive names for function parameters and local variables.

**Post audit.**

pid was changed to `_idToChange` variable thus the issue was resolved.

### Redundant require statements.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: stake()
In function, stake 2 checks respond to checking the proper allowance and balance of the user. As later propc token is transferring by method transferFrom there is no need to make these checks as transferFrom method has built-in checks to prevent problems with the allowance and balance of the user.

**Recommendation:**

Remove 2 checks of allowance and balance of the user.

**Post audit.**

Balance and allowance validation was removed from the function

### Misleading Comment.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: _pending Rewards ForStake, line 243.
Within the `_pendingRewardsForStake()`, there is a misleading comment associated with the `claimTimeLimit` that could cause confusion about its usage. The **comment reads**: not long enough in the pool to retrieve rewards. However, the logic is intended to limit the time window in which rewards can be claimed after they are earned. The comment incorrectly suggests that it is related to a minimum staking duration that is required before rewards can be claimed.

**Recommendation:**

Update the comment to accurately reflect the functionality.

**Post audit.**

The validation logic has been modified in response to the comment.

### Double check.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: unstake()
In function, there is a check for preventing cause when `stakeId` index is out of array. However, this function calls function `deleteStake`, which calls only ones in contract, and this function has the same check. So, there is no need to check the same arguments twice.

**Recommendation:**

Remove double check from `deleteStake` function.

### Unused variable.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: stake()
In function, there is a check the amount of tokens that the user wants to stake is higher than the amount of pool minimum stake amount. However, there is no function or argument in function setPool where the admin can set minStakeAmount for the pool, that's why this check is unreachable.

**Recommendation:**

Add argument to function setPool to set minimum stake amount for the pool or remove check in function stake and variable in the struct 'PoolInfo`.

**Post audit.**

Parameter minStakeAmount is set in function setPool() now.

### Decreasing penalty calculation.

**Severity**: Informational

**Status**: Resolved

**Description**

PROPCStakingV2.sol: _penalty()
In the case of setting up a pool with the function of decreasing the number of penalties and withdrawing tokens after some time, the number of penalties is counted the other way round.
If penalty time limit is set to 10 hours with 9% penalty. If the user deposits 150 tokens, his maximum penalty will be 13.5 tokens, so each hour is equal to 1.35 tokens. If the user wants to withdraw tokens after an hour, then supposedly the amount of commission should be 1.35 tokens less, that is 12.15 tokens.
But at withdrawal the amount of commission is equal to 1.35 tokens. The calculations lead to the result that the later the user withdraws tokens, the higher the commission is.

**Recommendation:**

Confirm that the commission's logic is correct and follows the business logic.

**Post audit.**

The logic of calculation in the case when the pool has a decreasing fee was fixed during business logic. The penalty amount decreases according to how much time a user is in the pool.
