**Auditors**

[AuditOne](https://twitter.com/auditone_team)

# Findings

## High Risk

### Initializers could be front-run 

**Description:**

Initializers could be front-run, allowing an attacker to either set their values, take ownership of the contract, and in the best case force a re-deployment. The script deployment is permitting this front-run attack.

**Recommendations:** 

Make the initialize function executed in the same transaction.



## Medium Risk

### The function getCurrentPeriod() returns wrong value which results in staking before startDate

**Description:** 

If block.timestamp < startDate, then the function returns 0, while it also returns 0 if block.timestamp - startDate < periodLength. Due to return of 0 for block.timestamp < startDate, it is possible to stake before startDate.

**Recommendations** 

Change the function accordingly.



### Later claimer might not be able to claim reward 

**Description:**If User is claiming reward very late (many years after) then getAvailableReward might revert with out of gas. This is because currentAvailableRewardPeriod might become greater than rewardPeriods. Since contract is not checking such scenario, looping the periods from 0 to currentAvailableRewardPeriod might cause out of gas error.

**Recommendations:**

 Loop should not go beyond rewardPeriods, if currentAvailableRewardPeriod>rewardPeriods



### It's not guaranteed that users will receive the rewards 

**Description:** 

The users should believe in the contract owner to add the rewards after deployment, if the user stake waiting to receive a reward the owner could simply not add the rewards into the Staking contract.

The owner should send the rewards through initialize function guaranteeing that the user will have the rewards.

**Recommendations:**

 It is recommended to add functionality into initialize to validate the rewards sent to the Staking contract and add another function to retrieve not used rewards after the expiring date.



### Users can stake for more than 52 weeks Severity: Medium

**Description:** 

Docs says the user can stake from 1 to 52 weeks but this value is different from what is implemented. In initialize function is possible to set the rewardPeriods to any value, there isn't a max cap, and setting it to a higher value than 52 enables any user to stake more than 52 weeks.

**Recommendations:**

 It is recommended to add a max cap to rewardPeriods if the intention is to have a maximum of 52 weeks.



## Low Risk

### Unsafe ERC20 methods 

**Description:** 

There are many Weird[ ERC20 Tokens that won'](https://www.hacknote.co/17c261f7d8fWbdml/doc/182a568ab5cUOpDM#n_undefined_h3_0)t work correctly using the standard IERC20 interface. For example, IERC20(token).transferFrom() and IERC20(token).transfer() will fail for some tokens as they may not conform to the standard IERC20 interface.

**Recommendations:**

Consider using SafeERC20 for transferFrom, transfer and approve.



### Both startDate and endDate should not be inclusive for staking

**Description:**

 The staking should only be available from startDate(inclusive) to endDate exclusive. For ex: if startDate is 0 and rewardPeriods is 3 and periodLength is 5, then staking time should be 0 to 14(inclusive). endDate will be 15 which should not be included.

**Recommendations**:  Exclude the endDate while staking



## Informational

### Missing important events

**Description:**

There is no event emitted in important functions like stake, unstake, etc

**Recommendations:** 

It is recommended to add all events.



### Run out of gas in getAvailableReward

**Description:**

 The calculation for currentAvailableRewardPeriod is not capped at a maximum of rewardPeriods, after a long time this number could be great enough to make the loop run out of gas, if the user still has some rewards to receive it would not be possible.

**Recommendations:**

 Should cap the currentAvailableRewardPeriod at a maximum of rewardPeriods.



### Use != 0 instead of > 0 for unsigned integer comparison

**Description:**

To save gas in comparison you can use ≠ 0 instead of > 0

**Recommendations:** 

Change the comparison from > 0 to != 0. ![](Aspose.Words.8685a4cb-16aa-442b-8cd3-a7ceb982dc2f.025.png



### Increments can be unchecked in for-loops 

**Description:** 

Make the increments in for-loop unchecked

**Recommendations:**

Change the for-loop to make the increment unchecked



### Using private rather than public for constants saves gas

**Description:** 

If needed, the values can be read from the verified contract source code, or if there are[ multiple values t](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178)here can be a single getter function that returns[ a tuple of the valu](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178)es of all currently-public constants. Saves 3406-3606 gas in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it’s used, and not adding another entry to the method ID table.

**Recommendations:** 

Change the constant from public to private. Status: Resolved.



### Pre-increments and pre-decrements are cheaper than post- increments and post-decrements

**Description:** 

It's cheaper to use ++i or --i than i++ or i--

**Recommendations:**

It's recommended to change the increment in the loop for ++i.



### Long revert strings

**Description:**

Using a revert string of less than 32 bytes save gas.

**Recommendations:**

It's recommended to change the long revert string to a short string.



### Don’t initialize variables with the default value Severity: Quality Assurance

**Description:**

It's not necessary to declare a variable with the default value.

**Recommendations** 

It's recommended to not declare variables with the default value.



### Use Custom Errors Severity: Quality Assurance

**Description:**

 So[urce Inst](https://blog.soliditylang.org/2021/04/21/custom-errors/)ead of using error strings, to reduce deployment and runtime cost, you should use Custom Errors. This would save both deployment and runtime cost.

**Recommendations:** 

Change the errors to custom errors. Status: Resolved.



### Use calldata instead of memory for function arguments that do not get mutated

**Description:**

Mark data types as calldata instead of memory where possible. This makes it so that the data is not automatically loaded into memory. If the data passed into the function does not need to be changed (like updating values in an array), it can be passed in as calldata. The one exception to this is if the argument must later be passed into another function that takes an argument that specifies memory storage.

**Recommendations:** 

Change from memory to calldata in this instance.



### Use assembly to check for address(0)

**Description:**

Use assembly to check for address(0). 

**Recommendations:** 

Change the check for assembly check. Status: Resolved.



### Simplification loop length Severity: Quality Assurance

**Description:**

The for-loop validation is always calculation currentPeriod + 1 and it can be simplified to i <= currentPeriod.

**Recommendations:**

 Should change the for-loop to the following code:



### Redundant startDate validation in increaseRewardsScores Severity: Quality Assurance

**Description:**

In the increaseRewardsScores function, it checks if the current time is more than the start date in line 188, but this check is already made in the getCurrentPeriod function and could be removed from the increaseRewardsScore.

**Recommendations:**

Remove validation from line 188



### The _rewardPeriods variable can be omitted. Severity: Quality Assurance

**Description:** 

rewardPeriods is supposed to be the _rewardPerPeriod length, and it's possible to use the _rewardPerPeriod length directly instead of having another variable to it.

**Recommendations:**

 Remove the rewardPeriods variable and use the _rewardPerPeriod length.



### Incorrect condition on getCurrentPeriod Severity: Quality Assurance

**Description:**

If block.timestamp>=endDate then all reward period have been processed and rewardPeriods could be returned directly.



### Redundant check Severity: Quality Assurance

**Description:** 

The if check is not required. If userAvailableTokens[user][i] is zero, it won't change availableTokens.

**Recommendations:**

Remove the if condition.
