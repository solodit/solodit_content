**Auditors**

[Guardian Audits](https://twitter.com/guardianaudits)

# Findings

## Medium Risk

### Global-1 | Centralization Risk

**Description**

Elevated privilege leads to multiple DoS attack vectors:

When a harvest is called, `_updatePosition` relies on there being enough OATH in the contract to cover the relic’s accumulated rewards. If there is an insufficient amount of OATH, users would be unable to withdraw without giving up their entire rewards.

The `operator` role is able to add and modify the `rewarder` for a given pool. If the `rewarder` lacks the `rewardToken` funds to pay out a relic or reverts in some other manner, users would be unable to harvest their rewards.

If an operator sets an `emissionSetter` that specifies an emission rate greater than the upper bound of 6e18, the pool will not be able to be updated so no position would be able to be updated. Therefore deposits and regular withdrawals would be halted due to their reliance on `_updatePosition`.

**Recommendation**

Ideally only multi-sig addresses are granted the `operator` role and/or introduce a timelock for improved community oversight.

### RLQ-1 | Inefficient Reward Design

**Description**

If `userA` has their funds deposited but does not call `updatePosition` for a significant period of time, the next time they update their position they receive rewards based on the level of their position the last time it was updated. In this case `userA` earns less than `userB`, who regularly calls `updatePosition` as soon as their relic’s maturity reaches the next level and therefore received more rewards at the higher levels.

**Recommendation**

If this is not desired behavior, consider evaluating the current level of the user’s relic in `_updatePosition` for distributing rewards, therefore removing the need to call `updatePosition` regularly. If this approach is taken, consider introducing logic that prevents users from simply holding their positions to receive all of their rewards based on a higher level, when in reality a portion of those rewards should have been weighted at a lower level.

### RLQ-2 | Withdrawal Maturity Setback

**Description**

Because the `position.entry` increases on withdraw in `_updateEntry`, the position’s maturity decreases. As a result, when `_updateLevel` is called, a user’s position may be set back to a lower level and most likely a lower allocation. If the user’s whole position reached a certain level, there is no consequence to having the post-withdraw amount stay at that level.

**Recommendation**

If this is intended behavior, leave as is. Otherwise, do not allow for `position.level` to be decreased on withdraw until the whole position is withdrawn and the relic is burned.

### RLQ-3 | Burn-on-Transfer Tokens

**Description**

Deposits and withdrawals are not capable of handling certain deflationary tokens. A burn-on-transfer token, complying with the `IERC20` interface of `_lpToken`, would make the `position.amount` seem higher than the true amount for a given relic. As a result, `emergencyWithdraw` would surely fail as there are not enough funds to cover the transfer unless someone else’s deposit is eaten into.

**Recommendation**

Verify whether support for burn-on-transfer tokens is desired. If so, compare balance after a transfer to the balance before transfer to see the amount that is received in the contract.

### RLQ-4 | Incorrect Truncation Avoidance

**Description**

Lines 508 and 511-512 are mathematically equivalent, but they should be swapped in order to avoid truncation.
An example to demonstrate this:
If `oldValue` was 1 wei but `addedValue` was 1 ether, the `else if` branch on line 510 would be entered. `weightOld` would be rounded down to 0 and `weightNew` would be exactly 1e18. However, if the branch on line 508 were to be entered instead, you would get slightly less than 1e18 as the `weightNew` - providing a more accurate result.

Vice versa, if `oldValue` was 1 ether but `addedValue` was 1 wei, the `if` branch on line 508 would be entered. In this case, the `weightNew` produced would be 0 as the denominator is larger than the numerator. However, if line 510 was entered, the `weightNew` produced would be 1 - providing a more accurate result.

**Recommendation**

Swap the weight calculation logic between lines 508 and 511-512 in order to avoid the truncation inaccuracy.

### REW-6 | Lost Rewards Upon Withdrawal

**Description**

If you deposit but then withdraw even a tiny amount before the `cadence` is reached, the `lastDepositTime` is reset to 0 and you must deposit again in order to be able to claim a deposit bonus.

**Recommendation**

Verify whether this is expected behavior. If not, either provide a weighted bonus or in the withdraw function `delete` the `lastDepositTime` only if `_claimDepositBonus` returns true or the user’s new position amount is less than the minimum.

## Low Risk

### RLQ-5 | Unclear Naming

**Description**

Although `_poolBalance’s` notice states that it returns "The total deposits of the pool's token”, it weighs the balance of each level of the pool by its corresponding allocation. Because it doesn’t simply sum up the balance of each level, it doesn’t reflect the total deposits. Rather, it reflects the balance of the pool adjusted by level allocation.

**Recommendation**

Update the comment to accurately reflect what the function is returning.

### RLQ-6 | Redundant Pool ID Read

**Description**

In line 397, `position.poolId` is accessed yet the value is already stored in the variable `poolId`.

**Recommendation**

Replace `position.poolId` with `poolId` for the transfer.

### DESC-1 | Bad Decimal String UX

**Description**

There are a few instance where `generateDecimalString` produces strings which may be further refined.

If the decimals argument is set to 0, a number with an appended decimal point is produced such as “1.”.

The function does not remove trailing zeroes. For example, `generateDecimalString(10, 2)` produces “0.10”.

The function produces varying strings for equivalent numbers. For example, `generateDecimalString(10, 1)` produces “1.0” while `generateDecimalString(1, 0)` produces “1.”.

**Recommendation**

Verify that this is expected behavior. Otherwise, rectify the string generation using https://gist.github.com/wilsoncusack/d2e680e0f961e36393d1bf0b6faafba7 or Uniswap’s NFTDescriptor as a model.
