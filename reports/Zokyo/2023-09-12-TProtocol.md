**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

## Lack of Input Validation for Liquidation Threshold

**Severity**: Medium

**Status**: Acknowledged

**Description**

rUSTPool.sol - Functions `liquidateBorrow` and `flashLiquidateBorrow` can be called by any wallet to liquidate a user's borrow position with any amount, even though the debt position of the user is healthy. Because anyone can liquidate a wallet, a bot can be set up to listen to `borrowUSDC` function calls in the mem-pool. When a user calls borrowUSDC the bot instantly liquidates them, griefing the user by not being able to use the protocol and losing yield.  

**Recommendation** 

It is advised to enable the liquidation within a certain specified threshold that is related to `depositedSharesSTBT` and `borrowedAmount`. A similar formula to the one applied in `_requireIsSafeCollateralRate` needs to be implemented for that scenario.

### APR Unintentionally Set to High Value

**Severity**: Medium

**Status**: Resolved

**Description**

In InterestRateModel.sol It is noted from `setAPR()` that the value of APR should be less than 8e5. Despite that the initial value of APR is being set to 42e5. It is pointed out in the comments that it is meant to be 4.2% which is 4.2e5.
```solidity
// Assuming the maximum is 4.2%
uint256 private APR = 42 * 1e5;

function setAPR(uint256 newAPR) external onlyRole(DEFAULT_ADMIN_ROLE) {
	require(newAPR <= 8 * 1e5, "apr should be less then 8%");
	...
}
```
**Recommendation** 

Initial value of APR needs to be changed or adjust the comments to be consistent with the code. 



### Missing Checks For Whether Arbitrum Sequencer Is Active

**Severity**: Medium

**Status**: Acknowledged

**Description**

The protocol intends to deploy to L2 chains as well.
Chainlink recommends that users using price oracles, check whether the Arbitrum sequencer is active
https://docs.chain.link/data-feeds#l2-sequencer-uptime-feeds
If the sequencer goes down, oracles may have stale prices, since L2-submitted transactions (i.e. by the aggregating oracles) will not be processed.

**Recommendation**: 

Use sequencer oracle inside the LiquidatePool.sol contract , else it might give stale prices for USDC and affect the liquidation process where we demand a certain peg from the returned amount.
 
### Funds Can Be Permanently Lost if PK leak

**Severity**: Medium

**Status**: Acknowledged 

**Description**

In the contract, `LiquidatePool.sol,` an issue can arise if the private key is compromised. This can lead to unintended consequences if the processing time is changed and someone is trying to call `finalizeLiquidationById()`.

**Recommendation**: 

Use a multi-sig wallet to combat this issue.

### Centralization Risk

**Severity**: Medium

**Status**: Acknowledged 

**Description**

The wallet behind the `POOL_MANAGER_ROLE` and `DEFAULT_ADMIN_ROLE` wallet has excessive control over the contracts in the audit. The centralized `POOL_MANAGER_ROLE` and `DEFAULT_ADMIN_ROLE` wallets have control over the following functions: 
`pause()` -  toggle the usability of the protocol. 
`unpause()` -  toggle the usability of the protocol. 
`initLiquidatePool()` - set a liquidate pool that they chose.
`initMigrator()` - change the migrator.
`revokeMigrator()` - revoke the migrator from being able to exchange old tokens.
`claimReservesFee()` - claim protocol fees. 
`setReserveFactor()` - set the reserve factor to change the capability of loans
`setInterestRateModel()` - increase or decrease the interest rate for the protocol.
`setSafeCollateralRate()` - change the collateralization for the protocol 
`acceptFlashLiquidateProvider()` - Allow a user to be a flash loan provided for the Curve pool.
`RecoverERC20()` - Withdraw any non-rUSTP from the protocol. 
`claimUSTP()` - claim a specific amount of rUSTP that needs to be recovered.
`setBorrower()` - set the borrower for the migration contract. 
`setAPR()` - set a new APR for the protocol. 
`setProcessPeriod()` - change the withdrawal time from liquidating USDC. 
`setLiquidateFeeRate()` - set the fee for liquidation.
`setFeeCollector()` - set a new wallet to collect protocol fees. 
`setReedemMXPFeeRate() - change the fee that Matrix Portal receives from liquidations. 
`setReedemPool()` - set a new Matrix Portal redemption pool. 
`setCurvePool()` - change the curve pool.
`setReedemThreshold()` - change the value for redemption of Matrix Pool. 
`setPegPrice()` - set the price for USDC

**Recommendation**: 

We recommend using a timelock for sensitive functions such as `RecoverERC20()` and using a multi-sig for all centralized functions to ensure that the protocol cannot incur large losses from a leaked private key.


### Deprecated Chainlink Usage

**Severity**: Medium

**Status**: Resolved  

**Description**

In the contract, `LiquidatePool.sol`, the function latestAnswer is a deprecated function call. If the call returns stale data, then it could return zero as the result and cause unintended consequences. It is recommended to change to lastestRoundData to get the price instead. ChainLink does not recommend using latestAnswer in their documentation.  

**Recommendation**: 

We recommend using lastestRoundData instead of latestAnswer.



### Reserves MightBe Calculated As 0 Due To Rounding

**Severity**: Medium

**Status**: Acknowledged

**Description**

Inside rUSTPool.sol inside modifier realizeInterest at L95 , we calculate reserves depending upon the total interest accrued. In case where reserveFactor is close to FEE_COEFFICIENT and totalInterest accrued is high enough , the reserves value at L98 would be calculated as 0 due to rounding. 
Example -> reserveFactor = 1e3 , in this case, if totalInterest is less than 1e5 the reserves would be calculated as 0 and be not accounted for total unclaimed reserves and rUSTP supply.

**Recommendation**: 

Have proper normalization done for the amount of the reserve or ensure that reserves to not be 0.

### Mixed up shares/rUSTP conversion

**Severity**: Medium

**Stratus**: Resolved

**Description**

rUSTP.sol - Function _burnShares() in lines: 272, 276 _sharesAmount is supposed to be converted to rUSTP but the method getSharesByrUSTPAmount is used which is expected from its formula to convert rUSTP amount to shares amount.
```solidity
272     uint256 preRebaseTokenAmount = getSharesByrUSTPAmount(_sharesAmount);
273     newTotalShares = _getTotalShares().sub(_sharesAmount);
274     totalShares = newTotalShares;
275     shares[_account] = accountShares.sub(_sharesAmount);
276     uint256 postRebaseTokenAmount = getSharesByrUSTPAmount(_sharesAmount);
```
**Recommendation** 

Function `getrUSTPAmountByShares()` needs to be used instead.

### Incorrect Share Calculation 

**Severity**: Medium

**Stratus**: Resolved

**Description**

During testing the client spotted an issue with the share calculation.The issue can occur if the protocol has less than 100% utilization rate for borrowing USDC.  When a user borrows USDC, the amount of shares is incorrectly calculated. When the user pays back the USDC, they can pay a lower amount than intended leaving a shortfall in the contract as rUSTP is rebased at a higher value. To fix this, the borrow needs to be divided by the total amount borrowed instead of by the total supply of rUSTP. 

The client correctly fixed this by creating new functions to correctly check the share values based on the total borrowed shares instead of the total supply of rUSTP. The fix was corrected in commit: 0aba6588a763ab82c0798f0fe6c32a1ff9a45cd8

## Low Risk

### Unsafe ERC-20 Transfer

**Severity**: Low

**Status**: Acknowledged

**Description**

Contracts include several occurrences of unsafe transfer of ERC20 tokens. This might lead to undesirable results if the transfer is not successful while the transaction is.
LiquidatePool.sol - Function finalizeLiquidationById()
usdc.transfer(msg.sender, redeemAmountAfterFee);
usdc.transfer(feeCollector, protocolFee);

In rUSTPool.sol -
In supplyUSDC() the transfer of usdc from msg.sender is not being ensured to fulfill by a safe transfer.
As well in _supplySTBTFor()
stbt.transferFrom(msg.sender, address(this), _amount);

As well in withdrawSTBT
stbt.transfer(msg.sender, _amount);

In withdrawUSDC
usdc.transfer(msg.sender, _amount);

In borrowUSDC
usdc.transfer(msg.sender, _amount);

In repayUSDC
usdc.transferFrom(msg.sender, address(this), _amount);

In repayAll()
usdc.transferFrom(msg.sender, address(this), convertToUSDC);

In liquidateBorrow()/flashLiquidateBorrow()
stbt.transfer(address(liquidatePool), repayAmount);

In migrate()
stbt.transferFrom(_borrower, address(this), _amount);

**Recommendation** 

Utilize SafeERC20 for token transfers.

**Fix**:  As of commit b092c1c issue is fixed in LiquidatePool.sol and only one occurrence in rUSTPool.sol while the all the other transfers remain unaddressed.

### Missing Pause Condition From transferShares Function

**Severity**: Low

**Status**: Acknowledged

**Description**

Inside rUSTP.sol , at L159  for the function transferShares it is mentioned that contract must not be paused , but there is no pause functionality implemented , either it is overlooked or the comment is deprecated.

### convertToUSDC Would Be Calculated Incorrectly If USTP Amount Is Below One

**Severity**: Low

**Status**: Resolved

**Description**

Inside rUSTPool.sol at L357 we calculate convertToUSDC by dividing by 1e12 . It’s not necessary that repayrUSTP is at least 1 USTP , so if repayrUSTP is less than 1e12 the USDC amount calculated would be incorrect.

**Recommendation**: 

Proper normalization should be done for the cases where repayrUSTP is less than 1 or ensure that repayrUSTP is at least 1.

### Input Value of j is Not Validated Combined With Unsafe Casting

**Severity**: Low

**Status**: Acknowledged

**Description**

In LiquidatePool.sol - Function flashLiquidateSTBTByCurve() does not validate the value of j to be within the accepted range. It is shown here that uint256(int256(j-1)) should be within the bounds of coins array length.
IERC20 targetToken = IERC20(coins[uint256(int256(j - 1))]);

**Recommendation** 

Require j to be within the expected valid range to avoid unexpected results.


### `withdrawUSDC` Can Cause Unexpected Losses

**Severity**: Low

**Status**: Resolved

**Description**

As rUSTP is an interest bearing token, the price should positively reflect over time. As the value goes up, it will be worth more than $1. If a user calls `withdrawUSDC()`, they can have leftover dust and think they forfeited their money. As this can be avoided by adding a function that withdraws all. 

**Recommendation**: 

We recommend changing the function name to `EmergencyUSDCWithdraw()` as depending on the deposit size of the user. The loss of interest could be detrimental to the user. 

**Fix**: 

in commit:  b092c1c80c1c77701a1b1030a72638ba8492f91c


### Hard Coded Stablecoin Price

**Severity**: Low

**Status**: Acknowledged

**Description**

Because the price of STBT is hardcoded to 1, this can cause issues if STBT starts to de-peg and can cause many unintended consequences.

**Recommendation**: 

It is always recommended to use an Oracle instead of hardcoding the price.

## Informational

### repayAll Executed With Zero Shares

**Severity**: Informational

**Status**: Resolved

**Description**

rUSTPool.sol - Function repayAll() triggers the user to repay the borrowedShares[msg.sender]. Even if the borrowedShares[msg.sender] is zero. No require statement to revert early with an error message that describes the negative result of such a transaction.

**Recommendation** 

add require statement that ensures msg.sender does have non-zero borrowedShares.

### Wasted computation on a paused contract

**Severity**: Informational

**Status**: Resolved

**Description**

rUSTPool.sol - The modifier realizeInterest precedes whenNotPaused in
function supplyUSDC(uint256 _amount) external realizeInterest whenNotPaused 

Modifier realizeInterest is costly in terms of computation and all the computation is wasted if contract is already paused so its better to check if contract paused or not before going through the body of realizeInterest.

**Recommendation** 

switch modifiers order.

### Duplicate Logic

**Severity**: Informational

**Status**: Acknowledged

**Description**

In iUSTP.sol - Functions unwrapAll() and unwrap() contains duplicate logic. So in USTP.sol - Functions unwarpAll() and withdraw() And in rUSTPool.sol - Functions flashLiquidateBorrow() and liquidateBorrow().

**Recommendation** 

Implement a private function for the common logic (i.e. _unwrapProcedure(uint256 _shares)).

**Fix**:  

This informational note is addressed in the most significant occurrence in rUSTPool.sol but ignored elsewhere.


### Missing Emit Events

**Severity**: Informational

**Status**: Resolved

**Description**

LiquidatePool.sol - No events emitted in functions that trigger a change of significant parameters by admin. As users can use this to monitor centralized changes to the protocol. Emitted events are always welcome and help bring transparency to the protocol. 

	function setProcessPeriod(uint256 _processPeriod) external onlyAdmin 
	function setFeeCollector(address _feeCollector) external onlyAdmin 
	function setLiquidateFeeRate(uint256 _liquidateFeeRate) external onlyAdmin 
	function setRedeemMXPFeeRate(uint256 _liquidateMXPFeeRate) external onlyAdmin 
	function setRedeemPool(address _redeemPool) external onlyAdmin 
	function setCurvePool(address _curvePool) external onlyAdmin 
	function setRedeemThreshold(uint256 amount) external onlyAdmin 
	function setPegPrice(int256 _targetPrice) external onlyAdmin 

migrator.sol -
   function setBorrower(address _borrower) external onlyAdmin 

**Recommendation** 

Emit an event in the body of the setter functions.

### Remaining Todo and Missing Functionality

**Severity**: Informational

**Status**: Acknowledged

**Description**

In the contract, `rUSTPool.sol`, there are two TODO’s left on line 387 and 429. It is recommended that these be solved and implemented or removed with proper reasoning in the removal.

The contract is also missing a potential `whenNotPaused()` modifier on `transferShares()`. The comment lines should be updated if this is not intended or the function should have the added modifier.

**Fix**: 

Partially Resolved

### Missing Documentation

**Severity**: Informational

**Status**: Acknowledged 

**Description**

The contracts are highly complex in how they interact with each other. The documentation should have proper details for outside reviewers and developers to understand how the protocol works and interacts with STBT properly. 

**Recommendation**: 

It is recommended that more documentation is written to accurately describe what the intended behavior of a smart contract is and its intended functionality.


### Floating Pragma

**Severity**: Informational

**Status**: Resolved 

**Description**

Throughout the codebase, the contracts that are unlocked at version ^0.8.18, and it should always be deployed with the same compiler version. By locking the pragma to a specific version, contracts are not accidentally getting deployed by using an outdated version that can introduce unintended consequences.

**Recommendation**: 

Lock the compiler version to a specific one. Known bugs are featured here.

### Custom Errors

**Severity**: Informational

**Status**: Acknowledged

**Description**

Throughout the codebase, `require` statements are used instead of custom errors. Custom errors are available from solidity version ^0.8.4. Custom errors can roughly save 50 gas per call. This gas is saved because of not having to allocate the string and store the string through the revert. Throughout the contracts in the codebase, `require` statements are used instead of custom errors. Custom errors save gas on deployment as well.

**Recommendation**: 

Convert `require` statements to custom errors to save gas on each function call and deployment. 

### Typos

**Severity**: Informational

**Status**: Resolved 

**Description**

Throughout the codebase, there are many typos in comments, variable names, and function names. These typos should be fixed for the production environment to ensure all outside reviewers do not misinterpret the intended behavior of the protocol. 
The following typos occur: 
Function `unwarpAll()` -> should be spelled `unWrapAll()`
Exists in contract `USTP.sol` and `IUSTP.sol`.
`warp()` should be changed to `wrap()`.
* @dev Allows to recovery nUSTP incorrect grammar should be changed to * @dev Allows recovery of nUSTP.
* @dev Allows to recovery any ERC20 token should be changed to * @dev Allows recovery of any ERC20 token except `rUSTP`.
* @dev unwarp iUSTP to rUSTP should be changed to * @dev unwrap iUSTP to rUSTP.
* @dev warp rUSTP to iUSTP should be changed to * @dev wrap rUSTP to iUSTP.
`lqiuidateShares` in nUSTP is spelled incorrectly and should be changed to `liquidateShares`.
`lqiuidateAmount` should be changed to `liquidateAmount`.
// At migrate. we don't check healthy should be changed to be clearer.


**Recommendation**: 

We recommend fixing the typos. 


### transferShares triggered for same address sender/recipient

**Severity**: Informational

**Status**: Acknowledged

**Description**

In rUSTP.sol - Function transferShares() is triggered even when recipient is the sender of shares. This takes place since function _transferShares(address _sender, address _recipient, uint256 _sharesAmount) also does not validate the _sender and _recipient not same address.

**Recommendation** 

Require statement to ensure addresses are not equal.
