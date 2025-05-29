**Auditors**

[ZOKYO](https://x.com/zokyo_io)

# Findings

## High Risk

### Usage of msg.value in the loop.
**Description**

Perpetual TokensFarmSDK.sol, function notice Reduced StakeWithoutStakeld(), line 1433.
TokensFarmSDK.sol, function notice Reduced Stake WithoutStakeld(), line 1451. The _payoutRewardsAndUpdateState() function is executed multiple times in the loop. There is a call of the_erc20Transfer() function in this function, where msg.value is used and is added to the 'totalFeeCollectedETH` storage variable. This way, the same msg.value is added multiple times. This can potentially prevent collecting ETH fees. Reference: https://github.com/crytic/slither/wiki/Detector-Documentation/#msgvalue-inside-a-loop

**Recommendation**

Avoid using msg.value in the loop. Add msg.value only once to `totalFeeCollectedETH`.

**Re-audit comment**

Resolved

### The value in `idInList` mapping can be wrong.
**Description**

TokensFarm.sol: function finalise Deposit(), line 1103.
TokensFarmSDK.sol: function finalise Deposit(), line 1061. Perpetual TokensFarmSDK.sol: function finaliseDeposit(), line 1114. Perpetual TokensFarm.sol: function finaliseDeposit(), line 1203. In case the user has doesn't have any deposit requests, they get removed from the 'waitingList and their ID in the waiting list is given to the last user in the 'waitingList`. However, the value in the 'idInList' mapping is not updated for 'lastUserInWaitingListArray'. Due to this, the finalization of the request for lastUserInWaiting ListArray can be blocked.

**Recommendation**

Update the value in the 'idInList' mapping for 'lastUserInWaiting ListArray with deletedUserld . Delete the value from the idinList mapping for the user. Take in account that 'lastUserInWaiting ListArray' can be equal to `user` in case there is only one address in the waiting list.

**Re-audit comment**

Resolved

### Users can withdraw the same stake more than once with the emergency withdrawal function.
**Description**

TokensFarm.sol: function emergencyWithdraw().
Perpetual TokensFarm.sol: function emergencyWithdraw(). Users can execute the emergencyWithdraw() function with the same stake, even when the amount of the stake is 0. Due to this, the 'participants' array is updated every time, deleting the first user from the array. This way, users can delete all users from the participants array and block all withdrawal functions to them.

**Recommendation**

Do not let users conduct emergency withdrawal of the same stake more than once.

**Re-audit comment**

Resolved

### Wrong stake amount is stored.
**Description**

Perpetual TokensFarm.sol: function_deposit(), line 1083. The_amount parameter is stored in 'stake.amount'. In case there is a stake fee, a greater amount would be stored instead of the value after taking the fee.

**Recommendation**

Store 'stakedAmount` in `stake.amount`.

**Re-audit comment**

Resolved

## Medium Risk

### User's stake can be reduced before stakes are finalized in SDK contracts.
**Description**

Perpetual TokensFarmSDK.sol
TokensFarmSDK.sol
During the execution of the make DepositRequest() function, the value in the `totalActiveStakeAmount[user] mapping is updated. This value is used in the notice Reduced StakeWithout Stakeld() function to verify that the user has sufficient stake amount. In case this function is called before the finalization of the stake, the user would be able to withdraw their non-finalized stake. Even though there is a validation that the stake is finalized (TokensFarmSDK.sol, line 1442), the issue still can occur in the cases when the user has only one stake that is not yet finalized. The issue is marked as medium since only the owner or the contract admin can execute these functions.

**Recommendation**

Ensure that stakes cannot be reduced or withdrawn via `noticeReducedStakeWithoutStakeld` before they are finalized, or ensure that accounting for `totalActiveStakeAmount` and `totalDeposits` correctly handles unfinalized stakes during such operations.

**Re-audit comment**

Resolved.

Post-audit.
The deposits that are not finalized can still be processed, potentially breaking the totalDeposits variable, and user's totalActiveStakeAmount`. Consider such a scenario:
1) The user has performed 3 different stakes with the following amounts:
a) Stake `0` with amount = 1 token.
b) Stake `1` with amount = 2 token.
c) Stake `2` with amount = 3 token.
2) Stake `0` and `2` are finalized, leaving stake `1` unfinalized.
a) `totalDeposits is equal 1+3=4 (Since only stakes `0` and `2` are finalized.)
3) The user withdraws the amount of 3 tokens with the notice Reduced StakeWithoutStakeld() function. In this case:
a) the amount of stake `0` will be equal 0.
b) the amount of stake `1` (unfinalized) will also be equal 0.
c) the amount of stake `2` will still be equal to 3.
d) `totalDeposits will be equal to 1 (Despite the fact that there is a finalized stake `2` with the amount of 3).
After this, the finalization of stake `1` will increase the 'totalDeposits despite the fact that the amount of stake `1` is equal to 0 (Because stakedAmount is also stored in the deposit request structure). And once totalDeposits is increased, the user will also be able to finalize stake `2`.

Post-audit 2.
A validation was added: in case `lastFinalised Stake[user] > 0, stakeld to finalize should be equal to lastFinalised Stake[user] + 1`. Yet, there is still a case, when the user can finalize the stake with the ID `0`, then stake with the ID `2` and won't be able to finalize stake with the ID `1`. Also, the user can finalize any stake at the very first time, thus not start with the stake `0`.

Post-audit 3.
Stakes can now be finalized only in the correct order.

## Low Risk

### Unnecessary validation in `finaliseDeposit`.
**Description**

Perpetual Tokens Farm.sol: function finalise Deposit(), line 1156.
TokensFarm.sol: function finaliseDeposit(), line 1058.
PerpetualTokensFarmSDK.sol: function finalise Deposit(), line 1076.
In PerpetualTokensFarm.sol and TokensFarm.sol, the `if statement will never return false since if the caller is not the owner, the transaction will revert to the 'onlyOwner modifier. In the Perpetual Tokens FarmSDK.sol contract, the function can be executed either by the owner or the contract admin. In case the function is called by the contract admin, the value of the local variable won't be assigned to the _user parameter and will be equal to msg.sender (which is the contract admin).

**Recommendation**

Remove the unnecessary validation.

**Re-audit comment**

Resolved.

Post-audit.
In PerpetualTokensFarmSDK.sol, the validation was removed. In other contracts, functions can now be called by the user, so the validation is necessary now.

### Unused internal functions in factory contracts.
**Description**

TokensFarm Factory.sol, Tokens FarmSDKFactory.sol: functions_getFarmArray(), _getFarmImplementation(). Functions are internal and are not used within the contract. However, they increase the size of the contract.

**Recommendation**

Remove unused functions.

**Re-audit comment**

Verified.

Post-audit.
Functions will be used in the future updates of the contracts.

### `noticeReducedStakeWithoutStakeld` might revert if not first stake was reduced with ID.
**Description**

TokensFarmSDK.sol, Perpetual Tokens FarmSDK.sol: function notice ReducedStakeWithoutStakeld().
In case the user has more than one stake and withdraws any stake but the first one, lastStakeConsumed[_user] will be equal to this stake. When the user decides to withdraw stakes using the noticeReducedStakeWithoutStakeld() function, it might revert in line 1463 since all the stakes before 'lastStakeConsumed[_user] will be skipped. The issue is marked as low since the user can still withdraw their stakes separately with the notice ReducedStake() function.

**Recommendation**

Make sure that notice Reduced StakeWithoutStakeld() processes all actual stakes.

**Re-audit comment**

Resolved.

Post-audit.
The validation was added: 'stakeld is less or equal to 'lastStakeConsumed[_user] and greater or equal to lastStakeConsumed[_user] + 1`. However, there are cases now when the user cannot withdraw all of their stake. For example:
1) The user has 4 stakes.
2) The user withdraws a part of stake `0`.
3) The user withdraws their stake `1`.
4) The user withdraws their stakes `2` and `3` without stake id.
5) The user can't withdraw the rest of stake `0` due to "Must consume the next stake, can not skip".

Post-audit 2.
It is now verified that stakes can be reduced only in the correct order.

## Informational

### Implicit variable visibility for `idInList`.
**Description**

Perpetual TokensFarm.sol: `idInList`.
PerpetualTokensFarmSDK.sol: `idInList`.
TokensFarm.sol: `idInList`.
Tokens FarmSDK.sol: `idInList`.
For better code readability, it is recommended to explicitly mark visibility for all storage variables and constants.

**Recommendation**

Mark visibility for all variables and constants in the contracts.

**Re-audit comment**

Resolved

### Use of magic numbers instead of storage constants.
**Description**

PerpetualTokensFarm.sol: lines 244, 245, 535, 536, 637, 654, 1066, 1562.
Perpetual Tokens FarmSDK.sol: lines 240, 535, 616, 1559, 1387.
TokensFarm.sol: lines 233, 234, 525, 545, 987, 1188, 1558.
TokensFarmSDK.sol: lines 243, 530, 1588, 1396.
Number 40 and 100 should be moved to a separate storage constant.

**Recommendation**

Move the numbers used in the code to storage constants.

**Re-audit comment**

Verified.

From the client.
In order not to exceed the contract size limit, constants won't be used.

### Unnecessary addition of zero `warmupPeriod`.
**Description**

TokensFarm.sol: function deposit(), line 1214.
TokensFarmSDK.sol: function deposit(), line 1159.
Adding 'warmupPeriod in both cases has no effect since it is previously checked that warmupPeriod` is equal to 0.

**Recommendation**

Remove adding 'warmupPeriod .

**Re-audit comment**

Resolved

### Finalizing pending deposits can be blocked by changing `warmupPeriod`.
**Description**

TokensFarm.sol: function finalise Deposit(), line 1063.
TokensFarmSDK.sol: function finalise Deposit(), line 1026.
Perpetual TokensFarmSDK.sol: function finaliseDeposit(), line 1081.
Perpetual TokensFarm.sol: function finalise Deposit(), line 1174.
There is a validation that 'warmupPeriod is not equal to 0 in these functions. However, in case the owner sets 'warmupPeriod' as 0 with the setWarmup() function, all pending deposit requests will be blocked, preventing users from depositing and withdrawing their funds. The issue is marked as informational since only the owner can change 'warmupPeriod`.

**Recommendation**

Verify that users' funds can't get blocked due to the changes of the warmup period.

**Re-audit comment**

Resolved

### Inefficient iteration in `getAllPendingStakes`.
**Description**

TokensFarm.sol: function getAllPending Stakes(), line 690.
TokensFarmSDK.sol: function getAllPending Stakes(), line 649.
Perpetual TokensFarm.sol: function getAllPendingStakes(), line 777.
The function performs iteration through the whole participants array, which may consume more gas than allowed per transaction. In order to reduce gas spendings, the 'waitingList array can be used, which already contains all the users who have current deposit requests.

**Recommendation**

Use the waitingList array instead of `participants`.

**Re-audit comment**

Resolved

### `totalRewards` miscalculation during reward redistribution.
**Description**

TokensFarm.sol: function withdraw(), line 1425.
TokensFarmSDK.sol: function_payoutRewardsAndUpdateState(), line 1246.
Perpetual TokensFarm.sol: function withdraw(), line 1437.
PerpetualTokensFarmSDK.sol: function_payoutRewardsAndUpdateState(), line 1245.
In case user's pending reward is to be redistributed, the_fundInternal() function is called, which increases the storage variable `totalRewards . However, the actual reward balance doesn't increase since the same reward token is funded. This way, there can be a situation when there are not enough rewards to pay to users. The ssue is marked as informational since it dosn't prevent thewithdrawal of stakes, but it can prevent collecting rewards for users.

**Recommendation**

Update `totalRewards correctly in case of the redistribution of pending rewards.

**Re-audit comment**

Verified.

From the client.
The totalRewards variable doesn't affect any calculations. It is intended functionality to increase this variable during every redistribution.
