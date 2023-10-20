**Auditors**

[Guardian Audits](https://twitter.com/guardianaudits)

# Findings

## High Risk

### LPS-1 | Rewards May Be Stolen

**Description**

The `uniswapV3Staker` contract which the `LPStaker` interacts with allows any arbitrary address to directly call the `unstakeToken` function and unstake for any depositor after the incentive key `endTime`.
When a deposit is unstaked from an incentive key directly from the `uniswapV3Staker`, those rewards will be incremented for the `LPStaker` contract, but not credited towards the user who staked. Therefore malicious stakers may unstake for other stakers and immediately claim their rewards as their own by unstaking through the `LPStaker` contract.

**Recommendation**

Consider using a modified version of the `uniswapV3Staker` where the depositor must always be the one to unstake. Otherwise be sure to manage the incentive keys extremely carefully and never allow an incentive key to reach its `endTime` while users have staked for it.

**Resolution**

Key Team: A modified version of the `uniswapV3Staker` was implemented.

### STK-1 | Reward Compounds Are Sandwichable

**Description**

There exists no fee or lockup period associated with staking to receive a portion of the rewards compounded during the `updateAllRewardsForTransferReceiverAndTransferFee` function.
A malicious actor may simply buy GMXKey and stake right before the `updateAllRewardsForTransferReceiverAndTransferFee` function to immediately accrue a portion of the collected rewards that were meant to be attributed to other stakers. The malicious actor can then immediately claim these rewards, unstake and sell GMXKey after the reward compound, therefore stealing rewards from other stakers.

**Recommendation**

Consider implementing a staking/unstaking fee or a “warmup period” where stakers cannot accrue rewards.

**Resolution**

Key Team: A new approach to rewards including reward periods was adopted.

### TREC-1 | Lost WETH Rewards Upon Upgrade

**Description**

During the account transfer process, the `RewardRouterV2` does not `claimForAccount` from the `feeGMXTracker`. Therefore any accrued WETH rewards will not be transferred to the new address upon upgrade of the `TransferReceiver`.
The `TransferReceiver` will then have no means of claiming these WETH rewards and injecting them into the `Rewards` system as the `amountToMint` calculation will underflow and revert since the receiver’s staked balances have been transferred out.

**Recommendation**

Require WETH rewards to be claimed and injected into the `Rewards` system before initiating the `TransferReceiver` upgrade process.

**Resolution**

Key Team: The Rewards logic was updated to allow any remaining WETH to be collected.

## Medium Risk

### GLOBAL-1 | Centralization Risk

**Description**

The `admin` address holds the ability to negatively impact the system in numerous ways, including but not limited to:

- Take all esGMX, sGMX and bnGMX via `reserveSignalTransfer` and `signalTransfer`.
- Use the `withdrawTokens` function to take any non-WETH ERC20 rewarded to the `TransferReceiver`.
- Lock all staked Uniswap V3 LP positions by pausing the `LPStaker` contract.
- Lock all `GMXKey` and `MPKey` stakes by pausing the `Staker` contract.
- Raise fees to 100% in the `Rewards` contract.

**Recommendation**

Ensure that the `admin` address is a multi-sig, optionally with a timelock for improved community trust and oversight. Attempt to limit the scope of the admin address permissions such as locking stakes and raising fees to 100%.

**Resolution**

Key Team: We have removed the pausable modifier for functions that would lock V3 positions and privileged addresses will be multi-sigs.

### TREC-2 | Invalid Assumption

**Description**

In the `acceptTransfer` function it is assumed that `A maximum of ~7% amount of GMX (as esGMX) would be added`, however that assumption does not hold in several cases.
A user could have removed their sGMX and only been left with esGMX and bnGMX or a user may have accepted a transfer from another account which perturbed this ratio.

**Recommendation**

Do not rely on this assumption holding and remove the comment. If it is paramount that only a small percentage of esGMX is added, add an explicit check.

**Resolution**

Key Team: The comment has been removed.

### TREC-3 | Unexpected Rewards

**Description**

The allowance is used to determine how much WETH to inject into the `Rewards` system and it is incremented based on the current balance of the `TransferReceiver`.
However the balance of the `TransferReceiver` can be inflated by transferring WETH directly to the `TransferReceiver` contract. Therefore rewards that are not explicitly from GMX are able to enter the `Rewards` system.
Additionally, it is possible that `privateTransferMode` is turned off for either esGMX or bnGMX in the future, which could also potentially perturb the `Rewards` system.

**Recommendation**

Consider if outside WETH should be included in the accounted rewards. If not, implement a before and after balance check when calling `rewardRouter.handleRewards` to get the actual WETH amount received from GMX.
Additionally, have a plan for the scenario where `privateTransferMode` is turned off for either `esGMX` or bnGMX.

**Resolution**

Key Team: The recommended before and after check was implemented.

### LPS-2 | onERC721Received Reentrancy

**Description**

During the `unstakeAndWithdrawLpToken` function, the `msg.sender` may re-enter into the `onERC721Received` function upon the `withdrawToken` call by transferring the withdrawn Uniswap V3 LP NFT back to the `LPStaker`.
This reentrancy can yield an unexpected state where the token still exists in the `tokensStaked` list for the owner, but not in the `idToOwner` or `stakedIndex`. Such an unexpected state may have unintended consequences and effect frontend systems reading from the contract or third party systems built on top of the `LPStaker`.

**Recommendation**

Move the `withdrawToken` call to the end of the for loop to follow Check-Effects-Interactions. Alternatively, add a reentrancy check to the `onERC721Received` function.

**Resolution**

Key Team: Check-Effects-Interactions was adopted.

### STK-2 | Users May Stake For Others

**Description**

The `stake` function in the `Staker` contract allows users to stake for any address rather than just their own. This can cause unexpected consequences for contract systems interfacing with the `Staker` contract, especially if the necessary staking “warmup period” is implemented.

**Recommendation**

Reconsider if this feature is necessary, and if so carefully document it and consider its impacts when combined with the solution for STK-1.

**Resolution**

Key Team: Users can no longer stake for others in the `Staker` contract.

### ADM-1 | Admin Role Changes Should Be Two Step

**Description**

As addressed in GLOBAL-1, the admin address carries numerous important abilities for the system.
However the `changeAdmin` function allows the admin address to be errantly transferred to the wrong address as it does not use a two-step transfer process.

**Recommendation**

Implement a two step “push” and “pull” admin transfer process. If it is desired to have a method to relinquish ownership, implement a separate function to do so.

**Resolution**

Key Team: The recommended push and pull transfer process was adopted.

### BTOK-1 | Dangerous Approve

**Description**

The `BaseToken` only exposes the dangerous `approve` function rather than an additional alternative `increaseAllowance` function.

**Recommendation**

Implement an `increaseAllowance` function so that users may increase their allowances without risk of frontrunning.

**Resolution**

Key Team: The recommended `increaseAllowance` function was implemented.

### LPS-3 | Fee-On-Transfer Tokens

**Description**

The `LPStaker` contract is not compatible with fee-on-transfer tokens for the `rewardToken` as it relies on the `uint` returned from the `uniswapV3Staker` `claimReward` function to increment the reward mapping.
Fee on transfer tokens will cause this returned value to be inaccurate and potentially leave users unable to claim their rewards and potentially locked in the contract.
It should also be noted that rebase tokens or other balance altering tokens will not be accurately accounted for in a similar way.

**Recommendation**

Consider if fee-on-transfer, rebase, or any similar tokens should be supported. If so, add before and after balance checks for the `claimReward` function to measure the reward claimed accurately.

**Resolution**

Key Team: The `rewardToken` will never be a fee-on-transfer token.

### REW-1 | Chain Incompatibility

**Description**

The `withdrawTo` function is used on WETH in the `_transferAsETH` function. This function is supported on Arbitrum, however it is not supported on the Avalanche C-chain or other networks that GMX may deploy on.

**Recommendation**

Do not use the `withdrawTo` function when deploying on Avalanche or other chains. Instead implement a method to receive Ether and relay it to the to address.

**Resolution**

Key Team: The `withdrawTo` function has been replaced with `withdraw`.

## Low Risk

### GLOBAL-2 | Custom Reverts

**Description**

Throughout the codebase `require` statements are used when instead custom errors may be implemented with `if` condition checks.

**Recommendation**

Replace `require` statements with `if` statements and custom error reverts to save gas.

**Resolution**

Key Team: Opted to keep the `require` statements.

### GLOBAL-3 | Use Standard ReentrancyGuard

**Description**

Throughout the codebase a non OpenZeppelin `ReentrancyGuard` contract is used. The custom `ReentrancyGuard` contract is inferior as it uses a `boolean` `_guard` storage variable.

**Recommendation**

Use the OpenZeppelin `ReentrancyGuard`.

**Resolution**

Key Team: Implemented the recommended OZ `ReentrancyGuard`.

### REW-2 | Redundant Transfers

**Description**

When claiming and updating a reward for a `TransferReceiver`, the fee is first transferred to the `TransferReceiver` before being transferred to the `msg.sender`.

**Recommendation**

Consider implementing a `feeTo` address parameter on the `updateAllRewardsForTransferReceiverAndTransferFee` function so that two redundant transfers are not needed.

**Resolution**

Key Team: The suggested `feeTo` address was implemented.

### CNV-1 | Inaccurate Comment

**Description**

In the `completeConversion` and `completeConversionToMpKey` functions it is stated that `the sender's vesting tokens must be non-zero` however the sender’s vesting tokens must be zero or else the `acceptTransfer` on the `RewardRouter` will revert.

**Recommendation**

Update the inaccurate comments.

**Resolution**

Key Team: The comment was updated.

### GLOBAL-4 | Lack of Events

**Description**

Throughout the codebase there are functions that alter the contract state in a significant way without emitting an event.
For example the `signalTransfer` and `reserveSignalTransfer` ought to emit an event for third party systems to be able to read.

**Recommendation**

Emit an appropriate event whenever a significant change is made in the contract system.

**Resolution**

Key Team: The suggested events were added.
