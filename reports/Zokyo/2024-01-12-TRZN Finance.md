**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk


### Missing access control in burn

**Severity**: Critical

**Status**:  Resolved

**Description**

In the TokenStableV6 contract, there is a missing access control in `burn()`. This function can be used by an attacker to burn tokens from any address. 

**Recommendation**: 

It is advised to either allow only the msg.sender to burn his own tokens, or add an appropriate access control modifier for the same. 


### Missing access control check

**Severity**: High

**Status**: Resolved 

**Description**

In Contract VaultETH_V2, the method RM_UpdateReward(...) should only be callable by an account that has the RISK_MANAGER  role as per the comment similar to the method RM_UpdateDeposit(...).
Since this method has no access control check, malicious users can call this method which can cause DoS for other users to mint ZeUSD tokens as this method internally calls the `SellETH()` method which can mint max amount of ZeUSD as the `SellETH()` method allows this contract to mint as many tokens as possible with no upper limit. 

**Recommendation**: 

Add the following check in the method RM_UpdateReward().

```solidity
       assert(hasRole(RISK_MANAGER, msg.sender));
```

### Reentrancy attack is possible on `SellETH` function

**Severity**: High

**Status**: Resolved

**Description**

The `SellETH` function sends Ether to risk managers, if these risk managers are contracts that have malicious code, there is a possibility of reentrancy attack. Even though this contract inherits ReentrancyGaurd, the guard will only apply to functions that have nonreentrant modifier.


**Recommendations**:

To mitigate this risk, it's generally a good practice to mark the function with nonreentrant modifiers from openzeppelin.

Also place Ether transfer operation at the end of function. By doing this, the transfer operation occurs after all state changes are completed, reducing the risk of reentrancy attacks.


### Function `RM_UpdateReward` in `VaultETH_V2` has no access control

**Severity**: High

**Status**: Resolved 

**Description**

Function `RM_UpdateReward` has no access control and therefore  can be called by any user even though the function seems to have to be called by risk managers.

**Recommendations**:

Mitigate these issues by performing checks to ensure that function is not called by unexpected addresses


### Risk Managers Are Not Paid Out In `Request_BuyETH`

**Severity** - High

**Status** - Resolved

**Description**

A user can request to buy ETH from the protocol in exchange with the stable tokens using the function Request_BuyETH in VaultETH_V2 . A portion of the ETH that should be sent to the user is reserved for the risk managers which is calculated at L380. 
After calculation of these amount the risk managers are not paid , just the amount to be transferred is calculated and stored into a local storage uint array.

**Recommendation**:

Add the transfers for the risk managers.

## Medium Risk

### Use `SafeTransfer` Instead Of Transfer

**Severity** - Medium

**Status** - Resolved

**Description**

In the swapBforA function (SwapOnetoOne_V2.sol) there is a ERC20 transfer taking place at L90 . The return value of the transfer is not checked so it is possible that the transfer fails silently (returning a false ) and the rest of the function executes normally . In that case token balances and fees would be updated without any transfer taking place.

**Recommendation**:

Use safeTransfer or check the return value of the transfer


### Wrong `avgDepositPrice` calculated

**Severity**: Medium

**Status**: Resolved

**Description**

In Contract VaultETH_V2, the method Request_BuyETH(...) allows users to buy ETH using ZeUSD tokens.

In the method, the epoch’s average deposit price is calculated as follows:
```solidity
tempEpochInfo.avgDepositPrice =
           (tempEpochInfo.rmDeposit *
               tempEpochInfo.avgDepositPrice +
               sendAmt *
               ethPrice) /
           (tempEpochInfo.rmDeposit + sendAmt);

       *// Update the total deposit Ether amount for the epoch*
       tempEpochInfo.rmDeposit += ethAmt;
```
`tempEpochInfo.avgDepositPrice` is the weighted average ETH price when the user calls the "Request_BuyETH" function, but this formula is using ZeUSD tokens amount for calculating the same which is incorrect.

**Recommendation**:  

Update the above formula as follows:
```solidity
tempEpochInfo.avgDepositPrice = (tempEpochInfo.rmDeposit * tempEpochInfo.avgDepositPrice + ethAmt * ethPrice) / (tempEpochInfo.rmDeposit + ethAmt);
```

### Centralization risk

**Severity**: Medium

**Status**: Acknowledged

**Description**

In Contract VaultETH_V2, there are several methods using the modifier `onlyRole(DEFAULT_ADMIN_ROLE)` which can be used to configure important parameters for the protocol such as oracles, WETH, etc. But in the constructor, DEFAULT_ADMIN_ROLE is being set to msg.sender which is an EOA.

There is also an emergencyWithdraw method which allows DEFAULT_ADMIN_ROLE to withdraw all the deposited ETH in the vault.

This risks the whole protocol being centralized and controlled by a single EOA.

**Recommendation** 

It is advised to decentralize the usage of these functions by using a multisig wallet with at least 2/3 or a 3/5 configuration of trusted users. Alternatively, a secure governance mechanism can be utilized for the same. 

**Client comment**: We originally planned to deploy multi-sig and update it in the future by replacing the admin address with multi-sig. If you would recommend ways to improve contract code stability, we would appreciate it so we may consider how to improve.

### Require check in `MultiSig_V2` can be bypassed

**Severity**: Medium

**Status**:  Resolved

**Description**

In the MultiSig contract, the require check statement of `_numConfirmationsRequired <= _whiteWallet.length` could be bypassed if `DeregisterWhiteWallet()` is used. For example, if the wallet `_whiteWallet.length` is 3 and the `_numConfirmationsRequired` is also 3, it is possible that `DeregisterWhiteWallet()` would be called in future. 

After the call, the `_numConfirmationsRequired` would become 3 whereas the `_whiteWallet.length` would be 2, which would be a violation of the statement in constructor which tries to ensure that `_numConfirmationsRequired <= _whiteWallet.length`.
```solidity
         require(
            _numConfirmationsRequired > 0 &&
                _numConfirmationsRequired <= _whiteWallet.length,
            "invalid number of required confirmations"
        );
```

**Recommendation**: It is advised to add an internal function that decreases the `_numConfirmationsRequired` accordingly in the `DeregisterWhiteWallet()` function. 

### Uncalled base constructor in MultiSig

**Severity**: Medium

**Status**: Resolved

**Description**



The base constructor of the Ownable contract is not called in the MultiSig contract. This means that the initialOwner variable of the Ownable contract is not assigned.

**Recommendation**: 

It is advised to initialize the constructor of the Ownable contract with a call to Ownable(initialOwner) in the constructor of the MutiSig_V2 contract. 
For example: constructor(address[] memory _whiteWallet, uint _numConfirmationsRequired, address _stableToken, address initialOwner) Ownable (initialOwner){}

### Missing Fallback Function to reject accidental Ether transfers to contract StableSwap_V2

**Severity**: Medium

**Status**: Resolved

**Recommendations**:

Consider implementing a non-payable fallback function with appropriate logging to handle any unexpected Ether transfers to this contract in order to prevent Ether from being stuck in the contract.


### Missing reentrancy guard in EmergencyWithdraw / WithdrawEscrow

**Severity**: Medium

**Status**: Resolved

**Description**

By adding the nonreentrant modifier, you can help ensure that the function is not vulnerable to reentrancy attacks, especially when interacting with external contracts.


### Possible Integer Overflow/Underflow in quicksort function in utils library

**Severity**: Medium

**Status**: Resolved

**Description**

The use of `uint(left + (right - left) / 2)` might result in an overflow if the right is a very large negative number. Consider validating the array indices to avoid potential overflow issues.


### No Price Staleness Check

**Severity** - Medium

**Status** - Resolved

**Description**

The function `latestRoundData()` has been used inside ERC20PriceOracle_V2 (getChainlinkPrice) to fetch the price of an asset , but there are no price staleness checks.



**Recommendation**:

Introduce price staleness checks as follows →
```solidity
(uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) = tokenInfos[token].chainlinkOracle.latestRoundData();
require(answeredInRound >= roundID, "Stale price");
require(timestamp != 0,"Round not complete");
```

## Low Risk

### Missing checks for failures in `getChainlinkPrice` Function in `ERC20PriceOracle_V2`

**Severity**: Low

**Status**: Acknowledged

**Description**

The `getChainlinkPrice` function does not include a check for potential failure scenarios when interacting with Chainlink oracles. 

**Recommendations**:

Implement error handling mechanisms to handle potential failures in fetching oracle data.

**Client comment**: If the Chainlink price fetching is down temporarily, it will be reverted and is as intended. However, in the case there is another reason the Chainlink service cannot be used, such as a chance in API service, the service can be continuously operation after updating the Oracle contract. 
In the future, if an error occurs in the implemented method (such as change in price fetching method from Chainlink, or service interruption from Chainlink), it will be updated so that another method can be used.




### 2-step ownership transfer

**Severity**: Low

**Status**: Resolved

**Description**

In Contract VaultETH_V2, method `transferOwnership()` allows the current DEFAULT_ADMIN_ROLE to set a new DEFAULT_ADMIN_ROLE. If the wrong address is set for the `newOwner`, it will be irreversible.
Instead, opt for a 2-step ownership transfer. In the first step, the current DEFAULT_ADMIN_ROLE sets the pendingNewOwner followed by pendingNewOwner accepting the ownership.

**Recommendation**: 

Update the ownership transfer to 2-step process as suggested.



### `calcFee` Would Return Incorrect Value For A Fee-On-Transfer targetToken

**Severity** - Low

**Status** - Acknowledged

**Description**

Inside calcFee function of VaultETH_V2 contract if the targetToken is a fee on transfer token then at the transfer on L263 , tokens less than ratioFee + fixedFee would be transferred . In this case the return value ratioFee + fixedFee would be incorrect since lesser tokens were transferred. 
**Client comment**: We are not going to use a fee-on-transfer tokens. 


### No check for address(0)

**Severity**: Low

**Status**: Resolved

**Description**

In Contract VaultETH_V2, the method transferOwnership(address newOwner) does not check if the `address newOwner`  is address(0) or not. If it is set by mistake then it will be irreversible.

In Contract VaultETH_V2, the method UpdateRiskManager(...) does not check if `address to` is address(0) or not. If set by mistake, the Risk Manager ratio will be set to address(0).

**Recommendation**: Update the above methods to add validations for address(0) check.



### Use `.call()` for transferring ETH

**Severity**: Low

**Status**: Resolved

**Description**

In Contract VaultETH_V2, several instances are using payable(...).transfer(...) for sending native tokens to addresses. 
This uses a fixed 2300 gas and gas repricing may break this leading to execution fees not being refunded and being stuck in the contract forever resulting in loss of funds.

**Recommendation**: 

Use `.call()` instead to transfer native tokens with proper reentrancy mitigation.



### Missing mechanism to track Votes 

**Severity**: Low

**Status**: Resolved

**Description**

There should be a mechanism to track votes for each round of decision. Otherwise it can result in incorrect consensus being calculated in new rounds of decision making. For example, let's say there are 3 users of the Multisig- Alice, Bob and Eve. Let's say in round 1, Alice casted true, Bob casted true and Eve casted false via the vote() function. Now let's say if the numConfirmationsRequired is 2, then the function getStatus() will return true. But let's say now that a new decision is to be made and a new round of Voting starts. This time everyone Votes in the same way except Bob who forgets to vote. In the case the votestatuses of Alice, Bob and Eve should have been true, false and false respectively. But instead it is now true, true and false respectively. This would result in getStatus() again returning as true instead of false. This can lead to inconsistent or incorrect decisions being made.

**Recommendation**: 

It is advised to review business and operational logic and introduce a mechanism to track votes for each round of decision being made. And then reset the voting status before each new round of voting.

### Typing error in the `RemoveContract_toStableToken()` function

**Severity**: Low

**Status**: Resolved

**Description**


In the MultiSig_V2 contract, the line: 145 in the RemoveContract_toStableToken() function is as follows:
```solidity
        IStalbeToken(StableToken).RemoveVaultManager(_target);(_target);
```
The code “(_target);” has been repeated twice in the function as an error and is not necessary. This could render confusion and render the contract undeployable.
     
**Recommendation**: 

It is advised to remove the duplication of (_target);  

### Transfer ownership to zero address in `RiskManagerEscrow_V2`

**Severity**: Low

**Status**: Resolved

**Description**

 The `transferOwnership` function can be used to accidentally transfer ownership of the contract to a zero address and revoke the role of the current owner. This would lead to  `onlyRole(DEFAULT_ADMIN_ROLE)` function uncallable. 

**Recommendation**: 

To avoid this it is advised to add a zero address check for the newOwner parameter of the function.

 
### Missing zero value check for gap in `RiskManagerEscrow_V2`

**Severity**: Low

**Status**: Resolved

**Description**

There is no check in the `setPara()` function to check that the `_ratio` parameter assigned to the gap variable should not be zero. A zero value assigned to gap can lead to division by zero panic on line: 172. Once the gap amiable is assigned, it cannot be changed again.
```solidity
        return riskManagerEscrowAmt[riskManager][escrowToken] * (10 ** deltaDecimal) * unitRatio / gap;
```
**Recommendation**: 

It is advised to add a zero value check for the gap variable.

### Unsafe Downcasting in `TokenStableV6_V2`

**Severity**: Low

**Status**: Resolved

**Description**

There is unsafe downcasting on line: 483 and 492. 
```solidity
Line: 483             uint16 currentEpoch = uint16((block.number - epochStart) / epochTerm + 1);

Line: 492 	     return uint16((block.number - epochStart) / epochTerm + 1);
```
This can result in silent overflows or truncation of the number.

**Recommendation**: 

It is advised to use a safecast library such as that of Openzeppelin’s Safecast library.

### MIssing sanity value checks in the constructor of `SwapOne2One`

**Severity**: Low

**Status**: Resolved

**Description**

In the `SwapOne2One` contract, there is missing sanity checks for parameters of the constructor which are- _tokenA, _tokenB and `_feePercentage`. 

**Recommendation**: 

It is advised to add appropriate sanity value checks as follows:
Add a zero address check for `_tokenA` and `_tokenB`
Add a zero value check for _feePercentage and an upper value check or limit such as the one in setFeePercentage() function on line: 134.


### Add openzeppelin’s `safeMath` Library in `StableSwap_V2` and `VaultETH_V2`

**Severity**: Low

**Status**: Acknowledged

**Description**

Arithmetic operations in this contract does not use openzeppelin safeMath library to perform operations.

**Recommendations**:

Utilize the SafeMath library to prevent overflow and underflow vulnerabilities for all arithmetic operations in this contract.

**Client comment**: Solidity 0.8.0 onwards, it is provided by default. Also, the safeMath lib has disappeared starting with Openzeppelin 5.0. Still, we would appreciate it if you could advise on how to apply it as necessary.



### Transfers To The Risk Managers Might Revert

**Severity** - Low

**Status** - Resolved

**Description**

A user can sell his ETH for stable tokens using the SellETH function in VaultETH_V2 , it is possible that no ETH is left in the call/contract to compensate for the risk managers at L318 , in that case the transfer would revert . 

**Recommendation**:

Have a require statement to make sure there is enough ETH in the contract.




### Use Of slot0 To Get `sqrtPriceX96` Can Lead To Price Manipulation

**Severity** - Low

**Status** - Resolved

**Description**

Inside OracleLibrary the price (`sqrtPriceX96`) from UniswapV3 AMM is pulled from slot0, which is the most recent data point and can be manipulated easily via MEV bots and flashloans with sandwich attacks. 
Though this is handled via the price deviation checks, the results can still be manipulated to an extent.

**Recommendation**: 

Use TWAP price instead.


### Withdraw Function Might Revert If Not Sufficient Balance Of Tokens

**Severity** - Low

**Status** - Resolved

**Description**

A liquidity provider can withdraw his tokens using the withdraw() function in the SwapOneToOne contract. If the logic enters the else if at L111 where there is not enough tokenA balance in the contract , then the deficit of tokenA is compensated through tokenB transfer at L114 . But if there was not enough tokenB in the contract the transfer at L114 would revert , same reasoning for else-if condition at L117.

**Recommendation**: 

Add a require statement to make sure the token balances is sufficient.

## Informational

### Manipulate112 Should Manipulate To uint112 Instead Of 128

**Severity** - Informational

**Status** - Resolved

**Description**

The function manipulate112 manipulates a int128  whereas it should be applied to int112 instead 

**Recommendation**: 

Cast to a int112 instead


### Missing checks for parameters for setParams Function in `ERC20PriceOracle_V2`

**Severity**: Informational

**Status**: Resolved

**Description**

The setParams function allows the owner to set parameters, including unitRatio, gap, and precisionLimit. 

**Recommendations**: 

Ensure that these parameters are set within reasonable and safe ranges to avoid potential vulnerabilities.


### Missing check for array length greater than 0 `getERC20Price_High` and `getERC20Price_Low` Functions

**Severity**: Informational

**Status**: Resolved

**Description**

In both functions, there is a loop over the AMMInfos array to calculate the average price. 

**Recommendations**: 

Consider adding a check to ensure that the AMMInfos array is not empty before entering the loop to prevent division by zero.


### CEI not followed in `WithdrawEscrow()`

**Severity**: Informational

**Status**:  Resolved

**Description**

Checks effects interactions pattern has not been followed in the `withdrawEscrow()` function of the `RiskManagerEscrow` contract. As a result a compromised admin may be able carry out a reentrancy attack on the contract.

**Recommendation**: 

It is advised to follow checks-effects interactions pattern as a best practice for the same.

### Missing checks in `CheckRiskManagerStatus` function

**Severity**: Informational

**Status**: Acknowledged

**Description**

 This assumes that the inputs (escrow, riskManager, inputAmt, tokenPrice, tokenPriceDecimal) are valid and within reasonable ranges. 

**Recommendations**: 

Consider including input validation checks to ensure that these values are as expected.


### Unused params

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract VaultETH_V2, the method mintableAmount() comments mention that it returns the mintable amount based on user and epoch value.
Although it doesn’t use `address user` anywhere in calculating the same. 

**Recommendation**: 

Either update the comment or the method as it suits the protocol.


### Repetitive condition check 

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract VaultETH_V2, the method Redeem_BuyETH(...) first asserts the following:
```solidity
 userWithdrawRequest.requestWithdrawAmt <= address(this).balance
```
Later on, check the same in an `if` condition which is not needed as it is already asserted to be true and no ETH transfers happened between those lines.

**Recommendation**: 

Remove this check from the `if` condition.

### Events parameters not indexed

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract VaultETH_V2, none of the events has `indexed` parameters to ease out the filter of event logs. Indexing seller and buyer parameters in the Sell and Buy events, respectively, is advised.

**Recommendation**: 

Update the events to index the suggested parameters.


### No validation when setting parameters

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract VaultETH_V2, methods `setParam(...)` and `setParams2(...)` set a lot of different values but they are not validated.

**Recommendation**: 

Add validation for setup parameters to ensure proper configuration values.


### CEI not followed in `WithdrawEscrow()`

**Severity**: Informational

**Status**:  Resolved

**Description**

Checks effects interactions pattern has not been followed in the withdrawEscrow() function of the RiskManagerEscrow contract. As a result a compromised admin may be able carry out a reentrancy attack on the contract.

**Recommendation**: 

It is advised to follow checks-effects interactions pattern as a best practice for the same.

### Floating pragma

**Severity**: Informational

**Status**: Resolved

**Description**

Audited contracts use the following floating pragma:
```solidity
pragma solidity ^0.8.19;
```
It allows to compile contracts with various versions of the compiler and introduces the risk of using a different version when deploying than during testing.

**Recommendation**: 

Use a specific version of the Solidity compiler.


### Typos and Incorrect comment

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract VaultETH_V2, correct the typo error for the mapping riksManagerList, it should be riskManagerList.

In Contract VaultETH_V2, mapping userAmountAtEpoch has an incorrect comment.

Recommendation: Update the suggested changes.


### The `SwapOne2One_V2` pool assumes token value to be constant

**Severity**: Informational

**Status**:  Acknowledged

**Description**

The pool follows an X = Y linear curve for swaps. Nobody will want to put the costlier token in the pool as a liquidity provider. This is because let's say the market price of token A is $5 and token B is $2. Now in the pool there are 1000 tokens A and 1000 tokens B. A user would immediately swap from token B to token A. This is because he would be getting a profit straightaway as he would be getting almost the same amount of tokens he deposits(minus fees). So let's say that a user swaps B for A for 500 tokens. Assuming the fees as negligible, the user would be profiting 500*5 - 500*2 = $1500. Now the pool will have 500 token A and 1500 token B (assuming fees to be negligible for simpler calculation). As the pool follows an X = Y curve, the pool actually always assumes that the price of token A and token B are the same. This would again result in draining of token A as its actual market price is higher than token B, and thus users would be getting token A at a discount. The pool will soon end up in place where there will be 0 token A and 2000 token B. Thus, no liquidity provider would want to provide token A as he would be losing the costlier token.
Thus this pool will soon lead to liquidity issues and become defunct quickly.

The amount to be swapped in this pool remains in a 1:1 ratio. The swappable amount from one token to the other always remains constant. This can be detrimental if the token depegs from its value. It would again result in the situation as discussed above and lead to market manipulation and liquidity crisis.

**Recommendation**: 

It is advised to not use tokens for this pool which are not pegged 1:1 with each other. 

### `RiskManagerEscrow` Cannot Receive Ether

**Severity** - Informational

**Status** - Resolved

**Description**

The transfer at L90 in RiskManagerEscrow_V2 transfers ETH in the contract to the admin , but  , there is no receive/fallback or payable function to receive Ether 

**Recommendation**:

Since there can never be ether in the contract (unless it is forced sent through self destruct) the logic to transfer eth can be removed.




### Directly Calling The `calcFee` Function Will Result In Lost Ether

**Severity** - Informational

**Status** - Resolved

**Description**

Directly calling the calcFee function in VaultEth_V2.sol will result in loss of ETH for the caller , since the function calculates the fee and deducts the fee from the user.

**Recommendation**:

Make the function only callable by the contract itself.




### Unnecessary Assignment

**Severity** - Informational

**Status** - Resolved

**Description**

The assignment tokenBBalance and tokenABalance in the constructor of SwapOneToOne assigns them to the balance of each  token , since during deployment the balance of each would be 0 , it does not make sense to query the balanceOf() for the tokens , just a simple declaration would do suffice.

**Recommendation**:

No need to assign to `balanceOf(token)`
