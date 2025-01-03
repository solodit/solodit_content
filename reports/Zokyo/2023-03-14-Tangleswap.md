**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Pre-estimating stakeEndBlock can lead to miscalculated rewards

**Severity**: Medium

**Status**: Resolved

**Description**

On staking void token , user.stakeEndBlock is being estimated in the future according to this

But it is assumed that the blockTime is changeable (i.e. depending on the chain contract deployed on) and changing blockTime during staking period causes tokenomical issues. Suppose in a scenario in which blockTime is increased, while a given investor is having a stake already for a period of one year and the stakeEndBlock is estimated to be around that stakeEndTime also after one year. If the blockTime due to chain parameter changes and admin updating the value, the stakeEndBlock shall be driven away after that stakeEndTime in an unpredictable.
To summarize, relying on blockTime to estimate how much reward should be taken in the future by estimating stakeEndBlock can lead to a misalignment between stakeEndBlock and stakeEndTime which in the end leads to a miscalculated reward received.

**Recommendation**: 

Relying on estimating rewards based on a blockTime that should be fixed for more than 3 years is the crux of this issue of misalignment of stakeEndTime and stakeEndBlock. A recommended way to deal with it is to adopt the reward model of PancakeSwap/SmartChefInitializable


block.number & lastRewardBlock get us the actual block count difference not just an estimation, therefore, developers shall need to add a checkpoint like lastRewardBlock for each user (i.e. mapping) and calculate reward based on the actual difference.


### Pending rewards are calculated incorrectly

**Severity**: Medium

**Status**: Resolved

**Description**

Admin can modify rewardPerBlock which can lead to issues. The code at line 135, line 185, and line 223 calculates the pending reward based on current value of rewardPerBlock rather than its value at the time of staking and account for changes during the stake period. Currently, any changes to the value of this variable in between are not considered while paying the rewards.
Example: A user stakes for period in range (T, T + 100) when rewardPerBlock was 10. Suppose at T + 20 reward is set to 5 and at T + 50 = 40. Ideally, user should get rewarded as follows:
[ blockCount(T, T + 20) * 10 + blockCount(T+ 21, T+50) * 5 + blockCount(T + 51, T + 100) * 40 ]
where blockCount(a,b) is the number of blocks in between the time a and b.
But, contract will reward (T, T+ 100) * whateverOwnerHasSet (at the moment of withdraw) and thus any changes to rewardPerBlock in between gets lost.

**Recommendation**: 

A recommended way to deal with it is to adopt the reward model of PancakeSwap/SmartChefInitializable. To achieve that assign for each user their own rewardPerBlock, we suggest you something like this code snippet which shall help also in issue#1.





### Centralization Risk

**Severity**: Medium

**Status**: Resolved

Admin enjoys too much authority. The general theme of the repo is that admin has power to call several functions like emergencyRewardWithdraw, recoverToken, general setters/mutators. Some functions can be more highly severe to be left out controlled by one wallet more than other functions; depending on the intentions behind the project.

**Recommendation** 

Apply governance / use multisig wallets.

## Low Risk

### No validation applied on the constructor arguments

**Severity**: Low

**Status**: Resolved

**Description**

No require statements to validate inputs of constructor during deployment.

**Recommendation**: 
Add require statements similar to the following:
```solidity
require(address(voidToken) != address(0));
require(minimumStakeableAmount >  0);
require(bonusEndBlock > startBlock )
require(blockTime > 0);
require(rewardPerBlock > 0);
```

### Revert messages are too long and costly

**Severity**: Low

**Status**: Resolved

**Description**

Error messages in some require statements are long and considered costly in terms of gas.

**Recommendation**:

Write shorter error messages

Use custom errors which is the goto choice for developers since solidity v0.8.4 details about this shown here: soliditylang.

## Informational

### Using msg.sender along with _msgSender()

**Severity**: Informational

**Status**: Resolved

**Description**

Confusion arises from using _msgSender() and msg.sender in the same contract. _msgSender() here returns msg.sender but it is recommended from a coding perspective to make it consistent. It shall be problematic though if TangleswapVEStaking is inherited by another contract that overrides _msgSender().

**Recommendation**: 

replace msg.sender by _msgSender()

### Unneeded use of keyword storage

**Severity**: Informational

**Status**: Resolved

**Description**

Literal storage being used in view functions unnecessarily for userInfo in pendingRewards, getUserInfo and isLockTimeExpired

**Recommendation**: 

replace with memory instead.

### Lock Solidity version

**Severity**: Informational

**Status**: Resolved

**Description**

All contracts, Lock the pragma to a specific version, since not all the EVM compiler versions support all the features, especially the latest ones which are kind of beta versions, So the intended behavior written in code might not be executed as expected. Locking the pragma helps ensure that contracts do not accidentally get deployed using, for example, the latest compiler, which may have higher risks of undiscovered bugs. 

**Recommendation** 

fix version to 0.8.17

9. Unnecessary redundant branches at lines 132, 120
Severity: Informational
Status: Resolved
Description: If statements at line 120 and 132 are repeated unnecessarily. 
Recommendation Code can be re-arranged to have only 1 branch (if statement) to save some gas.

