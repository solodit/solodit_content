**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Attacker Can Grief Liquidations And Repayments

**Severity** - Critical

**Status** - Resolved 

**Description**

To liquidate an unhealthy loan position the liquidate() function inside CreditorNFT can be called by anyone where the debtAmount of debt token is paid out by the liquidator.
This function in turn calls the liquidate function of LoanVault at L133.

Inside LoanVault.sol’s liquidate() it is checked if the debtAmount (initial debt amount when loan was created) is now equal to the balance of debt token in the vault , if not revert (L163)

An attacker can see a liquidation() call in the mempool and ->

a.) Frontruns this call to send the lowest amount of debt token to the vault , say 1
b.) Now when the liquidator tries to liquidate he sends out debtAmount of tokens to the vault , let’s say they were 100
c.) It is checked that debt amount and balance of debt token balance in the vault is equal
d.) But they are not since there are a total of 101 debt tokens now , liquidation reverts.

Due to this the vault/loan position can never be liquidated and the protocol will continue to incur huge losses as the collateral value falls down.

The same problem lies in repay functionality , at L150 in LoanVault.sol it will revert due to the same case as above and make it impossible for a debtor to repay their loan , resulting in forced liquidations.

**Recommendation**:

Have an internal accounting system or change the condition to if the balance in the vault is less than debt amount, then revert instead of a strict equality.



### Attacker Can Grief `mintWithSignature` And `mintWithAutomation` Calls

**Severity** - High

**Status** - Resolved

**Description**

A debtor can accept a creditor’s loan position from the DebtorNFT.sol contract . Aside from the usual mint function (which accepts a loan offer an mints a debtor NFT) there are two more options , one is mintWithDebtorSignature which uses offchain signature and the other is mintWithAutomation which interacts with the DefaultCreditorAutomation.sol.
These 2 functions calls the _validateLoan internal function at L70 and L109 (DebtorNFT.sol)

Inside the _validateLoan at L202 it checks that the balance of the vault should be exactly the debtAmount which was set during the initialization of the loan. 
An attacker can send the minimum amount of debt token directly to the vault contract and make this line revert, and because of this the _validateLoan would always revert since the balance of the vault would never equal debtAmount now.

**Recommendation**:

Instead of relying on balanceOf() have an internal accounting system or change the condition from a strict equality to less than comparison.





### Incorrect Fee Accrual In Case Of A Referral

**Severity** - High

**Status** - Resolved

**Description**

The activateLoan() function inside LoanVault.sol is called by the DebtorNFT contract when a loan is activated . Inside the function the ‘fee’ calculated at L114(in LoanVault.sol) , let’s say this fee was calculated to be 100.

Just after fee calculation there is a check to see if there was a referral fee , if there was then referral fee is calculated as a part of the fee , let’s say this comes out to be 20and this is paid to the referral address. Then fee is readjusted at L118 to exclude the referral fee , so in our example it would be 100 - 20 = 80 , this is the fee now that should be paid out.

Finally when the _takeFee is called at L120 instead of using the new updated 80 value as fee , fee is calculated again (which will be 100 again) and is paid out.
This means instead of paying 100 as fees , we paid 120 as fees.

**Recommendation**:

Change the _takeFee(collateral, (collateralAmount_ * _fees.initialisationFee) / 10000); to _takeFee(collateral, fee); 
deductions, is transferred.

## Medium Risk

### No Price Staleness Check

**Severity** - Medium

**Status** - Acknowledged

**Description**

The function latestRoundData() has been used to fetch the price of an asset, but there are no price staleness checks.
Instances:
L24 -> AggregatedChainnlinkOracle.sol
L25 -> WBTCOracle.sol


There should be checks for the roundId and timestamp , i.e.
```solidity
(uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) = _btcUsdFeed.latestRoundData();
require(answeredInRound >= roundID, "Stale price");
require(timestamp != 0,"Round not complete");
```

**Recommendation**:

Introduce price staleness checks.
**Comment**: The client has suggested they can monitor their oracles for the StalePrice event and replace the oracles asap without disrupting the loans

### Unlimited Token Minting for `CreditUSD` and `CSProtocolToken`

**Severity**: Medium

**Status**:  Resolved

**Description**

The registeredMinters can mint an unlimited amount of tokens via mint(),  if they are also authorized to call the onlyAuthorisedRole functions such as updateMintLimit(). Loss of private keys or collusion between the registeredMinters and onlyAuthorisedRoles can lead to unlimited token minting as well.  This can lead to the CreditUSD token losing its peg.

Similarly, CSProtocolToken contract’s mint() function can be used by a malicious admin to mint unlimited tokens.

**Recommendation**: 

This can be detrimental to the project and can cause the token to lose its value. It is advised to use multisig wallet with at least 2/3 or 3/5 configuration for the contracts above.
Additionally, consider adding a supply cap for CSProtocolToken to avoid unlimited minting of tokens.
**Comments**: The client said that the mint functions will only be callable by contracts such as the CreditUSDMinter or the DebtStaking contracts. No EOA, multisig or governance would be granted the rights to mint. However the rights to add authorised minters will be granted to a multisig or governance module with a sufficient number of signers. 

### Reentrance in `mint()` of `CreditUSDMinter`

**Severity**: Medium

**Status**:  Resolved

**Description**

The mint() function of CreditUSDMinter violates checks effects interactions pattern. The external call of safeTransferFrom is made on the line: 68, which is before all the state changes in the state variables. This is not advised, because external calls before state changes can lead to reentrancy attacks. The safeTransferFrom contains call to the _checkOnERC721Received() which passes the call to untrusted contracts. This can lead to reentrancy exploits. However, this would just be a griefing attack in this case, as the attacker carrying out this exploit would lose more than gaining anything. 

**Recommendation**: 

It is still advised to use checks effects interactions pattern or use Reentrancy Guard from Openzeppelin to avoid cross function or cross-contract reentrancy exploits.

### Reentrance in `CreditorNFT`

**Severity**: Medium

**Status**:  Resolved

**Description**

The Checks Effects interactions (CEI) is not being followed in _deposit() of the CreditorNFT contract. This is because the safeTransferFrom is called on line: 182 on the debt token. If the debt token is an ERC20 token that implements hooks like beforeTransferFrom() or afterTransferFrom() which transfer calls to untrusted contracts, then this could result in reentrancy. Again this is would be a griefing attack for the attacker as he would be losing more than gaining.

**Recommendation**: 

It is still advised to use checks effects interactions pattern or use Reentrancy Guard from Openzeppelin to avoid cross function or cross-contract reentrancy exploits.

## Low Risk

### Liquidation In LounVault.sol should have a onlyCreditor Modifier

**Severity** - Low

**Status** - Acknowledged

**Description**

The `liquidate()` function inside LoanVault.sol is called via `liquidate()` function inside CreditorNFT.sol . Inside liquidate of CreditorNFT the debt tokens are sent to the vault , then liquidate of LoanVault is executed.
It is possible to call the liquidate function in LoanVault directly since there is no onlyCreditor modifier check , though it will not cause any harm it will require the liquidator to send the debt tokens via a direct transfer . It would be better to have a onlyCreditor modifier on the LoanVault’s liquidate()

**Recommendation**:

Make the liquidate() function inside LoanVault.sol only callable by the CreditorNFT.sol


### Liquidations Might Not Be Profitable

**Severity** - Low

**Status** - Acknowledged

**Description**

The liquidation inside LoanVault.sol might not be profitable for the liquidator , the liquidator needs to provide the complete debt of the loan and will only get a portion of the collateral asset(minus the protocol and creditor fee)  in return as award . 

**Recommendation**:

Have a auto liquidation process which is done by a privileged role or increase the incentives for the liquidator.


### Ensure No Rounding

**Severity** - Low

**Status** - Acknowledged

**Description**

The calculation for rewardPerToken at L53 in RewardDistributor.sol  might be subject to rounding if supply > reward*PRECISION.

**Recommendation**:

Ensure the above case is not possible.


### Missing zero address checks for CreditUSDMinter

**Severity**: Low

**Status**:  Resolved

**Description**

There is missing zero address check for creditUSD_ and controller_ parameter of function initialize(). Also there is missing zero address check in the setCollateralCreditLimit() function for collateralAsset parameter. This can lead to CreditLimit of zero address being set, which is a logical error. It can also lead to unintended or undiscovered issues. Similarly, there is missing zero address check for backingAsset parameter of the setBackingDeduction() function.

**Recommendation**: It is advised to add a zero address check for the above functions.

**Comments**: The client added zero address check for creditUSD_ but not controller as they said it would not be required as it would revert if set to 0.


### Missing zero address and sanity checks in LoanVault

**Severity**: Low

**Status**:  Acknowledged

**Description**

There is missing zero address check for controller parameter in initialize() function. Once set incorrectly, it cannot be set again. Also missing sanity checks for interestRate and liquidationThreshold_. If incorrectly assigned, it cannot be set again. Additionally, there is missing zero value check for minDuration_. Again If it is incorrectly set, it cannot be set again.

**Recommendation**: 

It is advised to add above require checks for the same.

### Missing zero address checks in `ProtocolController`

**Severity**: Low

**Status**:  Resolved

**Description**

There is missing zero address check for initialAdmin and `feeCollector_` parameter in `initialize()` function. Once set, it cannot be set again.

**Recommendation**:

It is advised to add a zero address check for the above functions.

**Comments**: The client said that the feeCollector can be set to the zero address explicitly, in which case no fees are collected (that’s an easy way to turn off all fees). This implementation was missed in some cases, fixed in commits 6e4b7cf and 4eec52a


### Missing zero address check in `CreditUSD`

**Severity**: Low

**Status**:  Acknowledged

**Description**

In CreditUSD contract, there is missing zero address check for minter parameter in updateMinter() and collateralAddress parameter in updateMintLimit() functions.

**Recommendation**: 

It is advised to add missing zero address require check for the same.

### Missing sanity checks in `RewardAuctionHouse`

**Severity**: Low

**Status**:  Resolved

**Description**

There is Missing sanity checks for parameters in initialize() function. Here the minimumBid can be accidentally set to zero, or the auctionDuration can be zero as well as the beneficiary be set to zero address. This could lead to issues as these parameters can not be reset.
```solidity
        minimumBid = minimumBid_;
        auctionDuration = auctionDuration_;
        beneficiary = beneficiary_;
```

**Recommendation**: 

It is advised to add sufficient sanity checks for the above parameters.

## Informational

### Liquidations Might Revert If Collateral Is A Token Which Reverts On 0 Value Transfer

**Severity** - Informational

**Status** - Resolved

**Description**

The collateral token can be any token whitelisted . When liquidation is called in the LoanVault.sol at L159 , withdrawableInterest is calculated at L160 and if  withdrawableInterest  is more than total collateral in the vault then withdrawableInterest  = collateralBalance at L170 , let’s say this was true and now withdrawableInterest  = collateralBalance. 
Because of this remainingCollateral calculated at L172 will be 0 and so will be the protocolFee and creditorFee , then we do _takeFee at L181 which in this case will do a 0 value transfer to the fee receiver and if collateral token reverts on 0 value transfer then the liquidation flow reverts . 
Therefore in such a case bad position won’t get liquidated.

**Recommendation**:

Have a check that if the value is more than 0 only then _takeFee.


### USDC Blacklisting Impact on LoanVault Liquidation Process

**Severity**: Informational

**Status**: Acknowledged 

**Description**:

The LoanVault contract in the system is designed to handle collateral and debt for loans, potentially including the use of USDC as either collateral or debt. USDC, being a regulated stablecoin, includes a blacklisting feature allowing the USDC contract to prevent certain addresses from executing transactions. This feature can significantly impact the liquidation process of loans within the LoanVault contract if USDC is used.
Scenario:
Liquidation Failure: When the liquidation process is initiated, the contract attempts to transfer USDC (either as part of returning collateral to the borrower or moving debt to the liquidator). However, due to the blacklisting, the USDC transfer reverts.
Resulting Impact: The entire liquidation transaction fails, leaving the loan in a state where it can neither be repaid nor liquidated. This scenario leads to a deadlock, potentially causing financial loss and operational issues within the platform.

**Recommendation**:

Several approaches can be considered:
Pre-Liquidation Checks: Implement checks within the LoanVault contract to verify whether the involved addresses are blacklisted in the USDC contract before initiating liquidation.
Fallback Mechanisms: Develop a mechanism where alternative actions are taken if a USDC transfer fails due to blacklisting. This could involve using another form of collateral or a secondary process for dealing with blacklisted addresses.

### Asset Allowlist Check in LoanVault:

**Severity**: Informational

**Status** : Acknowledged

**Description**:

 The LoanVault checks the debt asset against an allowlist, but this is not checked in the CreditorNFT.
Scenario: An unapproved asset could be passed to LoanVault, leading to a failed transaction after several steps have been executed.

**Recommendation**: 

Perform the allowlist check earlier in the _deposit function to avoid unnecessary operations with unapproved assets.


### Careful Usage of Upgradeable contracts

**Severity**: Informational

**Status**:  Acknowledged

**Description**

Upgradeable contracts have been used throughout the codebase. For example, In CreditUSD contract, the Upgradeable ERC20 has been used. And in CreditUSDMinter, ERC721HolderUpgradeable has been used. Generally it is not recommended to use upgradeable contracts as it is against the notion of immutability of code. This can introduce trust issues with users as a malicious or compromised admin can change the code if it is an upgradeable contract. Or there can be unaudited changes to the code that could unintentionally introduce new bugs in the code.


**Recommendation**:

It is advised to carefully use the upgradeable contracts and use multisig wallets for the admin. It is also advised to audit the code changes done before the upgradation of the contracts.



### Make Sure That Timelock Contract Is Deployed As Part Of The Deploy Script

**Severity** - Informational

**Status** - Unresolved

**Description**

If the timelock contract has not been deployed yet providing the appropriate roles , then the proposals won’t get queued/executed unless the role is granted to the contract.

**Recommendation**:

Make sure the CSTimelock.sol is deployed as part of the deployment script.


### Allowing Fee-On-Transfer Tokens Might Be Problematic

**Severity** - Informational

**Status** - Acknowledged

**Description**

It is possible that the creditor whitelists a fee-on-transfer token as a allowed collateral token.
These tokens might be problematic in accounting and result in wrong event emissions. For example , inside DebtorNFT.sol at L151 the debtor transfers “amount” of collateral to the vault , but in case of a fee-on-transfer token the amount that will be sent to the vault will be less than the amount , it might be possible that due to this the position becomes subject to liquidation.

When adding collateral to a vault the addCollateral function is called inside DebtorNFT.sol at L123 , the token transfer is done at L126 and in case of a fee on transfer token the event emission will be incorrect since it indexes ‘amount’ whereas the actual amount deposited would be lesser.

**Recommendation**:

Either don’t allow such tokens or have proper accounting for such tokens.


### Resolve TODOs

**Severity** - Informational

**Status** - Acknowledged

**Description**

Throughout the code base there are instances of TODOs and even empty contracts , these should be resolved before deployment. This includes ->

TODO at L31-L51 in NFTBase.sol

RenderNFT.sol

L216 in LoanVault.sol

**Recommendation**:

Resolve the above




### CEI Violation

**Severity** - Informational

**Status** - Resolved

**Description**










There are instances in the codebase where the checks effects interaction pattern has not been followed . These include ->

a.) L58 of RewardAuctionHouse.sol , there are state changes that occur after this transfer at L68 and L69

b. ) L117 of LoanVault.sol , state is changed at L118 after the transfer.









