**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### New epoch can't be started in PerpetualTokensFarmSDK.
**Description**

Perpetual TokensFarmSDK.sol
In order to start a new epoch, allStakesAreMigrated should be equal to true. (startNewEpoch(), line 576). However, in the initialize() function it is set as false (Line 282). Due to this, epoch 1 can't be started. Also, during the execution of function startNewEpoch() allStakesAreMigrated should be set as false (Line 617). The issue was found during the 3rd iteration of the audit in the newly added migration functionality.

**Recommendation**

Set 'allStakesAreMigrated as true during initializing and as false during startNewEpoch() execution.

**Re-audit comment**

Resolved.

Post-audit:
The issue was found simultaneously with the TokensFarm team, thus by the time of the reporting the issue was meant to be resolved. Nevertheless, the team of auditors has checked the solution as well.

### Usage of msg.value in a loop.
**Description**

Perpetual Tokens FarmSDK.sol, function notice Reduced StakeWithout StakeId(), line 1433.
TokensFarmSDK.sol, function notice ReducedStakeWithout StakeId(), line 1451.
The_payoutRewardsAndUpdateState() function is executed multiple times in the loop. There is a call of the_erc20Transfer() function in this function, where msg.value is used and is added to the 'totalFeeCollectedETH storage variable. This way, the same msg.value is added multiple times. This can potentially prevent collecting ETH fees. Reference: https://github.com/crytic/slither/wiki/Detector-Documentation/#msgvalue-inside-a-loop

**Recommendation**

Avoid using msg.value in the loop. Add msg.value only once to 'totalFee CollectedETH .

**Re-audit comment**

Resolved

### The value in the `idInList` mapping can be wrong.
**Description**

TokensFarm.sol: function finalise Deposit(), line 1103.
TokensFarmSDK.sol: function finalise Deposit(), line 1061. Perpetual Tokens FarmSDK.sol: function finalise Deposit(), line 1114. Perpetual TokensFarm.sol: function finalise Deposit(), line 1203. In case the user doesn't have any deposit requests, they get removed from the waitingList', and their ID in the waiting list is given to the last user in the 'waitingList'. However, the value in the `idInList mapping is not updated for 'lastUserInWaiting ListArray`. Due to this, the finalization of the request for 'lastUserInWaitingListArray can be blocked.

**Recommendation**

Update the value in the 'idInList' mapping for 'lastUserInWaitingListArray with deletedUserId'. Delete the value from the 'idInList' mapping for the user. Take into account that 'lastUserInWaitingListArray can be equal to 'user' in case there is only one address in the waiting list.

**Re-audit comment**

Resolved

### Users can withdraw the same stake more than once with the emergency withdrawal function.
**Description**

TokensFarm.sol: function emergency Withdraw().
Perpetual TokensFarm.sol: function emergency Withdraw(). Users can execute the emergency Withdraw() function with the same stake, even when the amount of the stake is 0. Due to this, the 'participants' array is updated every time, deleting the first user from the array. This way, users can delete all users from the 'participants array and block all withdrawal functions to them.

**Recommendation**

Do not let users conduct emergency withdrawal of the same stake more than once.

**Re-audit comment**

Resolved

### The wrong stake amount is stored.
**Description**

Perpetual TokensFarm.sol: function_deposit(), line 1083.
The '_amount parameter is stored in 'stake.amount . In case there is a stake fee, a greater amount would be stored instead of the value after taking the fee.

**Recommendation**

Store 'staked Amount in 'stake.amount`.

**Re-audit comment**

Resolved

## Medium Risk

### User's stake can be reduced before stakes are finalized in SDK contracts.
**Description**

Perpetual TokensFarmSDK.sol
TokensFarmSDK.sol
During the execution of the makeDepositRequest() function, the value in the totalActiveStakeAmount[user] mapping is updated. This value is used in the notice Reduced StakeWithout StakeId() function to verify that the user has sufficient stake amount. In case this function is called before the finalization of the stake, the user would be able to withdraw their non-finalized stake. Even though there is a validation that the stake is finalized (Tokens FarmSDK.sol, line 1442), the issue still can occur when the user has only one stake that is not yet finalized. The issue is marked as medium since only the owner or the contract admin can execute these functions.

**Recommendation**

Ensure that stakes cannot be reduced or withdrawn via `noticeReducedStakeWithoutStakeld` before they are finalized, or ensure that accounting for `totalActiveStakeAmount` and `totalDeposits` correctly handles unfinalized stakes during such operations.

**Re-audit comment**

Resolved.

Post-audit:
The deposits that are not finalized can still be processed, potentially breaking the `totalDeposits` variable, and user's 'totalActiveStakeAmount . Consider such a scenario:
1) The user has performed 3 different stakes with the following amounts:
a) Stake '0' with amount = 1 token.
b) Stake `1` with amount = 2 token.
c) Stake `2` with amount = 3 token.
2) Stake `0` and `2` are finalized, leaving stake `1` unfinalized.
a) 'totalDeposits is equal 1+3=4 (Since only stakes `0` and `2` are finalized.)
3) The user withdraws the amount of 3 tokens with the notice Reduced StakeWithout StakeId() function. In this case:
a) the amount of stake `0` will be equal 0.
b) the amount of stake `1` (unfinalized) will also be equal 0.
c) the amount of stake `2` will still be equal to 3.
d) 'totalDeposits will be equal to 1 (Despite the fact that there is a finalized stake `2` with the amount of 3).
After this, the finalization of stake `1` will increase the 'totalDeposits despite the fact that the amount of stake '1' is equal to 0 (Because staked Amount is also stored in the deposit request structure). And once 'totalDeposits is increased, the user will also be able to finalize stake `2`.

Post-audit 2:
A validation was added: in case 'lastFinalised Stake [user] > 0, stakeId to finalize should be equal to 'lastFinalised Stake[user] + 1`. Yet, there is still a case, when the user can finalize the stake with the ID `0`, then stake with the ID `2` and won't be able to finalize stake with the ID `1`. Also, the user can finalize any stake at the very first time, thus not start with the stake `0`.

Post-audit 3:
Stakes can now be finalized only in the correct order.

### Possible Denial-of-Service in migration functionality.
**Description**

Perpetual TokensFarm.sol: function migrateUserStakes(), line 539.
Perpetual TokensFarmSDK.sol: function migrate UserStakes(), line 538.
There is a "require" statement which validates that the epoch ID of the user's stake strictly equals to epochId` - 1. Yet, there might be a case when the user has a stake, whose epoch ID is lower than 'epochId' 1. Thus, migration would be blocked until the user withdraws such a stake. The issue was found during the third iteration of the audit in the newly added migration functionality.

**Recommendation**

Consider changing "require" to "if" so that migration is not blocked in such a scenario.

**Re-audit comment**

Resolved

## Low Risk

### Unnecessary validation in `finaliseDeposit`.
**Description**

Perpetual TokensFarm.sol: function finaliseDeposit(), line 1156.
TokensFarm.sol: function finalise Deposit(), line 1058.
Perpetual TokensFarmSDK.sol: function finalise Deposit(), line 1076.
In Perpetual TokensFarm.sol and TokensFarm.sol, the 'if' statement will never return false since if the caller is not the owner, the transaction will revert to the 'onlyOwner modifier. In the Perpetual TokensFarmSDK.sol contract, the function can be executed either by the owner or the contract admin. In case the function is called by the contract admin, the value of the local variable won't be assigned to the_user parameter and will be equal to msg.sender (which is the contract admin).

**Recommendation**

Remove the unnecessary validation.

**Re-audit comment**

Resolved.

Post-audit:
In Perpetual Tokens FarmSDK.sol, the validation was removed. In other contracts, functions can now be called by the user, so the validation is necessary now.

### Unused internal functions in factory contracts.
**Description**

TokensFarm Factory.sol, TokensFarmSDKFactory.sol: functions_getFarmArray(),
_getFarmImplementation(). Functions are internal and are not used within the contract. However, they increase the size of the contract.

**Recommendation**

Remove unused functions.

**Re-audit comment**

Verified.

Post-audit.
Functions will be used in the future updates of the contracts.

### Reduce without stake ID might revert in case not the first stake was reduced with ID.
**Description**

TokensFarmSDK.sol, Perpetual Tokens FarmSDK.sol: function notice Reduced StakeWithout StakeId().
In case the user has more than one stake and withdraws any stake but the first one, lastStakeConsumed[_user] will be equal to this stake. When the user decides to withdraw stakes using the notice Reduced StakeWithout StakeId() function, it might revert in line 1463 since all the stakes before 'lastStakeConsumed [_user] will be skipped. The issue is marked as low since the user can still withdraw their stakes separately with the notice Reduced Stake() function.

**Recommendation**

Make sure that notice Reduced StakeWithout StakeId() processes all actual stakes.

**Re-audit comment**

Resolved.

Post-audit.
The validation was added: 'stakeId' is less or equal to 'lastStakeConsumed [_user]` and greater or equal to 'lastStakeConsumed [_user] + 1`. However, there are cases now when the user cannot withdraw all of their stake. For example:
1) The user has 4 stakes.
2) The user withdraws a part of stake '0'.
3) The user withdraws their stake `1`.
4) The user withdraws their stakes `2` and `3` without stake ID.
5) The user can't withdraw the rest of stake `0` due to "Must consume the next stake, can not skip".

Post-audit 2.
It is now verified that stakes can be reduced only in the correct order.

### Enormous gas spendings in migration functionality.
**Description**

Perpetual TokensFarm.sol, Perpetual TokensFarmSDK.sol: function migrateUserStakes().
The lastStakeMigrated storage variable is written on every step of the double cycle, which provides enormous gas spendings. Also, it is especially crucial since the variable is not used in the contracts at all. The issue was found during the third iteration of the audit in the newly added migration functionality.

**Recommendation**

Review the logic of the lastStakeMigrated usage, add conditions to provide storage updates just once for the last ID. E.g., consider adding a couple of conditions to write only the last ID in the double cycle, or consider moving the creation of the storage pointer for StakeInfo out of the cycle.

**Re-audit comment**

Resolved.

Post-audit.
The variable was removed from the contracts.

## Informational

### Implicit variable visibility for `idInList`.
**Description**

Perpetual TokensFarm.sol: `idInList`.
Perpetual TokensFarmSDK.sol: `idInList`.
TokensFarm.sol: idInList`.
TokensFarmSDK.sol: `idInList`.
For better code readability, it is recommended to explicitly mark the visibility for all storage variables and constants.

**Recommendation**

Mark the visibility of all variables and constants in the contracts.

**Re-audit comment**

Resolved

### Use of magic numbers instead of storage constants.
**Description**

Perpetual Tokens Farm.sol: lines 244, 245, 535, 536, 637, 654, 1066, 1562.
Perpetual TokensFarmSDK.sol: lines 240, 535, 616, 1559, 1387.
TokensFarm.sol: lines 233, 234, 525, 545, 987, 1188, 1558.
TokensFarmSDK.sol: lines 243, 530, 1588, 1396.
Number 40 and 100 should be moved to a separate storage constant.

**Recommendation**

Move the numbers used in the code to storage constants.

**Re-audit comment**

Verified.

From the client:
In order not to exceed the contract size limit, constants won't be used.

### Unnecessary addition of zero `warmupPeriod`.
**Description**

TokensFarm.sol: function deposit(), line 1214.
TokensFarmSDK.sol: function deposit(), line 1159.
Adding 'warmupPeriod in both cases has no effect since it was previously checked that `warmupPeriod' is equal to 0.

**Recommendation**

Remove adding 'warmup Period`.

**Re-audit comment**

Resolved

### Finalizing pending deposits can be blocked by changing `warmupPeriod`.
**Description**

TokensFarm.sol: function finalise Deposit(), line 1063.
TokensFarmSDK.sol: function finalise Deposit(), line 1026.
Perpetual Tokens FarmSDK.sol: function finalise Deposit(), line 1081.
Perpetual TokensFarm.sol: function finalise Deposit(), line 1174.
There is a validation that 'warmupPeriod is not equal to 0 in these functions. However, in case the owner sets 'warmupPeriod as with the setWarmup() function, all pending deposit requests will be blocked, preventing users from depositing and withdrawing their funds. The issue is marked as informational since only the owner can change warmupPeriod'.

**Recommendation**

Verify that users' funds can't get blocked due to the changes of the warmup period.

**Re-audit comment**

Resolved

### Inefficient iteration in `getAllPendingStakes`.
**Description**

TokensFarm.sol: function getAllPendingStakes(), line 690.
TokensFarmSDK.sol: function getAllPendingStakes(), line 649.
Perpetual TokensFarm.sol: function getAllPendingStakes(), line 777.
The function performs iteration through the whole participants array, which may consume more gas than allowed per transaction. In order to reduce gas spendings, the 'waiting List array can be used, which already contains all the users who have current deposit requests.

**Recommendation**

Use the waitingList array instead of 'participants`.

**Re-audit comment**

Resolved

### `totalRewards` miscalculation during reward redistribution.
**Description**

TokensFarm.sol: function withdraw(), line 1425.
TokensFarmSDK.sol: function_payoutRewardsAndUpdateState(), line 1246.
Perpetual TokensFarm.sol: function withdraw(), line 1437.
Perpetual TokensFarmSDK.sol: function_payout RewardsAndUpdateState(), line 1245.
In case user's pending reward is to be redistributed, the_fundInternal() function is called, which increases the 'totalRewards storage variable. However, the actual reward balance doesn't increase since the same reward token is funded. This way, there can be a situation when there are not enough rewards to pay to users. The issue is marked as informational since it doesn't prevent the withdrawal of stakes but can prevent collecting rewards for users.

**Recommendation**

Update totalRewards correctly in case of the redistribution of pending rewards.

**Re-audit comment**

Verified.

From the client:
The totalRewards variable doesn't affect any calculations. It is an intended functionality to increase this variable during every redistribution.

### Unnecessary `start` parameter in `migrateUserStakes`.
**Description**

Perpetual TokensFarm.sol, Perpetual TokensFarmSDK.sol: function migrateUserStakes().
Providing this parameter is unnecessary since it must always be equal to the storage variable 'lastUserMigrated . In order to simplify the execution of this function, it is recommended to remove this parameter. The issue was found during the third iteration of the audit in the newly added migration functionality.

**Recommendation**

Remove the 'start' parameter and use 'lastUserMigrated instead.

**Re-audit comment**

Verified.

From the client:
The Tokensfarm team prefers current functionality because the backend provides the start and end so it will be easier for BE to determine the failure in case that tx reverts.

### Owner can withdraw reward tokens using `withdrawTokensIfStuck`.
**Description**

Perpetual TokensFarm.sol, TokensFarm.sol: function withdraw TokensIfStuck().
There is no validation that provided _erc20' is not equal to erc20` (Reward token). Thus, in case the admin has withdrawn reward tokens, there might be not enough rewards for users to pay out. The issue is marked as informational since stakes can still be withdrawn with emergency Withdraw() function, though the capability of the admin to withdraw rewards should still be mentioned in the report. The issue was found during the third iteration of the audit in the newly added migration functionality.

**Recommendation**

Restrict the admin from withdrawing reward tokens.

**Re-audit comment**

Verified.

From the client:
This function is meant to be for the reward token also because there is a case where it can be funded more than needed, so it also stays like that.

### New epoch can be started during ongoing migration.
**Description**

Perpetual TokensFarm.sol, Perpetual TokensFarmSDK.sol. It is possible for the owner to start a new epoch while the migration is in progress. In this case, the previous migration won't be finished correctly and the new migration might not be started in the future. The issue is marked as informational since only the owner can execute these functions, and in this scenario, users will still be able to withdraw their stakes. The issue was found during the third iteration of the audit in the newly added migration functionality.

**Recommendation**

Verify that a new epoch can't be started while there is a migration in progress.

**Re-audit comment**

Resolved

### Pool updated twice and extra operations performed in PerpetualTokensFarmSDK `migrateUserStakes`.
**Description**

Perpetual Tokens FarmSDK.sol, migrateUserStakes()
The pool is updated directly in the function and through the call of _payoutRewardsAndUpdateState() (with the_amount $=0$ parameter). That is unnecessary gas spending.
The same applies to all math operations in_payoutRewardsAndUpdateState() (in the if(_unStake) statement), where a lot of operations are performed with re writing storage after the operations with 0_amount. The issue was found during the third iteration of the audit in the newly added migration functionality.

**Recommendation**

Review the logic of the pool update if there is no side effects of double pool updates and if it is ok for additional gas spending.

**Re-audit comment**

Verified.

Post-audit
1) It was confirmed with the TokensFarm team that the fix does not affect methods with _payoutRewardsAndUpdateState() calls.
2) The necessity of the updatePool() within the double migration cycle was confirmed.
3) The correctness of the logic was confirmed for the Perpetual TokensFarm.sol contract.
