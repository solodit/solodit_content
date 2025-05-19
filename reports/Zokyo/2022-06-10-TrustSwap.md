**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### SafeERC20 should be used.

**Description**

 SwapToken is used with regular transfer() and transferFrom() methods from the 
OpenZeppeling IERC20 interface. Though, in general, the ERC20 token may have not return 
value for these methods (see USDT implementation) and lead to the failure of calls and 
contract blocking. Thus the SafeERC20 library should be used or the token should be 
verified to implement to implement return values for transfer() and transferFrom() methods

 **Recommendation**:
 
 Verify the swapToken used to implement the OpenZeppelin version of IERC20 interface or 
use SafeERC20 library.

 **From client**:
 
 Only swap tokens would be used, which interface returns true or false

### Iteration through storage array.

 **Description**

 LockedStaking.sol: getRewardsWeight(), line 362. 
 Iteration through a storage array might consume more gas than allowed per transaction, 
thus function will always revert, preventing the protocol from correct operation. Issue is 
marked as medium, because only the owner can add reward tokens and control an amount 
of elements in an array.

**Recommendation**:

 Either add an ability to remove expired rewards or add a maximum limit of rewards, which 
can be added to array “rewards”.

 **Post audit**: 
 
Maximum limit was added

## Low Risk

### Users are able to create an unlimited amount of locks.

 **Description**

 LockedStaking.sol: function addLock(). 
 There is no limitation on the amount of locks for each user. Each lock is added to the user's 
array of locks. In case user has created too many locks, this can cause to revert when 
iterating through all user’s locks (function getUserClaimable(): Line 256, function 
getUsersTotalScore(): Line 273) and prevent user from withdrawing his rewards or locked 
tokens. Issue is marked as low, since there is no impact to overall protocol and other users.
 
 **Recommendation**:
 
 Either add a limitation on amount of locks per user, or add a minimum boundary for locked 
amount when creating locks(So that user is not able to add a lot of lock with low dusting 
locked amount(like several weis for example)).

 **Post audit**:
 
 Maximum limit was added

## Informational

### Variables lack validation.

  **Description**

 LockedStaking.sol: function addReward(). 
 Time-dependent parameters should be validated. It is a common practice to verify that 
time-dependent parameters are greater than the current block’s timestamp and that the 
“end” parameter is greater than the “start” parameter of any time frame. Although, the 
correctness of “start” and “end” variables are verified indirectly in line 103 (when subtract 
start from end), such error might be unclear. And also it still leaves room for the adding 
rewards with the timeline in the past - and in such way it will be left unnoticeable for the 
rewards weight update. 
 Variable “amountPerSecond” should be validated not to be zero. It may be validated 
indirectly in the IERC.transferFrom() call (through 0 token transfer validation), but such a 
check may be missing for the token. Issue is marked as info since only the owner is able to 
call this function and also rewards can be updated with function updateReward().
 
 **Recommendation**:
 
 Validate that parameter “start” is greater than block.timestamp, “end” is greater than “start” 
and “amountPerSecond” is greater than 0

### Public functions can be marked as external.

  **Description**

 Functions initialize(), getRewardsLength(), getLockInfo(), getUserLocks(), getLockLength(), 
getRewards(), addReward(), updateReward(), addLock(), compound(), updateLockAmount(), 
updateLockDuration(), getUserClaimable(), claim(), unlock(), getUnlockedRewards() can be 
marked external, since they are not called within other functions of the contract. Usage of 
external functions decrease gas spendings compared to public functions.
 
 **Recommendation**:
 
 Mark the following functions as external

### getUserClaimable() can be simplified.
  
   **Description**

 getUserClaimable() function can be simplified with the usage of internal 
calculateUserClaimable() function.

 **Recommendation**:
 
 The function can be simplified

### Undocumented logic.

   **Description**

 addLock() function creates new lock, though it is undocumented, that this function also 
provides lock of the claimable rewards (if the user has such). Neither function name nor 
parameters give info about this approach. It will be better for the code quality to add the 
natspec about such behavior or/and add a scheme to the documentation of the 
“autocompounding” approach
 
 **Recommendation**:
 
 Increase documentation quality about the “autocompounding” approach
