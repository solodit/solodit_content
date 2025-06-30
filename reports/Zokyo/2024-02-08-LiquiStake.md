**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Incorrect handling of slippage for rewards collection.
**Description**

StWSX.sol: oracleClaim Rewards(), line 332.
Currently, slippage is calculated as a certain percent from accrued reward and is stored in the array "amountOutMins". However, based on the logic of CompoundStake.sol, values "amountOutMins" are used as minimal amounts of WSX, which will be received through exchanging accrued rewards into WSX via the SharkSwap exchange. Thus, storing values representing the reward token's percentage may lead to reverting the swap since SharkSwap will compare the amountOutMin with WSX, not the reward token. Additionally, calculating slippage inside the smart contract before the swap will not protect against the frontrun attack and may lead to the loss of rewards. That is why the most optimal way of handling slippage is passing it as a function parameter and calculating it on the backend part in advance.

**Recommendation**

Calculate slippage as an amount of WSX expected to be received for the accrued reward amount. Pass an array with slippage values as a function parameter, calculating it on the backend in advance to protect against possible frontrun attacks.

**Re-audit comment**

Resolved.

Post-audit. The reward token list and exchange slippage are now passed as function parameters. According to the LiquiStake team, slippage will be calculated on the backend. Functionality connected to the slippage was removed from the contract.

### Updating of WSX, Staking Proxy or Validator may lead to the loss of funds.
**Description**

StWSX.sol: oracle Claim Rewards(), line 332.
StWSX.sol: setWSXToken(), setStaking Proxy(), setValidatorAddress().
Updating WSX, Staking Proxy, or Validator may cause users whose funds were not unstaked before the update to lose them. Thus, when updating these parameters, users' staked funds may get stuck.

**Recommendation**

Implement a migration mechanism for staked WSX when updating the core parameters.

**Re-audit comment**

Resolved.

Post-audit. Setters were removed so that WSX, Staking Proxy, and Validator parameters can only be set during deployment. The migration will be performed via redemption and deposit to a new contract.

### Inaccurate calculations of fee shares for DAO.
**Description**

StWSX.sol: oracle Claim Rewards().
When rewards are claimed, part of them are allocated to the DAO in the form of shares as protocol fees. However, the fees are not calculated accurately since the whole amount of the reward amount` is added to the 'totalPooledWSX before the calculating and minting of fees. As a result, the DAO will actually receive less when shares are redeemed. And in case rewardAmount is added to the totalPooledWSX after minting, DAO will receive more, cutting user rewards.

**Recommendation**

Before minting shares, add only the reward amount that should be distributed to users (reward Amount - reward FeeAmount) to the 'totalPooledWSX. The rest (rewardFeeAmount) should be added after minting.
Example of the code:
totalPooledWSX += (reward Amount - reward FeeAmount);
uint256 reward FeesShares = getSharesByPooledWSX(rewardFeeAmount);
_mintShares(DAO_ADDRESS, reward FeesShares);
totalPooledWSX += rewardFeeAmount;

**Re-audit comment**

Resolved.

Post-audit. Changes were applied.

## Medium Risk

### Initial number of Oracles is not adjusted.
**Description**

StWSX.sol, constructor
The initial Oracle is assigned the Oracle role, but the number of Oracles (numOracles) is not adjusted.

**Recommendation**

Add increment of numOracles to the constructor.

**Re-audit comment**

Resolved

### No validation for WSX rewards / unstaked amount.
**Description**

StWSX.sol, oracle Report Unstaked Withdraw(), oracle ReportRewards()
It is assumed that the necessary amount of WSX will already be present on the contract at the moment of reporting. However, there are no checks to show that the rewards / unstaked amount was actually transferred to the contract - either by Oracle or by another entity.

**Recommendation**

Add validation for the balance before and after reporting and/or add transferFrom() (or another hook) for WSX into the reporting functions. Auditors assume that such a check may also be added to the withdraw/claim functions, depending on the process of forwarding tokens from the Staking Proxy.

**Re-audit comment**

Resolved.

Post-audit. The process of processing the rewards collection and unstaking was updated as the auditors have suggested (See Info-12).

### Sum of balances doesn't match up with total supply after rewards collection.
**Description**

StWSX.sol
After the claiming of rewards with the function oracleClaimRewards(), the total supply and sum of users' balances don't match up. The difference is estimated in several wei only. However, it has the potential to increase as deposit balances and rewards increase. Moreover, users are able to specify more tokens than their balance when unstaking. However, the amount by which they could specify more doesn't exceed the difference between the sum of balances and total supply. If all the users and DAO unstake their shares, the total supply will remain non-zero. The issue is marked as Medium since the round of testing already showed that balances and total supply may differ, and this difference will increase as deposits and rewards increase.

**Recommendation**

Increase the accuracy of calculations when calculating shares.

**Re-audit comment**

Unresolved.

Post-audit. The LiquiStake team has acknowledged the issue and decided to implement a back-end service that will track the difference and perform a "dust rebase" on smart contracts. However, the fix wasn't implemented during the current audit iteration.

### Updating the total supply before minting of new shares.
**Description**

StWSX.sol: oracle Claim Rewards().
When collecting rewards, part of them is transferred to the DAO in the form of new shares. However, the shares are calculated and minted after the variable 'totalPooledWSX is increased (Contrary to deposit flow, where shares are calculated and minted first and the totalPooledWSX is increased after). Though the current tests showed that the change of order may increase accuracy up to several weis only, this can be increased as the number of rewards increases.

**Recommendation**

Change the order of minting shares and increasing 'totalPooledWSX , so that it corresponds to the deposit flow.

**Re-audit comment**

Resolved.

Post-audit. The order of operations correspond to the deposit flow now.

## Low Risk

### No events for critical information.
**Description**

StWSX.sol: oracleUnstake(), force Unstake(), forceWithdrawUnstaked(), oracleWithdrawUnstaked(), oracleClaim Rewards(), oracleReportUnstaked Withdraw(),
Mentioned functions require events to show the change of the requested amount. For now, the contract does not log either the receiving of WSX from Staking Proxy or the moment of fulfilling requests and re-setting of the waitingToUnstake variable and amount of the request. And so, the critical information cannot be retrieved in any other way than by reading the rough storage changes.

**Recommendation**

Add event tracking the moment of re-setting of the accumulated unstake request, amount, and responsible Oracles.

**Re-audit comment**

Resolved

### Infinite allowance usage.
**Description**

IStWSX.sol, INFINITE_ALLOWANCE, transferFrom()
By default, the contract allows user flows with an infinite allowance, which is, in general, bad practice. It creates a greater risk for users in case of protocol compromise or user misbehavior during interactions with the dApp.

**Recommendation**

Remove the default flow with infinite allowance and ensure that the UI has appropriate notification regarding the allowance policy and requests only strictly necessary amounts from users for approval.

**Re-audit comment**

Verified.

Post-audit. According to the Liqui Stake team, the functionality will remain for ease of use. Also, according to the dev team, infinite allowance is not used on the frontend part. Nevertheless, the security team leaves the concern regarding the existence of the infinite approve which increases the risk for users.

### Use of magic number.
**Description**

StWSX.sol, lines 418, 428, 438
The contract utilizes a magic number (10000) representing the precision of the percent calculation. In general, it is a bad practice, which decreases the readability of the code and may affect further development.

**Recommendation**

Consider usage of the constant for precision.

**Re-audit comment**

Resolved

### Missing value checks for fees and slippage.
**Description**

1. StWSX.sol: setMaxSlippage(). The parameter is not checked for a minimum value. This allows slippage to be set to 0 or a very low value, allowing it during reward harvesting.
2. StWSX.sol: setMintFee(), setRewardFee(). Fee parameters are not checked for a maximum value. This makes it possible to set fees to a large percentage (up to 100%), taking all the deposits from users or rewards.

**Recommendation**

Add validations for slippage and fees.

**Re-audit comment**

Resolved.

Post-audit. After a series of fixes, the validations were properly implemented. The LiquiStake team has removed setMaxSlippage() function due to its uselessness. As for the functions setMintFee() and setRewardFee(), a 35% limitation was introduced, allowing to set zero fees as well.

### Missing default visibility for storage variables.
**Description**

StWSX.sol
DAO_ADDRESS
totalPooledWSX
Visibility of variables should be explicitly marked, in order to increase the readability of the code. Issue is marked as Low-risk, as it is classified as standard Solidity vulnerability

**Recommendation**

Explicitly mark visibility of variables.

**Re-audit comment**

Resolved.

Post-audit. Visibility is now explicitly marked for all the variables.

## Informational

### Substandard Increase/Decrease allowance functionality.
**Description**

IStWSX.sol
Functions for increasing/decreasing allowance are substandard and usually not recommended for usage, as they amplify the risks connected to incorrect and dangling allowances. For example, OpenZeppelin removed them from their ERC20 implementation starting from OZ 0.5.x.

**Recommendation**

Consider removing of mentioned functions and rely on best practices of approve usage in case of required allowance change: approve(N) → approve(0) → approve(M).

**Re-audit comment**

Resolved.

Post-audit. The contract's interface corresponds to the interface of IERC20 of OpenZeppelin 0.5.x version. Increase/Decrease allowance functionality was removed.

### Unconventional usage of AccessControl.
**Description**

The current implementation of StWSX.sol bypasses the DEFAULT_ADMIN_ROLE and grantRole() mechanics in favor of a custom role distribution interface.

**Recommendation**

Verify the current access control implementation usage or consider the conventional flow usage.

**Re-audit comment**

Resolved.

Post-audit. After a series of fixes, the roles management functionality was properly implemented. DEFAULT_ADMIN_ROLE is used now and is granted in constructor. Functions grantRole(), revokeRole() and renounce Role() were overridden to contain necessary validations.

### Hardcoded testnet addresses in constructor.
**Description**

StWSX.sol, storage and constructor()
Currently, the contract utilizes testnet addresses for 3rd party contracts, dao and Oracle, which are hardcoded directly in the constructor. This approach may not be flexible and convenient when migrating to the mainnet and may lead to errors. Also, the correctness of current addresses should be verified.
Though the LiquiStake team left the acknowledging comment in the code, it may still affect the mainnet deployment and should be noted in the report - especially since the repo does not contain deployment scripts.

**Recommendation**

Pass addresses as constructor parameters. Prepare deployment scripts for testnet and mainnet in advance. Verify the correctness of current addresses in the testnet.

**Re-audit comment**

Resolved.

Post-audit. 3rd party addresses were moved to the constructor parameter. Deployment script was added for the testnet deployment.

### Lack of events for changes to critical information.
**Description**

StWSX.sol: setMintFee(), setRewardFee(), setMaxSlippage(), setStaking Proxy(), setCompoundStakeProxy(), setWSXToken(), setDAOAddress(), setValidatorAddress()
StWSX: claim(), unstake()
WstWSX.sol: wrap(), unwrap().
Setters should emit events in order to keep track of historical changes of the variables and user's actions.

**Recommendation**

Consider adding events, so the historical info will be available for the dApp users.

**Re-audit comment**

Resolved

### Gas optimization suggestions.
**Description**

1. StWSX.sol: deposit, line 129.
Function deposit() contains a redundant check for an amount greater than 0. The validation is redundant since the next validation checks that the amount is greater than the minimum deposit amount, automatically excluding the possibility of the amount being equal to 0.
1. StWSX.sol: event FeesCollected, parameter timestamp.
The parameter 'timestamp' is excessive and increases the gas cost for event emitting since the timestamp can be obtained from the block in which the event was emitted.
1. StWSX.sol: unstake(), line 240.
The function uses operator += to add requested amount to _unstakeRequests'. Instead of +=, operator = can be used, since_unstakeRequests is checked to be 0 in line 222.
1. Redundant initialization.
StWSX.sol: line 46, 65, 68, 71.
Variables are explicitly initialized with 0, though, all uint variables are initialized with by default.

**Recommendation**

- Recommendation: Remove redundant validation.
- Recommendation: Remove excessive parameter.
- Recommendation: Change $+=to=$.
- Recommendation: Remove explicit assigning to 0.

**Re-audit comment**

Resolved

### Potential optimization for IStWSX.
**Description**

IStWSX.sol,_emitTransferAfterMinting Shares()
Function_emitTransferAfter Minting Shares() seems over-engineered, as it is a particular case of the_emitTransferEvents() function, and can be eliminated, thus simplifying the code.

**Recommendation**

Consider optimization of_emitTransferAfter Minting Shares() function.

**Re-audit comment**

Verified.

Post-audit. The LiquiStake team has decided to leave the implementation as is for simplification of logic understanding.

### Possible manipulation with rewards period length.
**Description**

StWSX.sol, oracle Report Rewards()
Rewards reporting logic emits events with reward period length. However, there are no control methods to validate if oracles behave honestly - as for now periodLength can be arbitrary. For example, oracle may withhold the transaction that should be submitted for accounting to prolong the periodLength or submit the next one within a shorter period. Thus makes possible manipulation with the periodLength, which may be crucial for the accounting further in the dApp internal flow. The issue is marked as Info, as it is connected to the business logic and operations outside of the contract logic. Thus, it cannot be classified without comments from the LiquiStake team.

**Recommendation**

Verify the reward reporting flow and significance of the reward period length information for the dApp.

**Re-audit comment**

Verified.

From client: According to the Liqui Stake team, the period is used for APR calculations only. The calculation does not impact the functionality or actual APR provided by the contract and only serves as historical data of provided rewards.

### Potential loss of accuracy on unstake.
**Description**

StWXS.sol, unstake()
The interface suggests that the user enter the amount he wants to withdraw. However, since the accounting is performed in shares, the unstake function performs a double conversion, leading to accuracy losses. The issue is marked as Info, as it is on the list for confirmation during the testing stage.

**Recommendation**

Consider adding interface for operation in shares inputs.

**Re-audit comment**

Resolved.

Post-audit. The testing phase didn't show a loss of accuracy during the withdrawal operation. However, the security team decided to leave it as an informational note, as double conversion creates a slight risk of dust problems during further development.

### User funds may be stuck due to race conditions.
**Description**

StWSX.sol, claim() (Verified)
Users can wait a long time until they have the chance to withdraw the whole requested unstake amount, as by the contract's design, the possibility of the unstake is bound to the availability of WSX on the contract. Also, by design, the contract provides a pooled approach without diversification between users. Also, the contract does not have a WSX tracking mechanism (issue Medium-2). So, it is possible a situation where a user requested an unstake, then several more users requested an unstake and claimed rewards (after appearing on the contract) before the first user could, thus leaving him waiting for the next rewards round. All this time, user tokens are locked on the contract, waiting for the appropriate timeslot with enough WSX available on the contract. Thus, users become vulnerable to race conditions, and some may have funds stuck for a long time (until enough WSX is available on the contract)
The issue is marked as Info, as it refers to business logic decisions and cannot be classified without comments from the LiquiStake team.
Auditors have prepared a scenario for the race conditions:
1. The first user requests unstake.
2. Oracle finalizes unstake request, so the first user is able to call the claim.
3. The second user requests unstake.
4. The second user claims.
5. The first user is not able to claim and should wait for another Oracle finalization since another user has used the WSX tokens requested to satisfy the claim. In this case, the StSFX contract throws an error message stating 'Not enough WSX to satisfy claim.`

**Recommendation**

• At this point, auditors understand that by design, the contract supports a pooled approach and cannot support user diversification. Still, the first recommendation is to provide suche.g. via additional storage structures tracking the time of unstakes, and not allowing "next wave" users to claim before previous ones (with some appropriate waiting time limit, of course, for the first users)
• Since the contract already provides a pooled requests model, the same "pooled claim" model may be applied for claims - separating users into "claiming waves", ordering claims, and eliminating race conditions (again, with an appropriate waiting time limit to prevent the previous wave from blocking out the next one)
• The dApp UI interface must clearly reflect the possibility of race conditions and the absence of an exact waiting time for claiming availability.

**Re-audit comment**

Resolved.

From client. The Liqui Stake team has verified that this won't be an issue in the mainnet where the withdrawal period will be sufficiently long. Still, auditors leave a concern regarding the race condition available in the smart contract.

### Autocompounding is independent from StWSX functionality.
**Description**

StWSX.sol, oracleClaim Rewards()
Autocompounding flow is fully implemented on 3rd party contracts and components, has no effect on contract storage, and does not use any contract data. It is also crucial, as the autocompounding flow may leave dust on the StWSX contract with no ability to rescue it. Not saying about its own independent issues (see High-1).
Issue is marked as Info, as it refers to the business logic decisions.

**Recommendation**

Consider removing the autocompouding feature out of the contract into a separate independent one.

**Re-audit comment**

Verified.

From client: The auto compounding functionality will stay due to its significant utility.

### Typos in NatSpec documentation.
**Description**

1. WstWSX.tokensPerStWSX()
The 'return description indicates that it returns the amount of wstWSX for a 1 amount of wstWSX for 1 amount of wstWSX which is a mistake.
2. WstWSX.unwrap()
The_wstWSXAmount description has a mistake in the word 'uwrap`.

**Recommendation**

Change the description of the 'return param in function tokensPerStWSX() to: "Amount of wstWSX for a 1 stWSX" and correct the spelling of the word 'unwrap' in the unwrap() function.

**Re-audit comment**

Resolved

### Rewards and unstake flow can be simplified.
**Description**

The following functions can be merged:
• oracleClaim Rewards() & oracle Report Rewards()
• forceWithdrawUnstaked() & oracleReportUnstaked Withdraw() + oracleWithdrawUnstaked() & oracleReportUnstaked Withdraw().
The contract utilizes two separate functions to claim rewards and report the increased amount in the Staking Proxy. However, the functionality of functions can be merged since the increase in the staked amount happens during the execution of oracleClaimRewards(). For example, the following code snippet might be used to claim and report rewards:
uint256 stakedAmountBefore = _stakingProxy.accountStake(address(this));
_compoundStakeProxy.harvestAndCompoundStake(
reward TokenList,
amountOutMins);
uint256 rewardAmount = _stakingProxy.accountStake(address(this)) - staked AmountBefore;
oracleReportRewards (rewardAmount);
// continue the logic of Oracle Report Rewards in the same function call.

**Recommendation**

Merge the functions to eliminate excessive calls for oracles.

**Re-audit comment**

Resolved.

Post-audit. Updates were implemented, eliminating excessive calls for oracles.

### Role granting logic in revoking functions.
**Description**

1. StWSX.revokeRole() In case the role passed as an argument is not the same as the oracle and admin role, the role will be used as a role grant for the user and not the other way around.
2. StWSX.renounceRole() In case the role passed as an argument is not the same as the oracle and admin role, the role will be used as a role grant for the user and not the other way around.

**Recommendation**

Change grantRole to revokeRole to match the logic of the functions.

**Re-audit comment**

Resolved

### Unreachable scenario in `renounceRole`.
**Description**

StWSX.renounce Role has the check for confirmation that the caller's address and the address that would revoke the role are the same. But in case the role is an admin role, there is a check that the caller should not be the address from which the role will be revoked. In this case, this part of the code is illogical and unreachable.

**Recommendation**

Remove the block in case a user wants to remove administrator rights from himself and do a revert.

**Re-audit comment**

Resolved
