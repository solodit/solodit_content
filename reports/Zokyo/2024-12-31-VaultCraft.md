**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Attacker Can Grief The Management Fee To Be Lesser Than Expected

**Severity**: Critical	

**Status**: Resolved

**Description**

Fee is accrued when the share value increases / accrues value , then anyone can call the takeFees function which will accrue performance and management fee and send it to the fee recipient. 

But an attacker can do the following →

Call takeFees even when the share value has not increased, therefore no fee shares would be minted to the fee recipient but the feesUpdatedAt parameter would be updated anyhow.
```solidity
 if (shareValue > fees_.highWaterMark) fees.highWaterMark = shareValue;


        if (fee > 0) _mint(fees_.feeRecipient, convertToShares(fee));


        fees.feesUpdatedAt = uint64(block.timestamp);
```
Now when the share actually accrues value and admin calls the takeFees function the management fee would be calculated as follows →
```solidity
return
            managementFee > 0
                ? managementFee.mulDivDown(
                    totalAssets() * (block.timestamp - fees_.feesUpdatedAt),
                    365.25 days // seconds per year
                ) / 1e18
                : 0;
```
We can see it is dependent on the difference between the current block timestamp  and the feesUpdatedAt parameter which would be smaller than expected due to the attacker calling the takeFees function even when share value was same , if done perfectly the management fee can even be 0 if the attacker performs the takeFees call just before the share value would have increased (after a large deposit for example).

**Recommendation**:

Don’t update the lastUpdatedAt if the share accrued no value.



### convertToAssets is receiving asset’s decimals instead of share’s decimals, possibly leading to a miscalculation

**Severity**: Critical	

**Status**: Acknowledged

**Description**

When calculating how much worth a share is in the protocol (for example when calculating fees) we perform the following →
```solidity
function _accruedPerformanceFee(
        Fees memory fees_
    ) internal view returns (uint256) {
        uint256 shareValue = convertToAssets(10 ** asset.decimals());
```

But the function convertToAssets is being passed one asset instead of one share , although this might not have a difference if both shares and asset have the same decimals but if the asset is for example  USDC/UDST then the calculation would be incorrect , since we would be calculating of much 1e6 of a share is worth instead of 1e18 .

**Recommendation**:

Pass in one share instead of one asset to convertToAssets() 



### A vault can be added to OracleVaultController after receiving deposits leading to incorrect price

**Severity**: High	

**Status**: Acknowledged

**Description**

The function `addVault()` within the `OracleVaultController.sol` smart contract is used to add a vault to the controller to be able to update its price. The function always initialize the price to 1e18 (1:1) -- This is to prevent pausing the vault on the first update, so the `addVault()` function should be called before the vault has received any deposits. However, there not exist any check to ensure that there were no previous deposits when adding a new vault:
```solidity
function addVault(address vault) external onlyOwner { 
       highWaterMarks[vault] = 1e18;


       oracle.setPrice(vault, address(ERC4626(vault).asset()), 1e18, 1e18);


       emit VaultAdded(vault);
   }
```
As a result, a new vault with already assets deposited can be added, leading to recording and incorrect price.

**Recommendation**:

Implement a check to ensure that there are no assets within the vault (no previous deposits). If there are, revert.


### Insufficient Validation On Oracle Prices Could Lead to Stale Outputs

**Severity**: High	

**Status**: Acknowledged

**Description**

The PushOracle contract allows for the setting of prices for base / quote pairs by permissioned entities via the setPrice and setPrices functions however, there is insufficient validation on the freshness of these prices which may lead to stale and/or invalid quotes when attempting to rely on the PushOracle as a price feed. Should there be a significant movement in price for the tokens utilising the price feed, such prices may be considered outdated which may result in the loss or theft of tokens for contracts integrating with the PushOracle. 

**Recommendation**:

It’s recommended that the prices set introduces a timestamp at which the price was updated alongside its price. In addition to this, sufficient checks should be made to ensure that these prices have been updated within a reasonable time frame which could be configurable by these permissioned entities.

## Medium Risk

### Misconfiguration of Bounds Leading to Underpayment During Redeem Fulfillment

**Severity**: Medium

**Status**: Resolved

**Description**

The AsyncVault contract allows the owner to set upper and lower bounds using the setBounds function. These bounds are used in the convertToLowBoundAssets function to adjust asset calculations during redeem fulfillment. If the bounds.lower parameter is set to a value close to 1e18 (which represents 100% in the contract's context), the calculation can significantly reduce the amount of assets users receive when their redeem requests are fulfilled.

**Scenario**

Misconfiguration by Owner:

The owner sets bounds.lower to a high value, such as 0.99e18 (99%).
This setting implies that the vault should hold back 99% of the assets during redeem fulfillment.

User Redeem Request:

A user requests to redeem 100 shares.
Under normal circumstances, these shares might correspond to 100 assets.

Redeem Fulfillment:

The convertToLowBoundAssets function calculates the assets to return:

assets = totalAssets().mulDivDown(1e18 - bounds.lower, 1e18);


With bounds.lower = 0.99e18, the calculation becomes:

assets = totalAssets().mulDivDown(0.01e18, 1e18);\
This results in only 1% of the expected assets being available for redemption.
User Receives Significantly Less Assets:
The user receives only 1 asset instead of the expected 100 assets.
The remaining 99 assets are effectively withheld due to the misconfigured bound.

**Recommendation**

Restrict bounds.lower to a reasonable maximum value, such as 0.1e18 (10%), to prevent excessive withholding of assets.
Ensure that bounds.lower is less than or equal to a safe threshold that aligns with the vault's strategy and user expectations

### No slippage protection can lead to loss of funds

**Severity**: Medium	

**Status**: Acknowledged

**Description**

The `deposit()` function within the `BaseControlledAsyncRedeem.sol` smart contract is used for users to deposit assets in exchange for shares. The amount of shares minted to the users are calculated in the following way: 
```solidity
function convertToShares(uint256 assets) public view virtual returns (uint256) {
       uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.


       return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
   }
```

It can be seen that if the total number of assets increases, for the same supply, the amount of shares will decrease. This allows an attacker to front-run a deposit transaction in order to send assets to the contract so that the amount of received shares by the user is decreased in comparison to the expected amount.

**Recommendation**:

Add slippage control in order to avoid front-running issues. To implement an effective access control add a `minAmountOut` parameter and a `deadline` one.


### Fee On Transfer Tokens May Break Virtual Accounting When Withdrawing And Depositing

**Severity**: Medium	

**Status**: Acknowledged

**Description**

In DeFi tokenized vaults allow users to deposit their tokens and earn yield on those tokens deposited. These instances of ERC7540 make use of virtual accounting when attempting to calculate how many tokens are owed to the user when processing an asynchronous withdrawal. This is denoted in the internal _fulfillRedeem function within the BaseControlledAsyncRedeem contract when modifying claimable assets and shares as well as pending shares. Vault deposit and withdrawal work similarly to the ERC4626 standard however, commonly used tokens such as USDC and USDT have functionalities which enable those token owners to enable fees at any time which may cause breakages in the vault contract accounting.

**Recommendation**:

It is recommended that the balance of the deposited token is taken before and after critical function executions such as depositing, withdrawing, redeeming and minting to assert that the correct amount has in fact been transferred to the contract in addition to fees which may or may not exist. This could be done by using a modifier to make it widely accessible to multiple functions. 



### The Redeem Function In BaseControlledAsyncRedeem Does Not Check That msg.sender Can Spend Owner Funds Using Allowance

**Severity**: Medium

**Status**: Acknowledged

**Description**

The redeem function in the BaseControlledAsyncRedeem function allows users to claim their tokens by redeeming vault shares. The EIP-4626 (derived) standard enforces that when msg.sender creates a redeem transaction, there should be a check that funds can be spent by allowance. Currently, there is a check against isOperator to ensure that msg.sender has approval; however, there exists an assumption that the full balance for a user is approved which does not cater for cases where a user only wishes to approve half their balance.
```solidity
   function redeem(
       uint256 shares,
       address receiver,
       address controller
   ) public virtual override returns (uint256 assets) {
       require(
           controller == msg.sender || isOperator[controller][msg.sender],
           "ERC7540Vault/invalid-caller"
       );
       // ============= SNIP ============
```

**Recommendation**:

It’s recommended that isOperator is replaced with an allowance which utilises a uint256 as opposed to a bool to allow users to approve specific amounts to other users which removes the assumption that the whole balance has been approved forever.



### User’s funds can get locked due to ‘limits’ having no maximum limit.

**Severity**: Medium	

**Status**: Resolved

**Description**

The ´Limits´ struct within the `AsyncVault.sol` contract contains 2 variables:
`depositLimit`: Maximum amount of assets that can be deposited into the vault
`minAmount`: Minimum amount of shares that can be minted / redeemed from the vault

The owner can execute the `setLimits()` function to modify these limit variables.
```solidity
function setLimits(Limits memory limits_) external onlyOwner {
       _setLimits(limits_);
   }


   /// @dev Internal function to set the limits
   function _setLimits(Limits memory limits_) internal { 
       emit LimitsUpdated(limits, limits_);


       limits = limits_;
   }
```
The problem is that there are not any maximum upper or lower limits for these variables, meaning that they can be set to 0 or to 99999999999999999… leading to users not being able to deposit or withdraw their funds, therefore getting them locked in the contract.

**Recommendation**:

Define 2 immutable variables representing the maximum and minimum upper limits and check that the new limits set within the `setLimits()` function are between the mentioned limits variables.



### Missing valid range check for signature

**Severity**: Medium	

**Status**: Resolved

**Description**

There is a missing check for a valid range of s in the authorizeOperator() function of the BaseERC7540 contract
```solidity
       bytes32 r;
       bytes32 s;
       uint8 v;
       assembly {
           r := mload(add(signature, 0x20))
           s := mload(add(signature, 0x40))
           v := byte(0, mload(add(signature, 0x60)))
       }
```
The valid range for s is in 0 < s < secp256k1n ÷ 2 + 1
This means that the valid range for s must be in the "lower half" of the secp256k1 curve's order (n) to prevent signature malleability attacks. In other words:
0 < s < secp256k1n/2 + 1
Where 
secp256k1n =
0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141

**Recommendation**: 

Signature validation should include code :
```solidity
  if (uint256(s) > 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0) 
   revert("Invalid s value");
}
```

For example refer: 
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol#L143  

### Incorrect calculation of managementFee

**Severity**: Medium	

**Status**: Resolved

**Description**
 
The `_accruedManagementFee()` function within the `AsyncVault.sol` smart contract is used to calculate the managementFee. One of the parameters used for this calculator is the total amount of seconds per year:
```solidity
function _accruedManagementFee(
       Fees memory fees_
   ) internal view returns (uint256) {
       uint256 managementFee = uint256(fees_.managementFee);


       return
           managementFee > 0
               ? managementFee.mulDivDown(
                   totalAssets() * (block.timestamp - fees_.feesUpdatedAt),
                   365.25 days // seconds per year
               ) / 1e18
               : 0;
   }
```
It can be observed that the amount of second per year is incorrect as it has been defined as ‘365.25 days’ instead of ‘365 days’.

**Recommendation**:

Fix the amount of seconds per year using the exact correct amount in seconds: 31536000.


## Low Risk

### Use of Floating Solidity Version

**Severity**: Low	

**Status**: Acknowledged

**Description**

The contracts specify an unlocked (floating) Solidity version using pragma solidity ^0.8.25;. While this allows the contracts to compile with future Solidity versions, it also introduces potential risks. Any newly released compiler version could include breaking changes, vulnerabilities, or differences in behavior that may negatively affect the contract's functionality.
Locking the Solidity version to a specific compiler version ensures consistency and protects the contract from unanticipated changes or issues introduced in newer versions of the compiler. For instance, upgrades in Solidity may impact security features or gas optimizations.

**Recommendation**:

It is recommended to use a fixed Solidity version (e.g., pragma solidity 0.8.25;) to maintain stability and minimize unforeseen issues caused by compiler updates.

### Unbounded Gas Usage in Batch Updates

**Severity**: Low

**Status**: Acknowledged 

**Description**: 

The updatePrices and setLimits functions process arrays in a loop, with no upper limit on the number of items. A large input could cause excessive gas usage, leading to out-of-gas errors or DoS.

**Scenario**:

A keeper or owner submits a batch update with an excessively large array of price updates.
The transaction fails due to exceeding the gas limit, leaving other legitimate updates unprocessed.

**Recommendation**:

Set a reasonable maximum limit on the number of updates that can be processed in a single batch.

### Potential Misconfiguration in setFees Function

**Severity**: Low

**Status**: Acknowledged 

**Description**:

The setFees function allows the owner to set new fee rates. While there are checks to prevent setting fees above certain thresholds, there is a possibility of misconfiguration, such as setting the feeRecipient to the zero address.

Exploit Scenario:

The owner accidentally sets the feeRecipient to the zero address. As a result, when fees are taken, they might be sent to an unintended address or cause the transaction to fail.

**Recommendation**:

Enhance validation in the setFees function to prevent setting invalid fee parameters.

### Misconfiguration of Fees Leading to Excessive Charges

**Severity**: Low

**Status**: Resolved

**Description**

The setFees function allows the owner to set fee rates for performance fees, management fees, and withdrawal incentives. While there are checks to prevent setting these fees above certain thresholds (20% for performance fee, 5% for management fee, and 5% for withdrawal incentive), these thresholds themselves may be too high and can lead to excessive fees being charged to users. Additionally, there is no mechanism to prevent frequent changes to the fees, which could be exploited to charge users unexpectedly.

Scenario

Owner Sets Maximum Fees:

The owner sets the performance fee to 20% (2e17), management fee to 5% (5e16), and withdrawal incentive to 5% (5e16).
These high fees significantly reduce user returns and may not align with user expectations or industry standards.

Frequent Fee Changes:

The owner repeatedly changes the fee structure without notice.
Users cannot predict or calculate their expected returns accurately.

**Recommendation**

Reduce the maximum allowable fees to more reasonable levels (e.g., 2% management fee, 10% performance fee).
Introduce a timelock mechanism that delays fee changes, giving users time to react.

## Informational

### Redundant code and assumptions in tests can introduce bugs in the future

**Severity**: Informational

**Status**: Acknowledged

**Description**

 The following lines are redundant in the unit tests and fuzz tests and can be removed:
A) Test Contract Name: BaseControlledAsyncRedeemTest
1)      Function Name: testFuzz_WithdrawWithOperator()

      asset.mint(owner, redeemAmount);
      asset.approve(address(baseVault), redeemAmount);

2)      Function Name: testWithdrawWithOperator()
      asset.mint(owner, redeemAmount);
      asset.approve(address(baseVault), redeemAmount);


3)      Function Name: testPartialFulfillRedeem()
      asset.mint(owner, partialAmount);
      asset.approve(address(baseVault), partialAmount);


4)      Function Name: testRedeemWithOperator()
      asset.mint(owner, redeemAmount);
      asset.approve(address(baseVault), redeemAmount);


5)      Function Name: testFuzz_RedeemWithOperator()

      asset.mint(owner, redeemAmount);
      asset.approve(address(baseVault), redeemAmount);


6)      Function Name: testClaimableRedeemRequest()
      asset.mint(owner, redeemAmount);
      asset.approve(address(baseVault), redeemAmount);


7)      Function Name: testMaxWithdraw()

      asset.mint(owner, redeemAmount);
      asset.approve(address(baseVault), redeemAmount);


8)      Function Name: testMaxRedeem()

      asset.mint(owner, redeemAmount);
      asset.approve(address(baseVault), redeemAmount);


9)      Function Name: testFulfillRedeem()

      asset.mint(owner, redeemAmount);
      asset.approve(address(baseVault), redeemAmount);


10)     Function Name: testPartialFulfillRedeem()

      asset.mint(owner, partialAmount);
      asset.approve(address(baseVault), partialAmount);


B) Test Contract Name: AsyncVaultTest
1)      Function Name: testFulfillMultipleRedeems()

      asset.approve(address(asyncVault), redeemAmount * 2);



**Recommendation**: 

It is advised to remove the above lines from the said functions as these might be based on assumptions and can introduce unnecessary assumptions or vulnerabilities in future.

 

### Prices by PushOracle come from a centralized authority

**Severity**: Informational	

**Status**: Acknowledged

**Description**

The `setPrice()` function within the `PushOracle.sol` smart contract is used only by the owner to set and define the prices for base/quote assets. This means that the prices which are obtained and handled by the `PushOracle.sol` contract may come from a centralized authority or from an ‘unknown’ party within the scope of this contract.

```solidity
function setPrice(
       address base,
       address quote,
       uint256 bqPrice,
       uint256 qbPrice
   ) external onlyOwner {
       _setPrice(base, quote, bqPrice, qbPrice);
   }
```

**Recommendation**:

It is recommended to not used centralized oracles but instead use a decentralized one like Chainlink Price Feeds.

### Functions Across The Targeted Codebase Performs Operations Directly On State Variables Which May Result In Costly Transactions

**Severity**: Informational	

**Status**: Acknowledged


**Description**

There are various functions such as _fulfillRedeem in the BaseControlledAsyncRedeem contract which performs operations directly on currentBalance , a storage variable. Each storage variable read (opcode sload) can become quite gas intensive when operated on multiple times. When a storage variable is read for the first time (cold access), this will cost 2,100 gas units to execute the transaction. Each time after that (warm access) this will cost an additional 100 gas units thereafter. This was identified in the BaseControlledAsyncRedeem contract mainly around the usage of currentBalance being defined as a storage variable within certain functions. 


**Recommendation**:

It’s recommended that the desired storage variables are cached into a memory variable and updated once all operations are complete as memory variables only cost 3 cast units for each read.


### Internal Functions Should Be Prefixed With _ 

**Severity**: Informational	

**Status**: Acknowledged


**Recommendation**:

Internal and private functions for example handleWithdrawalIncentive and beforeFulfillRedeem which should be prefixed with an underscore (_). It is recommended that such functions be modified in order to improve the readability of the codebase.
