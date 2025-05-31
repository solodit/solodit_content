**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### The owner can change the reward rate.
**Description**

CloneRewarderTime.sol: function setReward PerSecond(). Function allows the owner to change reward per second without updating the current "accToken1PerShare" of the pools. This way, users might lose their pending rewards in case the owner sets a lower value or a 0.

**Recommendation**

Update all the pools before setting a new reward per second value.

**Re-audit comment**

Resolved.

Post audit:

The SharkSwap team verified that the Governance contract will always be calling "updatePool" function before setting new parameters and users will be notified within the dApp about all future changes in advance. However, such mandatories are not present in the current code.

### The owner can set the pool settings without updating the pool.
**Description**

MiniChefV2.sol: function set(). The owner can set new allocation points to the pool without updating the pool, which can lead to the loss of users' pending rewards. The owner can also reset the rewarder contract for the pool, which can also lead to the loss of pending rewards on the previous rewarder.

**Recommendation**

Update the pool before setting new parameters. Consider transferring the ownership of the contract to the DAO in the future and notify users about all the changes in advance so that they can claim their rewards in time.

**Re-audit comment**

Verified.

Post-audit:

The SharkSwap team verified that the ownership is set to multi-sig DAO contract and users will be notified about any changes in advance.

## Low Risk

### Rewards are claimed for "to" address instead of msg.sender.
**Description**

MiniChefV2.sol: function deposit(), line 209.

Based on the logic of other functions, such as withdraw(), harvest(), the rewards should be claimed for msg.sender and transferred to "to" address. However, during depositing, rewards are claimed for "to" address by passing "to" as the second parameter in _rewarder.onSushiReward() call. The issue is marked as low since it doesn't cause any fund loss for the provided address, but it should be verified whether this logic works as intended.

**Recommendation**

Pass msg.sender as a second parameter in_rewarder.onSushiReward() to claim rewards for msg.sender or verify that rewards should be claimed for "to" address.

**Re-audit comment**

Verified.

From the client.

According to the team, "to" parameter in the "deposit" function is a beneficiary, unlike the cases of withdrawal or harvest, when the beneficiary is msg.sender. Hence, the different params passed to onSushiReward.

## Informational

### Incomplete code sections in SushiMaker.
**Description**

There are to-dos in the source code of the SushiMaker contract on lines 88, 208, 215.

**Recommendation**

Resolve the issues with these sections.

**Re-audit comment**

Unresolved.

From the client.

Fixes are left for future updates.

### Unnecessary gas consumption from public visibility.
**Description**

The following functions are not called internally but have public visibility.
• CloneRewarderTime.sol: init(), setReward PerSecond(), reclaim Tokens(), transferLocks().
• MiniChefV2.sol: poolLength(), add(), set(), setSushi PerSecond(), setMigrator(), migrate(), deposit, withdraw(), harvest(), withdrawAndHarvest(), emergency Withdraw().
• Shark Token.sol: mint().
• SushiMaker.sol: setDestination().
• UniswapV2Router02.sol: quote(), getAmountOut(), getAmountIn(), getAmountsOut(), getAmountsIn().

**Recommendation**

They should be declared external.

**Re-audit comment**

Unresolved.

From the client.

Fixes are left for future updates.
