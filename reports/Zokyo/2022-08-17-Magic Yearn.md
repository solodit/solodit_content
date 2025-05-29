**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Stake begin time updated before reward calculation in MyStaking.
**Description**

MyStaking.sol: function_stake(). "stakeBegin" is used within the function getDivisorByTime() during the calculation of the claimable amount. Since_setTimer() is called before rewards are calculated (Lines 243-244), "stakeBegin" will be equal for msg.sender and the commission for the user will always be calculated with maximum fee.

**Recommendation**

Update "stakeBegin" after rewards are calculated.

**Re-audit comment**

Verified.

From the client:

The calculated rewards in_stake() are saved in userInfo[msg.sender].unclaimedAmount without the fee subtracted, so the user does not lose funds but the "fee-advantage".

### Incorrect storage of user rewards during staking in MyStaking.
**Description**

MyStaking.sol: function_stake(). During staking, the rewards are calculated with function claimableAmount(). This function calculates current accrued rewards and adds "userInfo[user].unclaimedAmount" to the claimable amount. After claimable rewards are calculated, they are added to "userInfo[msg.sender].unclaimedAmount" (Line 245), though unclaimed amount is already calculated. This way, malicious users can illegally increase their rewards.

**Recommendation**

Store rewards correctly so that there can't be any extra rewards during staking.

**Re-audit comment**

Resolved

### User funds lock in MyStaking due to stakerCount manipulation.
**Description**

MyStaking.sol: function_withdraw(). When the user stakes, there is the increase of the counter 'stakerCount on line 227. After that, when the user redeems their funds, there is the subtraction of the counter in the function_withdraw() on line 319. However, there is no check that the user is a staker. Due to this, the user can call redeem() as many times as they like, thereby resetting the counter stakerCount` and blocking withdrawals for all users.

**Recommendation**

Add the validation that user's share is greater than 0, thus, verifying that the user is actually a staker.

**Re-audit comment**

Resolved

### Fee can be applied multiple times to same rewards in MyStaking.
**Description**

MyStaking.sol: function claimableAmount(). The function calculates user's rewards and subtracts fees. However, the claimable amount is firstly calculated with the unclaimed amount and then a fee is applied to it. Since an unclaimed amount is stored with an already subtracted fee, the commission can be applied to user's rewards more than once, causing users to lose funds.

**Recommendation**

Apply the fee only to rewards that don't include unclaimed amount.

**Re-audit comment**

Verified.

From the client:

The amount without fees is already stored to unclaimed amount so that fees are applied only one time during claiming.

### Deprecated ETH transfer usage.
**Description**

MyStaking.sol: function daoWithdraw().
MyShare.sol: function withdraw Token().
MyExchange.sol: function withdraw().
MyWrapper.sol: Lines 201, 345, 463, 504, 530.
StakingFactory.sol: function withdraw Token().
Due to the Istanbul update, there were several changes provided to the EVM, which made .transfer() and .send() methods deprecated for the ETH transfer. Thus, it is highly recommended to use .call() functionality with mandatory result check or the built-in functionality of the Address contract from OpenZeppelin library. Also, the contract has neither payable function nor receive() function implemented, thus it isn't supposed to store ETH on its balance.

**Recommendation**

Correct the ETH sending functionality and verify its necessity in the contract.

**Re-audit comment**

Resolved

### Liquidity pool can be set multiple times in MyShare.
**Description**

MyShare.sol: function setLP(). In the comment section, it is declared that a liquidity pool can be set only once, and the "txnLiqPoolIsSet" variable is checked to be false to verify this. However, the stake of this variable doesn't change, meaning that it will always be equal to false and could be set more than once.

**Recommendation**

Assign true to "txnLiqPoolIsSet" after a liquidity pool is set.

**Re-audit comment**

Resolved.

Post-audit:

In order to validate that a pool can only be set once, the "txnLiqPool" storage variable is checked not to be zero address.

### Transfer to zero address might revert in MyStaking.
**Description**

MyStaking.sol: function_withdraw(), line 314. Transferring fees to zero address might revert, depending on the implementation of ERC20. For example, standard implementation by OpenZeppelin has such a restriction. As a result, with some of the ERC20 tokens, withdrawal won't be possible. Thessue is not marked as critical since a custom implementation of ERC20 can be used, which allows transfers to zero address.

**Recommendation**

Verify that the team will only use a custom implementation of ERC20 that allows transferring to zero address. Otherwise, transfer fees to the other address. For example, to address(1).

**Re-audit comment**

Resolved.

Post-audit:

The issue was resolved by transferring fees to address(1).

## Medium Risk

### Standard ERC20 transfer methods used instead of SafeERC20.
**Description**

Transferring ofERC20 tokens across all the contrracts is performed with regular transfer() and transferFrom() methods from the OpenZeppeling IERC20 interface. Yet, in general, the ERC20 token may have no return value for these methods (see USDT implementation) and lead to the failure of calls and contract blocking. Thus, the SafeERC20 library should be used or the token should be verified to implement return values for transfer() and transferFrom() methods.

**Recommendation**

Use the SafeERC20 library for transferring ERC20 tokens.

**Re-audit comment**

Verified.

Post-audit:

SafeERC20 is used in all the necessary parts of the code.

### Variable written twice in MyShare endOfEpoch.
**Description**

MyShare.sol: function endOfEpoch(). The "endTime" variable is written either in line 320 or in line 322 first time. After that it is written again in line 325.

**Recommendation**

Verify the correctness of the function so that the value of "endTime" is calculated correctly.

**Re-audit comment**

Resolved

## Low Risk

### Unused storage variables and constants in MyStaking.
**Description**

MyStaking.sol: variables my Token, peg Token, paired Token, staking Factory, router, zapPath.
MyStaking.sol: constants WETH.

**Recommendation**

Remove unused variables.

**Re-audit comment**

Resolved

### Staker count not calculated correctly in MyStaking.
**Description**

MyStaking.sol: function_stake(), line 237. stakerCount is increased every time anyone stakes tokens, including the cases where msg.sender is already a staker. This way, the variable can contain the wrong number of stakers. The issue is marked as low since stakerCount is not used within the contract and doesn't take part in any calculations. However, it might be used by the dApp.

**Recommendation**

Increase stakerCount only in case msg.sender is not already a staker.

**Re-audit comment**

Resolved

### Route not verified in FeeReducer buyBackBurn.
**Description**

FeeReducer.sol: function buyBackBurn(). The function is supposed to swap TEN token and burn the balance of MyShare token. Thus, the route is supposed to start with TEN token and end with MyShare token and should be validated.

**Recommendation**

Verify that the first element in the route is TEN token and the last one is MyShare token.

**Re-audit comment**

Verified.

From the client:

This feature is intended so that it is possible for the DAO to buy back and burn other tokens than TEN if they were sent to the contract.

### MyShare total supply checked for TEN burn amount in FeeReducer.
**Description**

FeeReducer.sol: function set Ten Burn FixAmount(), line 121. In this function, the total supply of MyShare token is checked instead of the TEN token. However, the function sets a burn amount for the TEN token. The issue is marked as low since it is unclear whether MyShare or TEN token's supply should be checked. Thus, the issue should be verified by the team.

**Recommendation**

Verify which token's supply should be checked and correct the validation in case TEN token's supply should be checked instead of MyShare.

**Re-audit comment**

Resolved

## Informational

### Owner ability to withdraw any ERC20 tokens in MyStaking.
**Description**

MyStaking.sol: function emergency Withdraw(). The owner can withdraw any tokens from the contract, including the stake token. Even though it is declared in the comment section that this function will be removed after the audit, we recommend to remove this function during the audit or restrict the owner from withdrawing the stake token.

**Recommendation**

Remove the function or add a restriction that the owner can't withdraw the stake token.

**Re-audit comment**

Resolved.

Post-audit:

The function was removed.

### Duplicate update of stakeBegin in MyStaking _stake function.
**Description**

MyStaking.sol: function_stake(). The call of the_claim(), function_setTimer() function updates "stakeBegin" variable in user's info (Line 243). The same action is performed in line 248. The issue doesn't affect code security but increases the gas spendings for users.

**Recommendation**

Update "stakeBegin" only once during function_stake().

**Re-audit comment**

Resolved

### Unnecessary validations in MyToken _doTransferFeeless.
**Description**

MyToken.sol: function _doTransferFeeless(). The function_doTransferFeeless() is internal and is called only within the_interTransfer() function. The function performs the same validations as_interTransfer() (Lines 314-315 in _interTransfer() and Lines 375-376 in_doTransferFeeless()), which is unnecessary and only increases gas spendings. Also, the validations that are performed in function_doTransfer() (Lines 336-339) are not performed in _doTransferFeeless().

**Recommendation**

Remove unnecessary validations. Verify that same validations as in_doTransfer() should not be performed in _doTransferFeeless().

**Re-audit comment**

Resolved.

Post-audit:

Unnecessary validations were removed from function_doTransferFeeless().

### Gas inefficiency in MyToken addSyncAddr.
**Description**

MyToken.sol: function addSyncAddr(). In order to determine whether the provided "inAddr" parameter is present in the "_syncAddr" array, an iteration through the whole storage array is performed to find inAddr. However, the value from mapping "isLP" can be checked for "inAddr". This way, gas spendings for the function can be significantly decreased.

**Recommendation**

Read value from mapping instead of iterating through the storage array.

**Re-audit comment**

Resolved

### Unused function _mintMinter in MyShare.
**Description**

MyShare.sol: function_mintMinter(), line 501. The function is not used within MyShare contract. Though it is internal and can be used by children contracts, such contracts are not present within the scope of the audit.

**Recommendation**

Remove the unused function or verify that it is needed for children contracts.

**Re-audit comment**

Resolved

### Owner ability to blacklist any account in MyToken.
**Description**

MyToken.sol: function blacklistAccount(). In the comment section it is declared that the owner can blacklist a dead wallet to prevent it from collecting fees. However, it is not checked whether the provided account is active or inactive, which means that the owner can put any account to blacklist. The issue is marked as critical since such functionality allows the owner to withdraw funds from any account.

**Recommendation**

Consider adding a mechanism of verifying whether an account is truly inactive and should be blacklisted.

**Re-audit comment**

Unresolved.

Post-audit:

The issue was verified by the Magic Yearn team as a part of the protocol. It is covered with appropriate role and the team will provide necessary actions to cover all possible back-door usage and active users tracking. Yet, this issue should be still placed in the report, so it was marked as Informational.

### Additional code optimizations.
**Description**

The following optimizations were discovered during unit-testing and should be considered as suggestions:

MyShare.sol: function_checkTransfer(), line 784.

The "amount" parameter should be validated not to be 0.

MyShare.sol: function_burn(), line 850.

Validating that parameter "account" is not a zero address is unnecessary, since the_burn() function is only called within functions_transfer() and burn(). In the_transfer() function, such validation is already performed in line 689. In the burn() function, the address of the message sender is taken, which cannot be zero.

The wrong revert messages in functions setMinterActivity() (Line 212), removeMinter() (Line 242), setIsBlacklisted() (Line 106). In line 212, the revert message is about adding a minter, though the function sets the activity of the minter. The same is in the function that removes a minter. In line 106, the revert message is about whitelist, whereas the function sets value to blacklist.

The remove Minter() function lacks validation that the provided "minter_" parameter is actually a minter.

MyExchange.sol: function migrateable(). The function can never reach the code in the "else" branch, since "migrated" can't be greater than "MAX_MIGRATE" and in the case when "migrated" is equal to "MAX_MIGRATE", it will return 0 as a result of subtraction in line 64.

**Recommendation**

1. MyShare.sol, _checkTransfer(): Add validation that "amount" is not 0.
2. MyShare.sol, _burn(): Remove unnecessary zero address check for "account".
3. Correct revert messages in MyShare.sol: setMinterActivity(), removeMinter(), setIsBlacklisted().
4. MyShare.sol, removeMinter(): Add validation that "minter_" is an actual minter.
5. MyExchange.sol, migrateable(): Remove the "else" branch as it's unreachable.

**Re-audit comment**

Resolved
