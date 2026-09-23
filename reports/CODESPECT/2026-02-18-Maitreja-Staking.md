**Auditors**

[shaflow01](https://x.com/shaflow01), [jecikpo](https://github.com/jecikpo)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/054_CODESPECT_MAITME.pdf)

# Findings

## Low Risk

### [L-01] Withdraw execution leads to lost rewards if not enough assets held by treasury

**Files:** [ProgressiveStaking.sol](https://github.com/whaleden-mjtd/maitme-contracts-staking/blob/ce61102843ceb1c27d875b394490ff859577016c/src/ProgressiveStaking.sol#L304)

**Description:**

The `executeWithdraw(...)` function allows the holder of a stake to execute assets withdrawal. Inside the functions the rewards are calculated for the stake and are sent to the owner on condition that there is enough reward asset held at the contract:

```solidity
if (rewards > 0 && treasuryBalance >= rewards) {
    treasuryBalance -= rewards;
    stakingToken.safeTransfer(msg.sender, rewards);
    emit RewardsClaimed(msg.sender, stakeId, rewards, block.timestamp);
}
```

While this is done according to the protocol’s design, the issue is that if the withdrawal is partial and the `treasuryBalance` is insufficient the function still updates the `lastClaimTime`:

```solidity
position.lastClaimTime = block.timestamp;
```

which means that the rewards for the remaining amount are also lost.

**Impact:** Loss of rewards for the remainder of the stake.

**Recommendation:** It is recommended to only update the `lastClaimTime` when the rewards are actually successfully claimed. This way in case the `treasuryBalance` is insufficient, the stake owner will still be able to collect rewards later for the remainder of the stake once the treasury is replenished.

**Status:** Fixed

### [L-02] Withdrawal request does not prevent rewards from accrual

**Files:** [ProgressiveStaking.sol](https://github.com/whaleden-mjtd/maitme-contracts-staking/blob/ce61102843ceb1c27d875b394490ff859577016c/src/ProgressiveStaking.sol#L240)

**Description:**

The `ProgressiveStaking` contracts provides a two-step withdrawal process where the staker first needs to call `requestWithdraw(...)`, then `executeWithdraw(...)`. where the `executeWithdraw(...)` can be called after `NOTICE_PERIOD` passes. The problem is that there is no effect on the stake after calling `requestWithdraw(...)`, so a user could call it immediately after creating a stake and then wait just until he wants to unstake and call `executeWithdraw(...)`, which in practical terms mean that the `NOTICE_PERIOD` can be ignored as longs the user calls `requestWithdraw(...)` early.

**Impact:** Practical lack of expected notice period for withdrawing the stake.

**Recommendation:** The `requestWithdraw(...)` function should stop accrual of the rewards.

**Status:** Fixed

## Informational

### [I-01] Emergency withdrawals blocked in case of too many stakes

**Files:** [ProgressiveStaking.sol](https://github.com/whaleden-mjtd/maitme-contracts-staking/blob/ce61102843ceb1c27d875b394490ff859577016c/src/ProgressiveStaking.sol#L341)

**Description:**

For the `StakePosition` dynamic array, there is no maximum length limit. Although the array decreases when positions are removed, a user holding too many positions at the same time could have the emergency withdrawals affected. Positions of Web2 accounts, are managed under a single admin address. If this address holds an excessive number of users’ positions, in case of a security issue, emergency withdrawals may fail due to out-of-gas errors. And it may affect the `calculateTotalRewards(...)` view function. The emergency withdrawal functionality will reach its gas limits between 7,672 and 8,172 amount of positions for a single address.

**Impact:** Emergency withdrawal functionality not available for Web2 accounts.

**Recommendation:** Place a reasonable limit on how many stakes each address can open. For Web2 distribute the stakes across multiple custodial addresses.

**Status:** Fixed

### [I-02] The MIN_STAKE_AMOUNT check is not applied during withdrawal requests

**Files:** [ProgressiveStaking.sol](https://github.com/whaleden-mjtd/maitme-contracts-staking/blob/ce61102843ceb1c27d875b394490ff859577016c/src/ProgressiveStaking.sol#L240)

**Description:**

In the `stake(...)` function, there is a position size check that requires the stake amount to be greater than `MIN_STAKE_AMOUNT`.

```solidity
function stake(uint256 amount) external nonReentrant whenNotPaused {
    if (amount == 0) revert ZeroAmount();
    if (amount < MIN_STAKE_AMOUNT) revert StakeAmountTooLow();
    //...
}
```

However, in the `requestWithdraw(...)` function, there is no check to ensure that the remaining position size after withdrawal is still greater than `MIN_STAKE_AMOUNT`. This may result in positions smaller than `MIN_STAKE_AMOUNT` existing in the contract.

**Impact:** Unexpected small positions may appear in the contract, which is unfavorable for management.

**Recommendation:** It is recommended to add a check in `requestWithdraw(...)`. If the remaining position size is not 0, it must be greater than `MIN_STAKE_AMOUNT`.

**Status:** Fixed

### [I-03] Withdrawals for web2 users may be delayed

**Files:** [ProgressiveStaking.sol](https://github.com/whaleden-mjtd/maitme-contracts-staking/blob/ce61102843ceb1c27d875b394490ff859577016c/src/ProgressiveStaking.sol#L253)

**Description:**

The document mentions that for web2 users in the system, tokens are managed in a single admin account. However, a single account can have at most 10 pending withdrawal requests within 90 days.

```solidity
function requestWithdraw(uint256 stakeId, uint256 amount) external nonReentrant whenNotPaused {
    //...
    if (pendingWithdrawCount[msg.sender] >= MAX_PENDING_WITHDRAWALS) revert TooManyPendingWithdrawals();
    //...
}
```

This makes it difficult for a single account to manage too many users, as their withdrawal requests may be delayed due to the admin account reaching the maximum number of pending requests.

**Impact:** Designing the system to manage all web2 users’ tokens in a single admin account may be unfeasible.

**Recommendation:** It is recommended to delegate web2 users in batches to multiple accounts or to implement a dedicated withdrawal function for this special case.

**Status:** Fixed

### [I-04] Withdrawal requests that are canceled or executed cannot be distinguished

**Original severity:** Best Practices

**Files:** [ProgressiveStaking.sol](https://github.com/whaleden-mjtd/maitme-contracts-staking/blob/ce61102843ceb1c27d875b394490ff859577016c/src/ProgressiveStaking.sol#L330)

**Description:**

Calling the `executeWithdraw(...)` function to process a withdrawal request sets the `executed` field to `true`.

```solidity
function executeWithdraw(uint256 stakeId) external nonReentrant {
    //...
    request.executed = true;
    //...
}
```

Calling the `cancelWithdrawRequest(...)` function to cancel a withdrawal request also sets the `executed` field to `true`.

```solidity
function cancelWithdrawRequest(uint256 stakeId) external nonReentrant {
    //...
    request.executed = true; // Mark as executed to prevent reuse
    //...
}
```

There is no other field to distinguish canceled and executed withdrawal requests in the history.

**Impact:** The historical withdrawal records returned by the `getPendingWithdrawals(...)` function cannot distinguish between canceled and executed requests.

**Recommendation:** It is recommended to add a field to distinguish between canceled and executed withdrawal requests.

**Status:** Fixed
