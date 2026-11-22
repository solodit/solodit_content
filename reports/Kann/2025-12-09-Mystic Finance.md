**Auditor**

[Kann Audits](https://kannaudits.com/audits/mystic-finance-liquid-staking-december-2025)

# Findings

## Medium Risk

### [M-01] Temporary DoS of User Withdrawals Due to Partial Plume Withdrawals Creating Artificial Shortfall

**Severity**

Medium

**Description**

When multiple users interact with the unstake and withdraw logic at different times, a user can be temporarily DoS’d from withdrawing their own funds because of how `plumeStaking. withdraw()` and the local deficit check are combined.

Consider this flow:

User A calls unstake with 50 ETH. This amount gets unstaked from Plume once either `withdrawalQueueThreshold` or `nextBatchUnstakeTimePerValidator` is reached.

User B calls unstake with a larger amount, for example 350 ETH. It takes at least four validators to satisfy this (assuming each has around 100 ETH capacity). Eventually, the full 350 ETH for User B is also unstaked on Plume. At this point, Plume holds around 400 ETH of unstaked funds belonging to both User A and User B.

User A then calls withdraw. Inside _withdraw, the contract executes:

```solidity
if (totalWithdrawable > 0) {
    uint256 balanceBefore = address(this).balance;
    plumeStaking.withdraw();
    uint256 balanceAfter = address(this).balance;
    withdrawn = balanceAfter - balanceBefore;
}
```

This `plumeStaking.withdraw()` call pulls all 400 ETH of cooled funds from Plume into the contract’s balance. Plume now shows zero withdrawable ETH, but the contract’s internal buffers reflect that ETH.

Later, other users (call them User Group M) also unstake their funds and get cooling/parked. Once their cooldowns finish, their amounts become withdrawable on Plume too.

Now User B calls withdraw. The code sees `totalWithdrawable > 0` again, so it calls `plumeStaking. withdraw()` a second time. This time, only the newly cooled funds from User Group M are withdrawn from Plume and assigned to withdrawn. This withdrawn value is less than the 350 ETH that User B expects.

The function then hits the deficit logic:

```solidity
if (withdrawn > 0 && withdrawn < amount) {
    require(amount - withdrawn < fee, "Insufficient funds to cover deficit");
}
```

Here, amount is User B’s requested 350 ETH, and withdrawn is the much smaller amount that just came from User Group M. The condition amount - withdrawn < fee will almost always be false because the fee is tiny compared to the missing difference. As a result, the withdrawal reverts.

At this point, User B is effectively DOS’ed from withdrawing. Every time they try, they will again pull only the small cooled balance from Plume, fail the same `require(amount - withdrawn < fee)` check, and revert. They remain stuck until either another user with a withdrawal request calls withdraw first and drains the Plume balance in a way that avoids the partial-withdraw branch, or enough new unstake events accumulate so that the next `plumeStaking.withdraw()` call returns at least 350 ETH in one shot.

So even though User B has correctly unstaked and the system as a whole has enough ETH across buffers `(currentWithheldETH, totalInstantUnstaked)` and future withdrawals, the specific interaction between `plumeStaking.withdraw()` and the `require(amount - withdrawn < fee)` deficit check can temporarily lock that user out of claiming their funds.

Additionally, User B would normally be allowed to withdraw using `currentWithheldETH` alone if the totalWithdrawable from plume was 0. However, because withdrawn is non-zero, the contract is forced into the deficit branch instead of using local liquidity because `withdrawn‘‘‘ <`amount‘. This guarantees a revert as soon as enters check withdrawn < amount, making the DOS unavoidable until conditions change.

**Team Response**

Fixed.

## Low Risk

### [L-01] Inefficient Max-Percentage Check Prevents Using Valid Partial Capacity

**Severity**

Low

**Description**

In the auto-distribution path of `_depositEther`, the contract decides whether a validator can be used by calling `getNextValidator(remainingAmount, validatorId)` with the full

```solidity
remainingAmount:

while (remainingAmount > 0 && index < numVals) {
    uint256 depositSize = remainingAmount;
    _validatorId = uint16(validators[index].validatorId);
    (uint256 validatorId, uint256 capacity) = getNextValidator(remainingAmount,
        _validatorId);

    if (capacity > 0 && capacity >= minStakeAmount) {
        if (capacity < depositSize) {
            depositSize = capacity;
        }

        if (depositSize < minStakeAmount) {
            currentWithheldETH += remainingAmount;
            return depositedAmount;
        }

        plumeStaking.stake{value: depositSize}(uint16(validatorId));
        remainingAmount -= depositSize;
        depositedAmount += depositSize;
        emit DepositSent(uint16(validatorId));
    }
    index++;
}
```

Inside `getNextValidator`, the validator’s new percentage is computed using `depositAmount` equal to that full `remainingAmount`:

```solidity
function getNextValidator(uint256 depositAmount, uint16 validatorId)
    internal
    view
    returns (uint256 validatorId_, uint256 capacity_)
{
    (bool active, , uint256 stakedAmount, ) = plumeStaking.getValidatorStats(
        validatorId);
    uint256 totalStaked = plumeStaking.totalAmountStaked();
    if (!active) return (validatorId, 0);

    (, capacity_) = _getValidatorInfo(validatorId);

    uint256 percentage =
        ((stakedAmount + depositAmount) * RATIO_PRECISION) /
        (totalStaked + depositAmount);

    if (maxValidatorPercentage[validatorId] > 0 &&
        percentage > maxValidatorPercentage[validatorId])
    {
        return (validatorId, 0); // validator is skipped entirely
    }

    if (capacity_ > 0) {
        return (validatorId, capacity_);
    }
    return (validatorId, 0);
}
```

The critical line is:

```solidity
if (maxValidatorPercentage[validatorId] > 0 && percentage > maxValidatorPercentage[
    validatorId]) {
    return (validatorId, 0);
}
```

If sending the entire `remainingAmount` would push a validator above its `maxValidatorPercentage`, `getNextValidator` returns a capacity of 0, and `_depositEther` skips that validator completely. However, `_depositEther` later caps `depositSize` to capacity only if capacity > 0, so this partial capping never happens for validators that are rejected at the percentage check.

This means that if a validator still has some allowed headroom under both capacity and max-percentage, but not enough to take the full `remainingAmount`, the contract refuses to use it at all. It does not attempt a partial deposit that would keep the validator within limits.

For example, a validator with max capacity 10 ETH and current stake 8 ETH still has 2 ETH of safe room. If `remainingAmount` is 5 ETH, the percentage is calculated as if 5 ETH were going to this validator. If that hypothetical 5 ETH would violate the max percentage, the validator is excluded, even though putting just 2 ETH there would be perfectly valid. As a result, those 2 ETH are never used for this validator; the loop just moves to the next one with the full 5 ETH still in `remainingAmount`.

Validators that still have valid headroom (for example, 1–2 ETH) can be skipped entirely if the full `remainingAmount` would break the percentage limit.

This leads to under-utilization of individual validator capacities and reduces how evenly stake is spread, since the allocator refuses to use small remaining slots on validators that are close to their limit.

In practice, more ETH may end up routed to fewer validators.

**Team Response**

Acknowledged.

### [L-02] Restake() May Re-Stake Funds Already Reserved for User Withdrawals

**Severity**

Low

**Description**

The `restake()` function can reuse funds that users have already unstaked and are waiting to withdraw. When users call `unstake()`, their ETH enters cooling/parked state under the contract, but `restake()` does not check whether those funds are already committed to pending withdrawal requests. It simply calls:

```solidity
plumeStaking.restake(validatorId, amount);
```

and consumes whatever cooling balance exists.

This allows the rebalancer to restake ETH that users are expecting to withdraw after their cooldown period. If this happens, users may reach the end of their cooldown and find that their withdrawal can-not be fulfilled because their cooled funds were re-staked before they could withdraw them. This can lead to temporary withdrawal delays or failed withdrawals until the protocol unstake cycle produces new cooled funds again.

**Team Response**

Acknowledged.

## Informational

### [I-01] Wrong Amount Check in unstakeGov Uses Total Validator Stake Instead of This Contract’s Stake

**Severity**

Informational

**Description**

The unstakeGov function currently does:

```solidity
function unstakeGov(uint16 validatorId, uint256 amount)
    external
    nonReentrant
    onlyByOwnGov
    returns (uint256 amountRestaked)
{
    _rebalance();
    (bool active, , uint256 stakedAmount, ) =
        plumeStaking.getValidatorStats(uint16(validatorId)); // total stake on this
            validator

    if (active && stakedAmount > 0 && stakedAmount >= amount) {
        amountRestaked = plumeStaking.unstake(uint16(validatorId), amount);
    }
}
```

Here, `stakedAmount` is the total amount staked on the validator by all users, not the amount staked by this contract. The function should instead compare amount against this contract’s own stake on that validator, using:

```solidity
uint256 validatorStakedAmount =
    plumeStaking.getUserValidatorStake(address(this), validatorId);
```

so the precondition matches what this contract is actually allowed to unstake.

In practice, if amount is larger than this contract’s stake, plumeStaking.unstake will revert internally, so funds are not at risk. However, the local check is misleading: it may pass when this contract’s own stake is smaller, and the revert will only happen deeper in the staking contract. This makes the API less clear and can cause unnecessary reverts instead of failing fast with an accurate condition.

The suggested fix is to keep active from getValidatorStats, but replace the stakedAmount comparison with a comparison against `getUserValidatorStake(address(this), validatorId)`.

**Team Response**

Acknowledged.

### [I-02] ERC20 Transfer Handling in recoverERC20

**Severity**

Informational

**Description**

The function uses:

```solidity
require(IERC20(tokenAddress).transfer(to, tokenAmount), "recoverERC20: Transfer
    failed");
```

This assumes every ERC20 token returns a bool. Many tokens do not return a boolean at all, which will cause this call to revert even when the transfer would succeed.

Because of this, the contract may be unable to recover certain ERC20 tokens sent to it by mistake.

Recommendation

Use `SafeERC20.safeTransfer`instead. Safe transfer helpers correctly handle tokens that do not return a boolean.

**Team Response**

Acknowledged.
