**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Withdrawals are stuck Forever if Staking is done at a time sufficiently before endTime

**Severity**: High

**Status**: Resolved

**Description**

In contract `StakingManager.sol`, if a user stakes at a time such that the `stake.timestamp + poolDuration > EndTime`, then the users funds will be stuck forever.
This is because of the following reasons-
 `withdraw()` internally calls the `_getCurrentIntervalTimestamp()` function. 
When withdraw is called at a time such that the Endtime has passed,  `_getCurrentIntervalTimestamp()`
```solidity
function always returns endTime as the timestamp. This is because of the else if statement on line: 194.
       else if (timeStamp > configurator.getEndTime()) {
           timeStamp = configurator.getEndTime();
       }
```
The `withdraw()`  function then checks for the require statement on line: 62-
```solidity
      require(currentIntervalTimestamp > stake.timestamp + poolDuration, "Withdraw not yet available");
```
Substituting `endTime` for `currentIntervalTimestamp`, we get
```solidity
=> require(endTime > stake.timestamp + poolDuration, "Withdraw not yet available")
```
But, as we know that  `stake.timestamp + poolDuration > EndTime`
```solidity
=>  require(false, "Withdraw not yet available")
=> revert with “Withdraw not yet available”
```

For example, consider that a user stakes at time t1 which is done at a time way after halfway of `startTIme` and `endTime` duration(refer the diagram of non-withdrawable scenario above). Now if the `poolDuration` is such that when we add the `stake.timestamp + poolDuration`, we get a time which is beyond `endTime`, then we get a scenario in which the user is unable to withdraw any of his stake.

**PoC**: 

The following changes to the code are made in the file StakeManager.test.ts to demonstrate the above scenario:
```solidity
   beforeEach(async function () {
     // We know that the EndTime - StartTime = 2 years
     // And for PoolID 0, 0.5 years
     // So we stake at a time little bit after 1.5 Years after StartTime


     const startTime = await stakingConfigurator.getStartTime();
     await time.increaseTo(Number(startTime) + DAY_IN_SECONDS * 365 * 1.5 + 10_000);
     const lat = await time.latest();
    
     console.log("Staked at: ", lat);


     await token.connect(user1).approve(stakingManager, depositAmount);
     await stakingManager.connect(user1).deposit(poolId, depositAmount);


     // Simulate passing time when the staking period has ended
     const endTime = await stakingConfigurator.getEndTime();


     // We travel to future skipping 5 years and try withdrawing after that
     await time.increaseTo(Number(endTime) + DAY_IN_SECONDS * 365 * 5);
     const lat2 = await time.latest();
    
     console.log("Withdrawal at time: ", lat2);
     console.log("End Time: ", endTime);
   });

```



   

**Recommendation**: 

It is advised to carefully review the business and operational logic of the code, and make changes to the `_getCurrentIntervalTimestamp()` and/or the require statement on line: 62 to make sure that the above scenario does not occur and funds are withdrawable.

### Withdrawals are stuck Forever if Staking is done at a time after `endTime`

**Severity**: High

**Status**: Resolved

In contract `StakingManager.sol`, if a user stakes at a time such that the `stake.timestamp > EndTime`, then the users funds will be stuck forever. In other words, if the user stakes or deposits his funds after endTime, then user’s funds will be stuck forever in the contract and non-withdrawable.

**PoC**: 

The following changes to the code are made in the file StakeManager.test.ts to demonstrate the above scenario:
```solidity
 describe("Withdraw", function () {
   const poolId = 0;
   const stakeId = 0;
   const depositAmount = ethers.parseEther(Number(100_000).toString());


   beforeEach(async function () {
     const startTime = await stakingConfigurator.getStartTime();
     await time.increaseTo(Number(startTime) + DAY_IN_SECONDS * 365 * 2 + 10_000);
     const lat = await time.latest();
    
     console.log("Staked at: ", lat);


     await token.connect(user1).approve(stakingManager, depositAmount);
     await stakingManager.connect(user1).deposit(poolId, depositAmount);


     // Simulate passing time when the staking period has ended
     const endTime = await stakingConfigurator.getEndTime();


     await time.increaseTo(Number(endTime) + DAY_IN_SECONDS * 365 * 10); // Forwarding time in order to withdraw after 10 years
     const lat2 = await time.latest();
    
     console.log("Withdrawal at time: ", lat2);
     console.log("End Time: ", endTime);
   });
```




**Recommendation**: 

It is advised to disallow staking after endTime and add a require check in `deposit()` to implement the same.

**Comments**: 

The client sufficiently added a check to ensure that staking is not allowed after endTime.

## Medium Risk

### `getRewards` For Stake Would Give Incorrect Rewards When User Has Withdrawn Already

**Severity** - Medium

**Status** - Resolved

**Description**

When a user withdraws their stake their `stake.status` is set to Withdrawn but the `stake.amount` still persists , though this is harmless since user can’t withdraw twice (due to the `stake.status` check in withdraw) this might lead to incorrect results when `getRewardsForStake` is queried . 
If a user has already withdrawn their stake the `stake.amount` is still the same and the function `getRewardsForStake` will still return the reward amount which is calculated with the `stake.amount`.

**Recommendation:**

Empty out the `stake.amount` on withdrawal.

### Introduce an emergency Withdraw Function

**Severity** - Medium

**Status** - Acknowledged

**Description**

The token being used in the system is the SnailBrook Token which is a token with a blacklist , i.e. an address could be blacklisted if malicious activities are found associated with that address. Therefore , it is possible that a user deposits some tokens into the `StakingManager` contract and due to some suspicious activities that user gets blacklisted , that user would not be able to call withdraw since the token transfer would revert and those funds would be stuck in the contract forever . Instead have an emergency withdraw function where the owner can rescue these stuck tokens.

**Recommendation:**

Have an emergency withdraw function where the owner can rescue these stuck tokens.It must be ensured that the address calling this function is trusted and is ideally a multisig with a timelock since it will introduce centralisation risks.

## Low Risk

### Missing zero address check for the constructor of `PearlPointsCalculator` & `StakingReardsManager`

**Severity**: Low

**Status**: Resolved

**Description**

In the contract PearlPointsCalculator.sol, there is missing zero address check for `stakingManager` in constructor. Given that `stakingManager` is immutable, if it is accidentally set to zero address, it cannot be changed again.

Similarly, in `StakingRewardsManager.sol`, there is missing zero address check for token & configurator in the constructor. This can lead to token and configurator accidentally set to zero address as well, and cannot be set again.

**Recommendation**: 

It is advised to add a zero address check for 
1) stakingManager in the constructor() of PearlPointsCalculator.sol 
2) token & configurator in the constructor() of StakingRewardsManager.sol


### Renounce Ownership can be called accidentally

**Severity**: Low

**Status**: Resolved

**Description**

In PearlPointsCalculator.sol & StakingConfigurator.sol: The `renounceOwnership` function can be called accidentally by the admin, leading to immediate renouncement of ownership to a zero address after which any `onlyOwner` functions will not be callable which can be risky.

**Recommendation**: 

It is advised that the Owner cannot call `renounceOwnership` without first transferring ownership to a different address. Additionally, if a multi-signature wallet is utilized, executing the `renounceOwnership` method for two or more users should be confirmed. 
Alternatively, the Renounce Ownership functionality can also be disabled by overriding it. 

### Transfer Ownership is 1-step and can lead to irrecoverable transfers

**Severity**: Low

**Status**: Resolved

**Description**

In PearlPointsCalculator.sol & StakingConfigurator.sol: The `transferOwnership()` function in contract allows the current admin to transfer his privileges to another address. However, inside `transferOwnership()` , the `newOwner` is directly stored into the storage owner, after validating the `newOwner` is a non-zero address, and immediately overwrites the current owner. 

This can lead to cases where the admin has transferred ownership to an incorrect address
and wants to revoke the transfer of ownership or in the cases where the current admin comes to know that the new admin has lost access to his account.

**Recommendation** 

It is advised to make ownership transfer a two-step process. Or use Openzeppelin’s `Ownable2Step` instead.
**Refer**- https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol 

### The nonReentrant modifier is unnecessary in `depositRewards` of StakingRewardsManager

**Severity**: Informational

**Status**: Resolved

**Description**

The nonReentrant modifier is unnecessary in `depositRewards()` function of StakingRewardsManager.sol.  This is because there is no practical scenario of a reentrancy vulnerability in `depositRewards()` function. In addition to that the function is already following the Checks-Effects-Interactions pattern

**Recommendation**: 

The `nonReentrant` modifier can be safely removed from `depositRewards()` function in order to save gas.
