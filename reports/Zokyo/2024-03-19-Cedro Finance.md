**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Method `depositRequest(...)` allows the last deposit to be more than the set deposit cap

**Severity**: High

**Status**: Resolved

**Description**

In Contract Branch.sol, the method `depositRequest()` checks if the `depositCap` has been reached before processing any deposit.

```solidity
       if (depositCapReached[id]) revert DepositCapReached(TAG, id);
```
However, the deposit cap reached is set as follows:
```solidity
if (liquidity > capAmount[id]) {
           depositCapReached[id] = true;
       }
```
Here the liquidity is calculated after tokens have been deposited and if liquidity is more than the cap amount, it does not revert.

For eg.

- ADMIN sets capAmount for MATIC as 1 million

- User A deposits 0.99 million MATIC and depositCapReached is still false as liquidity < capAmount.

- User B deposits 5 million MATIC and depositCapReached is set as true as liquidity > capAmount.

- Now total deposited MATIC is 5.99 million where whereas the set cap was 1 million only.

- This way a user can deposit much more than the set cap.

**Recommendation**: 

Update the method `depositRequest()` to revert if the current deposit is more than the `setCap` for that particular pool id.


### Repay allows to deposit without any upper limit

**Severity**: High

**Status**: Resolved

**Description**

In Contract Branch.sol, the method repayRequest(...) allows any user to repay their loan amount. However, there is no check on the amount being repaid. 

In Contract Core.sol, the repay(..) has the logic to deposit any extra amount other than the repay amount and mint ceTokens for the user.
Any user can use this method to deposit and mint ceTokens for any amount without any upper limit even after the deposit cap has been reached.

**Recommendation**: Add a check on `repayRequest(...)` method to not allow deposit amount more than the deposit cap.


### Minted `ceTokens` not added to users list if minted using `Repay()`

**Severity**: High

**Status**: Resolved

**Description**

In Contract Core.sol, the method `repay(...)` allows a user to deposit any extra amount more than the repay amount. This deposit mints `ceTokens` for the user but the same is not added using `_addToken(...)` method to keep a record of all the `ceTokens` held by the user.

This impacts the `borrow(...)` for the user or risk liquidation of users collateral because the earlier minted ceTokens will not be counted towards the health factor calculation because of the following logic:
```solidity
function healthFactor(
       address user
   ) public view returns (Data.HealthData memory health) {
       bytes32[] memory ids = users[user];
       if (ids.length > 0) {
           uint256[] memory prices = oracle.getPrices(ids);
           for (uint256 i; i < ids.length; i++) { … }
     … }
 … }
```
Here only ids in the `users[user]` list are used for calculating the health factor. 

**Recommendation**: 

Add the `ceToken` using `_addToken(...)` for users using `repay(...)` to mint ceTokens.





###  Liquidation bonus is 0% when `currentLTV > 1e18`

**Severity**: High

**Status**: Acknowledged

**Description**

In Contract LiquidationManager.sol, the method `liquidate(..)` has the following check:
```solidity
       uint256 discount; 
       uint256 liqPercent = 1e18;
       if (currentLTV < 1e18) {
           discount = 1e36 / currentLTV - 999000000000000000;
           console.log("discount", discount);
           if (discount > config.liqMaxDiscount) {
               discount = config.liqMaxDiscount;
           }
…}}
```
Here discount is 0 if `currentLTV` is more than 1e18. This means there is no liquidation bonus for liquidators and will lead to a pile of bad debts as there is no incentive for users to liquidate.

**Recommendation**: 

Update the liquidation logic to assign a discount amount for liquidation to provide the incentive for the same.

**Client commented**: 

We have implemented the liquidation logic in a way to mitigate the toxic liquidation spiral. So for that reason the liquidation bonus is 0% when currentLTV>1e18. Please refer to https://arxiv.org/pdf/2212.07306.pdf


### Repay paused but Liquidation enabled

**Severity**: High

**Status**: Acknowledged

**Description**

The protocol can enter a state where repay is paused for borrowers but liquidation is open. 
```solidity
 function repayRequest(
       bytes32 id,
       uint256 amount,
       bytes32 route,
       uint256 airdropAmount
   ) external payable nonReentrant {
       if (protocolPause) revert ProtocolPaused(TAG);
       if (repayPause[id]) revert DepositPaused(TAG, id); 
… }
```
Given the volatility of the market, this state will prevent borrowers from repaying their loans leading to the risk of being liquidated. 

If repayment can be paused, liquidation should be paused at the same time as well.

**Recommendation**: 

Please ensure that the protocol does not enter this state. If repayment is paused then liquidation should be paused as well.

**Client commented**: We consider this scenario.

## Medium Risk

### `initPool(...)` can reset any existing pools' `ceScaled` and `dtScaled` values to default values

**Severity**: Medium

**Status**: Resolved

**Description**

In Contract Core.sol, the method `initPool(...)` allows an account with the role `INIT_POOL` to add a new pool. However, the same method can be used to set the configuration for any existing pool which will result in `ceScaled` and `dtScaled` to 1e18 i.e. default value which will lead to loss of funds for users as these values are used to calculate the LP tokens and withdrawal amount.

**Recommendation**: 

Update the `initPool` logic to not update `ceScaled/dtScaled` for any existing pool if not needed.


### Liquidation fails if the chainId is not configured for `collateralId` or `debtId`

**Severity**: Medium

**Status**: Acknowledged

**Description**

In Contract `LiquidationManager.sol`, the method `liquidate(...)` allows a liquidator to liquidate a borrowers’ default loan.

This method has the following check:
```solidity
if (chainId[collateralId] == "" || chainId[debtId] == "")
           revert ChainIdNotConfigured(TAG);
```
This check can prevent liquidations if the `chainId` not set for either `collateralId` or `debtId`. 

For ex. Initially MATIC, FTM, BNB price is 1$ and MATIC’s chainID is set.
User A deposited 10 MATIC 
User A deposited 10 BNB
User B deposited 10 FTM
User A borrowed 5 FTM 

FTM price surges to 3.5$ 

Now User B can liquidate User B for a max amount of ~12$ (as per liquidation percent)
But User B can at max liquidate User A’s MATIC collateral as it’s chainID is set but BNB will be untouched until chainID is set.

**Recommendation**: 

It is to be ensured that chainID is set for borrowers with all collateral IDs.

### Borrowers immediately liquidated once Repayment resumes

**Severity**: Medium

**Status**: Acknowledged

**Description**

Given that liquidation is also paused along with repayment, if repayment is paused, during the pause the borrowers can become subject to liquidation due to market fluctuations. Now when repayment and liquidation are resumed simultaneously, borrowers will be liquidated immediately by liquidation bots unless they can front-run the bots transactions.

This will lead to borrowers being liquidated due to no fault of their own. 

**Recommendation**: 

It is advised to add a grace period after repayment resumes during which they can not be liquidated.

**Client commented**: 

We will inform users through our documents that users should always keep their position healthy either by depositing more collateral or repaying debt. Even if repay pauses, the borrower can keep health factor healthy by depositing.

### Centralization issues

**Severity**: Medium

**Status**: Acknowledged

**Description**

In `CEToken.sol`, the `setCore()` allows to reset the core and `PoolId` and overwrite them. 
`setTransferBlockRelease()` can be used to freeze the assets of any user by a malicious admin indefinitely. Also, the `setTransferBlockWait()` can be set to a very large value. 

The function `setRoleManager()` on line: 82 can be used to change the address of the `roleManager` at any time. Same goes for the `setRewardController()` function on line: 105. 
The same goes for `setRoleManager()` function, `setFeeProvider()` and `setCore()` function in Pool.sol. 
It also exists in `setRoleManager()` and `setDexFactory()` functions in Oracle.sol. The same goes for `setCore()` and `setRoleManager()` of LiquidationManager.sol 
The `setCore()` and `setRoleManager()` functions of `FeeProvider.sol` and the `setRoleManager()` function of `Core.sol`. And also `setRoleManager()` function of `AdapterPool.sol`. 

The same is applicable for `setRoleManager()`, `setAdapter()` and `setStargateRouterAndVault()` of `Branch.sol`. 

**Recommendation**: 

It is advised to follow the following recommendations in order to fix this issue:

- Use a multisig wallet for the contract and the roles, in order to increase decentralization and minimize the risk of malicious admin or loss of private keys. 
- It is advised to introduce a sufficient MAX cap for the num parameter of setTransferBlockWait() function, otherwise it can also lead to a Denial of Service attack in case it is set to a very large value by a malicious or compromised admin.
**Comments**: The client said that they accept that there is centralization and will inform the user about it.

### Strict equality used in `CEToken`

**Severity**: Medium

**Status**: Acknowledged

**Description**

Strict equality has been used in CEToken.sol. The transfer function can be frontrunned by a user to send a small amount to another user so that the following code on line: 173 of `_removeChain()` does not execute. For example, if Bob is sending his complete balance to Alice, then Eve could frontrun this transaction so that the Bob still has this small amount left. This would result in the if statement not executing at all. 
```solidity
if (balancePerChain[from][chainIds[i]] == 0)
```
The same issue exists on line: 293 in the `burnFromChain()` and line: 333 in the `burn()` function.

Similarly, strict equality has been used in Core.sol. The _removeToken can be made to not work as intended by denying the if code block to execute. This can be done by either sending a small amount of CEToken or DebtToken to the user, before call to `_removeToken()` is made. This can be done by frontrunning the `_removeToken()` with a griefing transaction to the user on whom the `_removeToken()` is being used.
This would also mean that call to `removeTokenForBorrower()` by `roleManager` will succeed is not guaranteed. Assuming otherwise could lead to flawed code logic.
For example, if `_removeToken()` is being called internally on address of Bob, then Eve can frontrun this transaction to send a small amount either `CEToken` or `DebtToken` to Bob so that the Bob still has this small amount left. This would result in the if statement on line: 671 not executing at all.

**Recommendation**: 

Although this is a griefing atttack, it would still be advised to use private mempools for these transactions such as Flashbots. And it is recommended to avoid using strict equality in code logic.

**Comments**: 

The client said that there would not be any problem to the user if they have a small amount left due to the front run attack. As only the cetoken can be transferred to other users.


### Check effects Interactions Pattern not being followed

**Severity**: Medium

**Status**: Resolved 

**Description**

Checks effects interactions(CEI) is not being followed in the transfer function of CEToken.sol. There is external call being made on line: 165. Not following CEI can result in cross-function reentrancies. 

The variables such as CEToken.balancePerChain and CEToken.chains that share state with other function, can be used to carry out a cross-function reentrancy attack if CEI is not followed. 
```solidity
        super._transfer(from, to, transferCeAmount);
        _handleReward(from,to);
        ICore(core).handleTransfer(from, to, poolId);
```
The same issue exists for `burn()` function of CEToken.sol. It is advised to change the code to following to fix the issue in `burn()`
```solidity
        _burn(from, burnAmount);
        _handleReward(from,from);
        return burnAmount;
```
Also the same issue exists in `burnFromChain()` of CEToken.sol on line: 290. It is advised to change the code to following to fix the issue in `burnFromChain()`
```solidity
        _burn(from, amount);
        _handleReward(from,from);
```
Again the same issue exists in `burn()` function of `DebtToken.sol` on line: 136. Thus, it is advised to change the code to following to fix the issue.
```solidity
        _handleReward(from);
        _burn(from, amount);
```
Similarly, the issue exists in `deposit()` function of Core.sol on line: 297. It is advised to change the code to the following to fix the issue. It is also recommended to use a Reentrant modifier for the `deposit()` function along with the following state changes, because `refresh()` is being called on line: 289 which also makes an external call. 
```solidity
        pool.ceSupply = pool.ceSupply.add(ceAmount);
        chainLiquidity[chainId][id] += tokenAmount;
        totalLiquidity[id] += tokenAmount;
        updateInterestRate(id);
        _addToken(user, id);
        ICEToken(pool.ceToken).mintInChain(user, ceAmount, chainId);
        emit AssetPrice(id, oracle.getSinglePrice(id));
```
The issue also exists in the `liquidate()` function of CEToken.sol on line: 359. It is advised to change th code to the following to fix the issue.
```solidity
    super._transfer(from, to, transferCeAmount);
    _handleReward(from, to);
```
The issue also exists in the `mint()` function of DebtToken.sol on line: 125.  It is advised to change th code to the following to fix the issue.
```solidity
        _mint(to, amount);
        _handleReward(to);
```
The issue also exists in the `mintInChain()` of CEToken on line: 261. It is advised to change the code to the following to fix the issue.
```solidity
        balancePerChain[to][chainId] += amount;
        totalBalancePerChain[chainId] += amount;
        _addChain(to, chainId);
        _mint(to, amount);
        _handleReward(to,to);
```
Similar issue also exists in `refresh()` function of Core on line: 206. It is advised to change the code to the following to fix the issue.
```solidity
        pool.ceSupply = pool.ceSupply.add(ceAmountTreasury);
        ICEToken(pool.ceToken).mintInChain(treasury, ceAmountTreasury, chainId);
        emit Data.MintToTreasury(id, tokenAmountTreasury, ceAmountTreasury);
```
Similar issue exists in the `repay()` function of Core.sol on line: 522. It is advised to use a Reentrant modifier for the function. In addition to that it is advised to make the following changes in order to follow CEI pattern:
```solidity
1)      
        pool.ceSupply = pool.ceSupply.add(ceAmount);
        params.repayAmount += ceAmount.mul(params.ceScaled).div(1e18);
        ICEToken(params.ceToken).mintInChain(params.user, ceAmount, chainId);
2)    
        _removeToken(user, id);
        pool.debtSupply -= params.debtAmount;
        updateInterestRate(id);
        IDebtToken(params.debtToken).burn(params.user, params.debtAmount);
```
Similar issue exists in `withdraw()` of the Core.sol as there are multiple external calls made. It is advised to use a Reentrant modifier for this function.         

The issue also persists in `withdrawDeadTokens()` of Core.sol. It is advised to use a Reentrant modifier and do the following changes to the code:
```solidity
        _removeToken(user, id);
        pool.ceSupply = pool.ceSupply.sub(burnAmount);
        chainLiquidity[chainId][id] -= tokenAmount;
        totalLiquidity[id] -= tokenAmount;
        updateInterestRate(id);
        ICEToken(pool.ceToken).burnFromChain(user, burnAmount, chainId);
```

**Recommendation**: It is advised to follow CEI pattern wherever possible and follow the above recommendations in order to fix this issue. Along with the above recommendations, it is advised to review business and operational logic so ensure it remains consistent even after the fixes. 

## Low Risk

### No `address(0)` validation for `roleManager`

**Severity**: Low

**Status**: Acknowledged

**Description**

Across the protocol, several contracts set the role manager address in the `initialize ()` method. It is not validated if `roleManagerAddress` is `address(0)` or not. If `roleManager` is set as `address(0)`, it will be irreversible and a new contract will need to be created. 

Similarly, the method `setRoleManager()` needs to validate that the roleManagerAddress is not `address(0)`.

**Recommendation**: 

Add the validation for `initialize(...)` and `setRoleManager(...)` methods to check role manager is not set to address(0).


### Underflow issue in `swap()` method

**Severity**: Low

**Status**: Resolved

**Description**

In Contract Core.sol, the `swap()` method has the following logic:

```solidity
function swap(
       bytes32 id,
       bytes32 srcChainId,
       bytes32 destChainId,
       uint256 tokenAmountInSrc,
       uint256 tokenAmountInDest
   ) external onlyRole(IRoleManager(roleManager).POOL()) {
       if (tokenAmountInDest > tokenAmountInSrc) {
           revert ExcessLiquidityInDest(TAG);
       }
       chainLiquidity[srcChainId][id] -= tokenAmountInSrc; 
…}
```
Here if `chainLiquidity[srcChainId][id] < tokenAmountInSrc`, it will lead to underflow panic error.

**Recommendation**: 

It is advised to check if `chainLiquidity[srcChainId][id] >= tokenAmountInSrc` before the operation.




### Oracle price not always validated

**Severity**: Low

**Status**: Resolved

**Description**

In Contract Oracle.sol, the method `getPrices(...)` and `getPrice(...)` retrieve price from Chainlink or Uniswap V3.

The returned price values (if stable coin or price fetched from dex) are not always validated to check if it is 0 or within range using the `maxValue` and `minValues`. 

These values are directly used in calculating health factors, and liquidity across the protocol which can lead to unexpected results if the price is set to 0 or not within range.

**Recommendation**: 

It is recommended to validate that the final Oracle price is not 0 and within range if required.


### Chainlink `basePrice` can be non-positive

**Severity**: Low

**Status**: Resolved

**Description**

In Contract Oracle.sol, the method `getPriceFromChainlink(...)` fetches assets prices from chainlink as follows:

```solidity
(
           uint80 roundID,
           int256 basePrice,
           /*uint256 startedAt*/,
           uint256 timeStamp,
           uint80 answeredInRound
       ) =
        AggregatorV3Interface(
           aggregator[_symbol]
       ).latestRoundData();


       if (basePrice == 0) revert ChainlinkMalfunction(TAG, _symbol); 
```
Here `basePrice` is of type int256 meaning it can be a negative value as well. Checking it for just equal to 0 can lead to prices being negative.

**Recommendation**:
Update the above check as follows:
```solidity
if (basePrice <= 0) revert ChainlinkMalfunction(TAG, _symbol);
```


### Incorrect Price validation 

**Severity**: Low

**Status**: Resolved

**Description**

In Contract Oracle.sol, the following check is done on the prices fetched from chainlink:
```solidity
function priceCheck(
       bytes32 _symbol,
       uint256 _price
   ) public view returns (bool status) {
       if (
           _price == 0 ||
           _price == minValue[_symbol] || 
           _price == maxValue[_symbol]
       ) {
           return true;
       }
   }
```
Here a price is a valid price even if price is less than `minValue` or price is more the `maxValue`.

**Recommendation**: 

update the above check to ensure that the fetched price is not 0 and within the price range.

```solidity
function priceCheck(
       bytes32 _symbol,
       uint256 _price
   ) public view returns (bool status) {
       if (
           _price == 0 ||
           _price <= minValue[_symbol] || 
           _price >= maxValue[_symbol]
       ) {
           return true;
       }
   }
```



### Missing validation in the method `swapRequest()`

**Severity**: Low

**Status**: Resolved

**Description**

In Contract Branch.sol, the method `swapRequest(...)` allows to swap funds between different chains. It also allows ETH but does not check if `msg.value > qty` or not. The transaction will fail if `msg.value < qty`.

Even if it’s equal to qty, it will still fail as there is no `msg.value` left for the starGate router fee.

**Recommendation**: Add the following check:
```solidity
       if (msg.value == 0 && msg.value > qty) revert InsufficientValue(TAG, msg.value, 0);
```

### The unhandled returned value in `LiquidationManager`

**Severity**: Low

**Status**: Resolved

**Description**

The function `liquidate()` ignores the return value of `transferCeAmount` from `liquidate()` on line: 255. 

**Recommendation**: 

It is advised to add necessary require checks to validate and handle the returned value properly to avoid unintended issues.

### Missing sanity value checks in `initPool()` of Core

**Severity**: Low

**Status**: Acknowledged 

**Description**

There is missing sanity value checks for config parameter in `initPool()`. It is advised to add appropriate require checks such as zero value check for `optimalUtilizationRatio`, etc. 

Failing to do so can lead to division by zero ppanic such as on line: 178. Also the `optimalUtilizationRatio` should never be 1 and it is advised to add a require check for this in `initPool` as well, 
otherwise it can lead to division by zero error again on line: 188. 
The `updateInterestRate` function in the Core contract, along with the `ConfigData` structure, lacks essential validations for various parameters. This absence of validation checks can lead to scenarios where critical financial parameters are set to values that could destabilise the pool's economic model or render it susceptible to manipulation or dysfunctional behaviour.
**Optimal Utilization Ratio**:
- Missing validation for optimalUtilizationRatio in ConfigData could allow it to be set to impractical values, affecting the interest rate calculation.
**Base Interest Rate**:
- Absence of checks on baseRate allows setting unrealistic base rates, potentially leading to either extremely high or low interest rates.
**Loan to Value Ratio**:
- If the Loan to Value (LTV) ratio is set to 0, it could prohibit any borrowing, negating the purpose of the lending pool.
**Liquidation Threshold**:
- Setting the liquidation threshold to 100% could lead to immediate liquidations, posing a risk to borrowers.
**Treasury Percent**:
- Manipulation of treasuryPercent could lead to unfair distribution of liquidation proceeds or fees.
**Liquidity Closing Factor**:
- Without validation, liqClosingFac could be set to values that either make liquidations overly punitive or ineffective.

**Exploit Scenario**:
- An administrator or entity with the INIT_POOL role could set these parameters to extreme values during pool initialization or configuration updates.
- This could lead to a pool that is either too risky or unattractive for lenders and borrowers, disrupt the pool's economic balance, or make the pool vulnerable to economic attacks.

**Recommendation**: 

It is advised to add appropriate sanity value checks for the same.
Note#1: We were already aware of it and  will be checking all the values properly while deploying

### No check for staleness of Price in Oracle

**Severity**: Low

**Status**: Resolved 

**False Stale Data Declaration**: 

There is no check for staleness of price. The Oracle data feeds can return stale pricing data for many reasons. If the returned pricing data is stale, this code will execute with prices that don’t reflect the current pricing resulting in a potential loss of funds for the user and/or the protocol. The contract might incorrectly mark price data as stale (outdated) and revert transactions due to its static `acceptedDelayTime`, even when Chainlink's data update frequency (heartbeat) accommodates longer intervals. This issue could unnecessarily prevent the execution of valid transactions, causing operational inefficiencies and potentially harming user trust.

Smart contracts should always check the `updatedAt` parameter returned from `latestRoundData()` and compare it to a staleness threshold.

The staleness threshold should ideally correspond to the heartbeat of the oracle’s price feed. This can be found on Chainlink’s list of Ethereum mainnet price feeds by checking the “Show More Details” box, which will show the “Heartbeat” column for each feed. For networks other than Ethereum mainnet, make sure to select your desired L1/L2 on that page before reading the data columns.

Acceptance of Actually Stale Data: Conversely, the contract might accept price data as fresh based on its acceptedDelayTime criteria, even when the actual data is considered stale according to Chainlink's heartbeat intervals. This discrepancy can result in the execution of transactions based on outdated price information, exposing the protocol and its users to financial risks, including incorrect valuation of assets and vulnerability to exploitation.

In addition to that, it is advised to use different heartbeats and thus different intervals to check the staleness of each price feed

Missing check for Arbitrum L2 Sequencer: Also for Arbitrum, the Oracle must check whether the L2 Sequencer is down or not. 

For more details, refer- https://medium.com/cyfrin/chainlink-oracle-defi-attacks-93b6cb6541bf 

**Recommendation**: 

It is advised to follow the above 3 recommendations as stated:

Add a staleness price check for the Oracle prices. And use different heartbeats and different intervals to check staleness of price. It is advised to amend the Oracle contract to dynamically adjust its criteria for data freshness based on the heartbeat and deviation thresholds specific to each Chainlink Price Feed. This ensures that the contract's assessment of data freshness accurately reflects the intended update frequency and market conditions, as defined by Chainlink.

Check if L2 Sequencer is down or not for Arbitrum Oracles 
Note# : Recommendation 1 accepted, but as oracle.sol was originally decided to be deployed in bsc, l2 sequencer check was avoided 
Commit ID: 
09bc7206959dc780ce7bf4d02a2c280d36811940


### Insufficient Validation of Token Addresses in Pool Initialization

**Severity**: Low

**Status** : Acknowledged 

**Description** : 

The `Core.initPool` function in the given smart contract lacks necessary validations to ensure the distinctiveness of `_ceToken` and `_debtToken` (debt token) addresses. This absence of validation could lead to critical issues in the contract's lending and borrowing functionalities.

**Recommendation** : 

**Implement Address Validation**: Modify the initPool function to include checks ensuring that `_ceToken` and `_debtToken` are not only non-zero but also distinct from each other
Note#2 : admin will take care of it while initialization

### Isolated assignments of Critical variables addresses

**Severity**: Low

**Status**: Acknowledged

**Description**

There are isolated and separate assignment of critical variable addresses for each contract although they are expected to be the same across the codebase.
But it is not advised to set the addresses of roleManager, core, etc. separately in each contract. This is because this can introduce errors as it could result in 
different addresses being assigned for each of the variables. For example it is possible that `Address_V1` is assigned to `roleManager` of DebtToken.sol whereas Address_V2 
is assigned to roleManager of Pool.sol, whereas in fact it should have been Address_V2 being assigned to both the variables of the two contracts.


**Recommendation**: 

It is advised to assign the address variables in only one contract, and let the rest of the contracts read, fetch and assign values from there during initialization/deployment of the contracts.

## Informational

### No method to unset a set router

**Severity**: Informational

**Status**: Acknowledged

**Description**

In Contract Adapter.sol, the method `setRoute(...)` sets a route to be used but it can be unset in case the allowed route is compromised or not needed.

**Recommendation**: 

It is advised to add a method to unset a route if not needed


### No `amount(0)` validation for tokenAmount 

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract Pool.sol, the method `loop(...)` has a parameter `tokenAmount` which needs to be validated to be greater than 0.

**Recommendation**: 

Add the validation to check if `tokenAmount > 0` or not.


## Import not being used

**Severity**: Informational

**Status**: Resolved


In Contract Pool.sol, OwnableUpgradeable is imported but not used.
In Contract LzRoute.sol (Branch), OwnableUpgradeable is imported but not used.

**Recommendation**: Remove unused imports.



### Incorrect comments / Typos / NatSpec comments issue

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract LzRoute.sol (Branch), the comment for the `_blockingLzReceive()` method is incorrect as the cross-chain message can be from the root chain only.

Across the protocol in `initialize ()` method, the comment for `roleManagerAddress` is incorrect as it is a contract, not an EOA.

In Contract Core.sol, the method `handleTransfer()` natspec 3rd param has a typo.

In Contract CEToken.sol, the method ) natspec 1st param has an incorrect comment as it core address, not the role manager address.

In Contract Branch.sol, the internal method `_pay(...)` missing natspec comment for few parameters.

In Contract LiquidationManager.sol, the method `liquidate(...)` has a typo 
```
) revert ClosingFactotExceeded(TAG);
```
 Should be
 ```
) revert ClosingFactorExceeded(TAG);
```

In Contract LiquidationManager.sol, the method liquidate(...) has a wrong comment 
// mint and burn CEToken and DebtToken then update supply
It should be
// burn and burn CEToken and DebtToken then update supply

**Recommendation**: Update the incorrect comments/typos/natspec comments


### Protocol uses a variety of roles

**Severity**: Informational

**Status**: Acknowledged

**Description**


Across the protocol, several roles are used to perform key operations, and assigning one such role to any unwanted address or any role account turning malicious can lead to unexpected results for the protocol.

**Recommendation**: 

It is advised to assign roles to as minimum accounts as possible. Make sure to revoke roles when not needed.



### Repay `params.tokenAmount` should be greater than `params.repayAmount`

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract Core.sol, the method `repay(...)` allows a user to deposit any extra amount more than the repay amount. However, the check is as follows:
```solidity
if (params.tokenAmount >= params.repayAmount) {
           ceAmount = params.tokenAmount.sub(params.repayAmount).mul(1e18).div(
               params.ceScaled
           );
 …. }
```
Here `params.tokens` should be more than `params.repayAmount` to check the `ceAmount`.

**Recommendation**: 

update the above condition as following
```solidity
if (params.tokenAmount > params.repayAmount) { … }
```







### ceTokens `_transfer()` does not check for dead chains

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract CeToken.sol, the internal method `_transfer(...)` does not check while transferring tokens if any chain is dead or not.

```solidity
for (uint256 i; i < chainIds.length; i++) {
           amountPerChain = balancePerChain[from][chainIds[i]]
               .mul(ceAmount)
               .div(ceTokenBalance);
           balancePerChain[from][chainIds[i]] -= amountPerChain;            balancePerChain[to][chainIds[i]] += amountPerChain;
           transferCeAmount += amountPerChain; // for handling precision
           if (balancePerChain[from][chainIds[i]] == 0)
               _removeChain(from, chainIds[i]);
           _addChain(to, chainIds[i]);
       }
```
Here a user can transfer another user tokens from dead chains affecting the withdrawable capacity of the later.

Also, it is not checked if `amountPerChain >= balancePerChain[from][]chainIds[i]` before the following operation;
```
balancePerChain[from][chainIds[i]] -= amountPerChain; 
```
This can lead to an underflow panic error with transaction reverting without any reason.

**Recommendation**: 

It is advised to add a check if there is enough `balancePerChain` for the user before deducting the `amountPerChain` as done in `burn(...)` and `liquidate(...)` amount. 


### External call to unknown contract

**Severity**: Informational

**Status**: Acknowledged

**Description**

Contract `CEToken` and `DebtToken` make external calls to `rewardController` address in the `handleReward()` internal method. Since the inner workings of this contract are unknown, it is risky and can lead to unexpected results.

**Recommendation**: 

It is advised to make external calls to trusted contracts only.



### Missing `nonreentract` modifier

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract Branch.sol, the method `transferTokens()` allows to transfer ETH to the recipient address which can result in a callback. 
```solidity
if (token == address(0)) {
           (bool sent, ) = payable(receipient).call{value: localAmount}("");
           if (!sent) revert NativeTransferFailed(TAG);
       } else {
           IERC20Upgradeable(token).safeTransfer(receipient, localAmount);
       }
```

**Recommendation**: 

It is advised to add `nonreentrant` modifier to the `transferTokens()` method.


### Method `loop()` is inherently risky for users

**Severity**: Informational

**Status**: Acknowledged

**Description**

In Contract Pool.sol, the method `loop(..)` can be used to leverage collaterals to borrow and deposit the same assets in loops to earn fees/rewards.

This is risky because during the process market fluctuations can lead to the liquidation of the collateral deposited along with no borrowed assets. So in this situation, a user is left with no collateral and no borrowed assets.

**Recommendation**: 

It is recommended to acknowledge and advise accordingly to the users.

### Oracle prices fetched from DEXs can be manipulated 

**Severity**: Informational

**Status**: Resolved

**Description**

When fetching a price from an Oracle, it is advisable to minimize the reliance on prices obtained from Decentralised Exchanges (DEXs), as these prices are susceptible to manipulation. Currently, the contract allows fetching prices from both DEXs and Chainlink oracles. The potential vulnerability lies in the fact that DEX prices can be manipulated, which may lead to incorrect price data being used 

**Recommendations**:

Implement Price Deviation Check: Introduce a mechanism to compare prices obtained from DEXs with those from Chainlink. If the deviation exceeds a predetermined threshold (e.g., 5%), further verification or corrective measures should be initiated. This approach aims to ensure that the prices used by the Oracle are within a reasonable range of the more stable and less manipulable Chainlink prices.
Time-Interval Checks for Price Updates: Setting a longer time interval (e.g., 600 seconds) between price updates can deter manipulation by increasing the cost and effort required to influence prices consistently over time.
Note# We will use chainlink as primary oracle source and dex will be only used as fallback option when the chainlink malfunctions. So the dex comparison with chain link will not make sense in our oracle contract.

### Usage of Unchecked in TickMath and FullMath

**Severity**: Informational

**Status**: Acknowledged

It can be seen that unchecked block has been used in calculations of FullMath and TickMath. It is not recommended to use unchecked math as it can lead to wrapping, overflowing and underflowing risks. 

**Recommendation**: 

It is recommended to stick to using the checked math only.

**Comments**: 

The client said that they had used it in order to lower gas fees.

### Layerzero best practices not followed 

**Severity**: Informational

**Status**: Acknowledged 


This is missing check for payload size in LzApp.sol. As a result, it can lead to a large payload size being executed or undesired payload size being executed leading to unintended issues.

Additionally, `ILayerZeroUserApplicationConfig()` has been imported but it has not been used or inherited in the LzApp.sol contract. It is advised to inherit it and implement it according to the best practices of Layerzero. It is advised to implement ILayerZeroUserApplicationConfig interface in the LzApp, including the `forceResumeReceive()` function which, in the worst case can allow the owner/multisig to unblock the queue of messages if something unexpected happens.

Also there is missing `setMinDstGasa()` function  in the LzApp.
The function call on the destination chain requires a specific amount of gas or, otherwise, it will revert with out-of-gas exception.

It is the User Application’s responsibility to make sure that there are correct limits set that will instruct relayers to specify the correct amount of gas at the source chain to prevent users from inputting too low the value for gas.

Especially, when the application supports multiple message types it is recommended to specify the minimum amount of gas for each of them. The minimum amounts of gas can be set using the setMinDstGas function in LzApp contract.

Additionally, you can use custom adapter parameters (_adapterParams) to specify the gas limits for each call. When specifying the gas, one must not forget about fees that cover that. You can get the fees using estimateFees function from the endpoint contract. 

Refer these for more info- https://layerzero.gitbook.io/docs/layerzero-tooling/best-practice and https://composable-security.com/blog/secure-integration-with-layer-zero/ 

**Recommendation**: It is advised to follow best practices for Layerzero as mentioned above.

### Double assignment in LzRouteV2

**Severity**: Informational

**Status**: Resolved

**Description**

In LzRouteV2 of the Branch folder, the variable executorGas has been initialized twice in the constructor. First on line 45 and then on line: 49. 

**Recommendation**: 

It is advised to initialize the variable only once.

### Missing events for critical functions

**Severity**: Informational

**Status**: Acknowledged

**Description**

Missing events for critical functions such as `setRoleManager()` function in Branch.sol and `setRoleManager()` function of Pool.sol. 

**Recommendation**: 

It is advised to emit events for critical state changes, as emitting events would be a best practice for offchain monitoring.

**Comments**: 

The client said that the required events utilized by backend are added.

