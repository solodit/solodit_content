**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Medium Risk

### [M-01] The `revoke` mechanics are not compatible with tokens that implement a block list feature

**Impact:**
High, as important functionality in the protocol won't work

**Likelihood:**
Low, as a special type of ERC20 token has to be used as well as the attacker's address has to be in a block list

**Description**

Some tokens, for example `USDC` and `USDT` implement an admin controlled address block list. All transfers to a blocked address will revert. Since the `revoke` functionality forcefully transfers the claimable vested tokens to an address with a `vestingSchedule`, all calls to `revoke` will revert if such an address has claimable balance and is in the token's block list.

**Recommendations**

Use the [Pull over Push](https://fravoll.github.io/solidity-patterns/pull_over_push.html) pattern to send tokens out of the contract in a `revoke` scenario.

### [M-02] Insufficient input validation in function `createVestingSchedule`

**Impact:**
High, as it can lead to users never vesting their tokens

**Likelihood:**
Low, as it requires a malicious/compromised admin or an error on his side

**Description**

The input arguments of the `createVestingSchedule` function are not sufficiently validated. Here are some problematic scenarios:

1. `_start` can be a timestamp that has already passed or is too far away in the future
2. `_cliff` can be too big, users won't be able to claim
3. 1 is a valid value for `duration`, the `!= 0` check is insufficient
4. If `_slicePeriodSeconds` is too big then the math in `_computeReleasableAmount` will have rounding errors

**Recommendations**

Add sensible lower and upper bounds for all arguments of the `createVestingSchedule` method.

### [M-03] Contract can receive ETH but has no withdraw function for it

**Impact:**
High, as value can be stuck forever

**Likelihood:**
Low, as it should be an error that someone sends ETH to the contract

**Description**

The `TokenVesting` contract has `receive` and `fallback` functions that are `payable`. If someone sends a transaction with `msg.value != 0` then the ETH will be stuck in the contract forever without a way for anyone to withdraw it.

**Recommendations**

Remove the `receive` and `fallback` functions since the ETH balance is not used in the contract anyway.

### [M-04] Users won't be able to claim vested tokens when contract is paused

**Impact:**
High, as owner has the power to make it so that users can't claim any vested tokens

**Likelihood:**
Low, as it requires a malicious or a compromised owner

**Description**

The owner can currently execute the following attack:

1. Call `setPaused` with `paused == true`, so pause the contract
2. Now all user calls to `releaseAvailableTokensForHolder` will fail, since it has the `whenNotPaused` modifier
3. He can not unpause the contract forever or even renounce ownership

This is a common centralization problem which means the contract owner can "rug" users.

**Recommendations**

Remove the `whenNotPaused` modifier from `releaseAvailableTokensForHolder`, so users can claim vested tokens even if admin pauses the contract.

## Low Risk

### [L-01] Limit the max size of the `vestingSchedulesIds` array and `holdersVestingScheduleCount`

If too many vesting schedules are added for a user it is possible that the `getVestingSchedulesIds` method will take too much gas and won't be executable (if it gets over the block gas limit, for example). Also in `releaseAvailableTokensForHolder` there is a `for` loop that loops `vestingScheduleCount` number of times, which can also be problematic, as it can lead to a DoS state with the function. Limit the max size of both, for example up to 500 vesting schedules created from the contract.

### [L-02] The `onlyIfVestingScheduleNotRevoked` modifier will not revert even if the given `vestingScheduleId` is non-existent

The modifier will pass successfully when the `vestingScheduleId` passed is of a non-existent vesting schedule, because the default `Status` of a vesting schedule is `INITIALIZED` anyway. Validate that the `vestingSchedules` exists, by checking that `vestingSchedules[vestingScheduleId].duration != 0`.
