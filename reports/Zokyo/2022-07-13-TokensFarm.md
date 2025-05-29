**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### No money back.
**Description**

TokensFarm.sol / Perpetual TokensFarm, function deposit / makeDepositRequest

If the farm contracts allow commission via eth (isFlatFeeAllowed), and the user sends eth more than the required limit, the rest will not be returned to him.

**Recommendation**

Add the processing of this case, returning the balance of the necessary funds to the user. Be careful and consider the reentrancy vulnerability.

**Re-audit comment**

Resolved.

From client:

>= was replaced with ==

## Medium Risk

### Iteration through the whole storage array.
**Description**

TokensFarm.sol: function_remove Participant(), line 419.

Perpetual TokensFarm.sol: function_removeParticipant(), line 332.

When a user performs withdrawal,_removeParticipant() is called to verify whether the user should be removed from participants. An iteration through all user's stakes is performed to determine this. In case, user has too many stakes, iteration will consume more gas than allowed per one transaction. Also, fully withdrawn stakes are not removed from the array, which also increases gas spendings.

**Recommendation**

Remove stakes, which are fully withdrawn. Consider adding a limitation on the number of stakes, allowed per one user. Or add a mapping with the amount of active staking.

**Re-audit comment**

Resolved

## Low Risk

### Zero amount.
**Description**

TokensFarm Factory.sol, function withdraw TokenslfStuck On Farm, _fundInternalFarm
In this functions you have that requirement.

require(

);
_amount $>=0$ 

"Amount must be over or equal to 0"

But after that, there are calls for approval and transfer. It turns out that it is possible to call a transfer/approval with a zero value of amount.

**Recommendation**

Replace >= with >

**Re-audit comment**

Resolved

### User can deposit zero amount and become participant.
**Description**

TokensFarm.sol: function deposit().

Perpetual TokensFarm.sol: function deposit().
Function has no validation, that deposited amount is greater than zero, thus, any user is able to create an invalid stake and become a participant of the staking. Although the issue doesn't affect rewards distribution, a malicious user is able to become a participant of the staking with multiple accounts and flood the contract with invalid participants.

**Recommendation**

Add a validation that the deposited amount is greater than 0.

**Re-audit comment**

Resolved

### Accuracy loss when adding funds to farm.
**Description**

TokensFarm.sol: function _fundInternal(), line 321.

Perpetual TokensFarm.sol: function _fundInternal(), line 436.

In the function _fundInternal() in order to calculate the additional time, the farm will be on, _amount is divided by rewardPerSecond. In case, _amount is lower than reward PerSecond, the result of the division will be zero, meaning that the endTime won't be changed, while storage total Funded Rewards and totalRewards variables will be increased. Issue is marked as low, since it doesn't affect reward distribution, and the lost funds would be low (lower than reward PerSecond).

**Recommendation**

Either improve the calculations precision or add a restriction, that_amount should be greater than rewardPerSecond.

**Re-audit comment**

Resolved.

From client:

resolved as recommended

## Informational

### Useless payable modifier.
**Description**

TokensFarm.sol, function makeWithdrawRequest

Function with payable modifier, but eth is not used anywhere in the function.

**Recommendation**

Remove the modifier or add the necessary processing logic.

**Re-audit comment**

Resolved

### Optional variable usage.
**Description**

In files: MerklelterativeVesting.sol, MerkleLinearVesting.sol variable noOfUsers not used.
In files: IterativeVesting Farm.sol, LinearVesting Farm.sol, Perpetual TokensFarm.sol, TokensFarm.sol you can use participants.length instead of noOfUser

**Recommendation**

Delete and replace the variable with participants.length. You can use a separate getter.

**Re-audit comment**

Verified.

From client:

TokensFarm team uses noOfUsers for BE and FE components

### Token burning to address(1).
**Description**

TokensFarm.sol, function withdraw

1300: _erc20Transfer(address(1), pendingAmount);
When a user pays a fine, the reward tokens are irretrievably destroyed, because they send to the address that you don't. Burn functionality needs verification from the team

**Recommendation**

Confirm or correct this logic.

**Re-audit comment**

Verified.

From client:

TokensFarm team confirms that this is correct way to burn the tokens

### Internal or never used functions.
**Description**

TokensFarm Factory._getFarmArray(uint256),

TokensFarm Factory._getFarmImplementation(uint256)

Vesting Farm Factory._getFarmImplementation(uint256)

These functions have an internal modifier and are not used anywhere.

**Recommendation**

Change the modifier or remove functions if you don't intend to inherit from these contracts.

**Re-audit comment**

Verified.

From client:

TokensFarm team confirms that these functions are meant for further usage so it is okay to have them.

### Unnecessary uint non-negativity checks.
**Description**

The checks below don't make sense for the $>=0$ comparison, as how the variables being tested are of type uint - which is unsigned and has a minimum value of 0.

- require(bool, string)(_earlyClaimAvailablePer >= 0 && _earlyClaimAvailablePer <= 100, Claim percentage should be above 0 and below 100) (contracts/IterativeVesting Farm.sol#158-161)
- require(bool, string)(_beforeStartClaimPer $>=0$ && _beforeStartClaimPer < $\angle=100$ Claim percentage should be above 0 and below 100) (contracts/Iterative Vesting Farm.sol#162-165)
- amountEarned $>=0$ (contracts/IterativeVestingFarm.sol#702)
- require(bool, string)(_earlyClaimAvailablePer $>=0$ && _earlyClaimAvailablePer $<=100,Claim$ percentage should be above 0 and below 100) (contracts/LinearVesting Farm.sol#149-152)
- require(bool, string)(_beforeStartClaimPer $>=0$ && _beforeStartClaimPer < $<=100,$ Claim percentage should be above 0 and below 100) (contracts/LinearVesting Farm.sol#153-156)
- amountEarned $>=0$ (contracts/LinearVesting Farm.sol#653)
- require(bool, string)(_earlyClaimAvailablePer $>=0$ && _earlyClaimAvailablePer $<=100$,Claim percentage should be above 0 and below 100) (contracts/Merklelterative Vesting.sol#162-165)
- require(bool, string)(_beforeStartClaimPer $>=0$ && _beforeStartClaimPer <= 100, Claim percentage should be above 0 and below 100) (contracts/Merklelterative Vesting.sol#166-169)
- require(bool, string)(additionalFunds $>=0$ Total budget can not be below 0) (contracts/ MerklelterativeVesting.sol#284-287)
amountEarned $>=0$ (contracts/MerklelterativeVesting.sol#913)
- require(bool, string)(_earlyClaimAvailablePer $>=0\&\&$ _earlyClaimAvailablePer $<=100,$ Claim percentage should be above 0 and below 100) (contracts/Merkle LinearVesting.sol#151-154)
- require(bool, string)(_beforeStartClaimPer $>=0$ && _beforeStartClaimPer $<=100$ Claim percentage should be above 0 and below 100) (contracts/Merkle LinearVesting.sol#155-158)
- require(bool, string)(additionalFunds $>=0$, Total budget can not be below 0) (contracts/ Merkle LinearVesting.sol#244-247)
- amountEarned $>=0$ (contracts/MerkleLinearVesting.sol#853)
-r - require(bool, string)(_flatFeeAmountWithdraw $>=0$,_flatFeeAmountWithdraw can not be below 0) (contracts/PerpetualTokensFarm.sol#506-509)
- require(bool, string)(_stakeFeePercent $>=0$ && _stakeFeePercent $<=100$,_stakeFeePercent can not be below 0) (contracts/Perpetual TokensFarm.sol#494-497)
- require(bool, string)(_rewardFeePercent $>=0\&8$ _rewardFeePercent <= 100,_reward FeePercent can not be below 0) (contracts/Perpetual TokensFarm.sol#498-501)
- require(bool, string)(_flatFeeAmountDeposit $>=0$,_flatFeeAmountDeposit can not be below 0) (contracts/Perpetual TokensFarm.sol#502-505)
- require(bool, string)(_minTimeToStake $>=0$,_minTimeToStake can not be below 0) (contracts/ Perpetual TokensFarm.sol#490-493)
- require(bool, string)(_minTimeToStake $>=0_{1\_}$ _minTimeToStake can not be below 0) (contracts/ Perpetual TokensFarm.sol#559-562)
- require(bool, string)(_stakeFeePercent $>=0$ && _stakeFeePercent $<=100$,_stakeFeePercent can not be below 0) (contracts/Perpetual TokensFarm.sol#594-597)
- require(bool, string)(_rewardFeePercent $>=0$ && _rewardFeePercent <= 100,_reward FeePercent can not be below 0) (contracts/Perpetual Tokens Farm.sol#614-617)
- require(bool, string)(_flatFeeAmount $>=0$,_flatFeeAmount can not be below 0) (contracts/ Perpetual TokensFarm.sol#635-638)
- require(bool, string)(_flatFeeAmount $>=0$,_flatFeeAmount can not be below 0) (contracts/ Perpetual TokensFarm.sol#655-658)
- require(bool, string)(_warmup $>=0$ Warmup needs to be over or equal to 0) (contracts/ TokensFarm.sol#269-272)
require(bool, string)(_coolDown $>=0$, CoolDown needs to be over or equal to 0) (contracts/ TokensFarm.sol#273-276)
require(bool, string)(_minTimeToStake $>=0,$ _minTimeToStake can not be below 0) (contracts/ TokensFarm.sol#493-496)
- require(bool, string)(_stakeFeePercent $>=0\&8$ _stakeFeePercent $<=100,$ _stakeFeePercent can not be below 0) (contracts/TokensFarm.sol#528-531)
- require(bool, string)(_rewardFeePercent >= 0 && _rewardFeePercent <= 100,_rewardFeePercent can not be below 0) (contracts/TokensFarm.sol#548-551)
- require(bool, string)(_flatFeeAmount $>=0$,_flatFeeAmount can not be below 0) (contracts/ TokensFarm.sol#569-572)
- require(bool, string)(_flatFeeAmount $>=0,$ _flatFeeAmount can not be below 0) (contracts/ TokensFarm.sol#589-592)
- require(bool, string)(_coolDown $>=0$,_coolDown can not be below 0) (contracts/ TokensFarm.sol#624-627)
- require(bool, string)(_warmup $>=0$,_warmup can not 0 or below) (contracts/ TokensFarm.sol#644-647)
- (_farmIndex >= 0 && _farmIndex <= uint8(Farm Types. Perpetual Farm)) (contracts/ TokensFarm Factory.sol#199)
- require(bool, string)(_amount $>=0$, Amount must be over or equal to 0) (contracts/ TokensFarm Factory.sol#271-274)
- require(bool, string)(_amount $>=0$, amount must be over or equal to 0) (contracts/ TokensFarm Factory.sol#926-929)
- require(bool, string)(start $>=0$ && end <= allDeployed FarmsByTypes [farmType].length, One of the index is out of range) (contracts/TokensFarm Factory.sol#1014-1017)
- require(bool, string)(amount $>=0$ Amount needs to be over or equal to 0) (contracts/ Tokens Farm Factory.sol#1051-1054)
- (_farmIndex >= 0 && _farmIndex <= uint8(Farm Types. MERKLE_ITERATIVE)) (contracts/ Vesting Farm Factory.sol#168-171)
- require(bool, string)(_amount $>=0$,Amount must be over or equal to 0) (contracts/ Vesting Farm Factory.sol#243-246)
- require(bool, string)(amount $>=0$, Amount needs to be over or equal to 0) (contracts/ Vesting Farm Factory.sol#858-861)
- require(bool, string)(start >= 0 && end <= deployed Farms.length, One of the index is out of range) (contracts/Vesting Farm Factory.sol#1050-1053)

**Recommendation**

Remove the unnecessary part of the check.

**Re-audit comment**

Verified.

From client:

The team is acknowledged of extra checks and that it will automatically be reverted if somebody passed to contract negative number because it is uint256. The team has left checks to determine what was the error, and get the description

### Owner withdrawal of vested tokens.
**Description**

IterativeVesting Farm.sol, LinearVesting Farm.sol, MerklelterativeVesting.sol, MerkleLinearVesting.sol: function emergencyAssetsWithdrawal().

Owner is able to withdraw vestedToken, thus, preventing users from receiving the funds.

**Recommendation**

Add a validation that vestedToken cannot be withdrawn. In case, it is required to withdraw any exceeded amount of vested Token, add a corresponding validation.

**Re-audit comment**

Verified.

From client:

After the consultation with The TokensFarm team it was verified that emergencyAssetsWithdrawal can be called only from factory calling emergencyAssetsWithdrawalOnSpecificVestingFarm and this function can be only called by Congress(multi sig wallet). And this function is meant to be called in case of emergency to withdraw whole balance of farm to our congress.

### Index out of range check order.
**Description**

Tokens Farm Factory.sol function getDeployed Farms

FarmTypes farmType = FarmTypes(_farmIndex);

1007:

1008:

1009:

require(

1010:

isEnumInRange(_farmIndex),

1011:

"Farm index is out of range"

1012: );
In this function, you first take the desired type, and only after that you check if it falls within the bounds. The issue has info status, sunce it will fail anyway, but without the reasonable error message.

**Recommendation**

Raise the check above taking the type.

**Re-audit comment**

Unresolved
