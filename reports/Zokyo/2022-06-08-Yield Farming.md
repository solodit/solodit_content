**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### In contract UnifarmNFTManagerUpgradeable, function stakeOnUnifarm, variable farmToken it’s an user input that represents an address that it is not sanitized before, this lead to re- entracy attacks and unpredictable behaviours.

**Recommendation**:

Whitelist the farmToken address or ad re-entracy attacks to the stake and burn function and
the other functions that modified the same state data that the stake function does. (Done)

## Low Risk

### Contracts UnifarmCohort, UnifarmNftManagerUpgradeable, etc... are making use of SafeMath for mathematical operations, but since the solidity 0.8.0, the use of SafeMath is not necessary anymore because the revert happens automatically in case of overflow or underflow, so SafeMath is not adding any benefits and it’s just making taking more gas with no addeds benefits.

**Recommendation**:
Remove the use of SafeMath library and the use of it’s methods and replace them with normal
signs calculations. (Done we removed the SafeMath Now)

## Informational

### In contract UnifarmCohort, functions setPortionAmount and disableBooster does not contains sanity checks, even if they are only settable by the owner they still need to contains sanity checks to be in standard with best practices.

**Recommendation**:
Add sanity checks to the setPortionAmount and disableBooster functions. (this was
informational so skipping it)

### In contract UnifarmCohort, function safeWithdrawAll, for a better gas optimization, you can safe the value of tokens.length in memory and iterate over the variable that is safe in memory, it will be cheaper to read from memory every time then to read from storage.

**Recommendation**:

Create a new variable that it will store the value of tokens.length and use it in the while
condition. (Done)

### In contract UnifarmRewardRegistryUpgradeable, function safeWithdrawAll, for a better gas optimization, you can safe the value of tokens.length in memory and iterate over the variable that is safe in memory, it will be cheaper to read from memory every time then to read fromstorage.

**Recommendation**:
Create a new variable that it will store the value of tokens.length and use it in the while
condition. (Done)

### In contract UnifarmRewardRegistryUpgradeable, function setRewardCap, for a better gas optimization, you can save the value of rewardTokenAddresses.length in memory and iterate over the variable that is safe in memory, it will be cheaper to read from memory every time then to read from storage.

**Recommendation**:

Create a new variable that it will store the value of rewardTokenAddresses.length and use it in
the for loop. (Done)

### In contract UnifarmRewardRegistryUpgradeable, function sendMulti, for a better gas optimization, you can save the value of rewardTokens.length in memory and iterate over the variable that is safe in memory, it will be cheaper to read from memory every time then to read from storage.

**Recommendation**:
Create a new variable that it will store the value of rewardTokens.length and use it in the for
loop. (Done)

### In contract UnifarmRewardRegistryUpgradeable, function addInfluencers, for a better gas optimization, you can safe the value of userAddresses.length in memory and iterate over the variable that is safe in memory, it will be cheaper to read from memory every time then to read from storage.

**Recommendation**:
Create a new variable that it will store the value of userAddresses.length and use it in the
while condition. (Done)

### In library ConvertHexToString, function toString, for a better gas optimization, you can save the value of data.length in memory and iterate over the variable that is safe in memory, it will be cheaper to read from memory every time then to read from storage.

**Recommendation**:
Create a new variable that it will store the value of data.length and use it in the for loop.
(Done)

### Contract UnifarmCohort, variable factory is assigned only in the constructor and then the value is used all over the contract, from this behavior we understand that the scope of the factory variable is that it should be assigned only in contract constructor, for that we would recommend making the variable as ‘immutable’, (immutable variables can only be assigned inside the constructor and only there), for extra security and to be in accordance with best practices.

**Recommendation**:
Make variable factory from line 47 as immutable. (Done)

### In contract UnifarmCohort, all the revert messages form the require functions are very short and hard to understand, we understand the reason for this was the gas optimization but this is not necessary, the strings in solidity are defined on 32 bytes, so even if your string message hase only 2 characters in it, it will still take 32 bytes, so our recommendation would be to make the messages more clear and to take in consideration to not make them any longer then 32 bytes, if you can make them explicit and with a length of 32 bytes it will take as many gas as this short messages are taking right now.

**Recommendation**:

Make the messages more explicative but no longer then 32 bytes because they are already
defined on 32 bytes, so there is no added benefit to make them shorter then 32 bytes,
prioritize clarity and explanations. (this was informational so skipping it)

### In contract UnifarmNftManagerUpgradeable, all the revert messages form the require functions are very short and hard to understand, we understand the reason for this was the gas optimization but this is not necessary, the strings in solidity are defined on 32 bytes, so even if your string message hase only 2 characters in it, it will still take 32 bytes, so our recommendation would be to make the messages more clear and to take in consideration to not make them any longer then 32 bytes, if you can make them explicit and with a length of 32 bytes it will take as many gas as this short messages are taking right now.

**Recommendation**:
Make the messages more explicative but no longer then 32 bytes because they are already
defined on 32 bytes, so there is no added benefit to make them shorter then 32 bytes,
prioritize clarity and explanations. (this was informational so skipping it)
