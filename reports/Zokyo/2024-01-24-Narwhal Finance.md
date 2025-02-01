**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### The undercollateralized positions may avoid being liquidated

**Severity**: Critical

**Status**: Resolved

**Description**

The stopTrade function in the Trading contract is designed to be called for the liquidation of undercollateralized positions. This function verifies various conditions to ensure a trade can be legitimately closed. These include checking if the trade exists, ensuring compliance with the minimum acceptance delay, and confirming that the closing price has been reached. 
mo
Additionally, the stopTrade function includes a check
```solidity
require(ot.lastUpdateTime + minAcceptanceDelay <= block.timestamp,"wait");
```
which is intended to prevent the premature liquidation of a position. However, this mechanism can be exploited by the position owner.

The contract also includes an updateTPAndSL function, which allows the position owner to update the TP (Take Profit) and SL (Stop Loss) values of their open trades. 
Upon anticipating potential liquidation, the position owner can utilize the updateTPAndSL function to make slight adjustments to the TP/SL values. Each invocation of this function updates the lastUpdateTime of the open trade. Consequently, the owner can continually front-run the liquidation order and advance the lastUpdateTime, effectively preventing the execution of the stopTrade function due to the minAcceptanceDelay condition. This loophole allows the position owner to indefinitely delay the liquidation of an undercollateralized position.

**Recommendation**: 

Consider introducing a separate tracking mechanism for the lastUpdateTime that is only affected by market-related changes, not by TP/SL updates. 
Omit the lastUpdateTime condition if the transaction is intended to liquidate an undercollateralized position.

### Lack of incentives to liquidate small positions

**Severity**: High

**Status**: Resolved

**Description**

The protocol expects positions to be liquidated when their margin value falls below the liquidation threshold. Liquidators should be incentivized to carry out these liquidations. If there is no profit to be earned, no one will undertake the liquidation. However, if a user opens many positions with very low margins, and these become undercollateralized, the gas cost of liquidation may exceed the profits from liquidating these positions. This could result in bad debt remaining in the protocol.

**Recommendation**: 

Set a minimum position size to ensure that liquidations are profitable.

## Medium Risk

### Division by Zero in `_getNextClaimableAmount` Function

**Severity**: Medium

**Status**: Resolved

**Description**:

The `VestingSchedule` contract, particularly in its `_getNextClaimableAmount` function, is vulnerable to a division by zero error. This issue occurs when `groupVestingDuration[addressGroup[_account]]` is zero, a scenario that might not be uncommon, especially if vesting durations for certain groups are not set correctly or are inadvertently set to zero. The division by zero can lead to a panic error, causing contract execution to fail and potentially locking funds.
**Scenario:**
An account is added to the vesting schedule with a specific group.
The group's vesting duration is either not set or mistakenly set to zero.
The account tries to claim vested tokens.
The contract attempts to calculate the claimable amount using groupVestingDuration[addressGroup[_account]] which is zero, leading to a division by zero error.
This scenario can occur during regular operation, particularly in cases where new vesting groups are added without proper validation of input parameters.

**Recommendation:**

Implement the following changes to mitigate this vulnerability:
Input Validation: Ensure that groupVestingDuration is never set to zero in the addGroupCliff function. Add checks to prevent setting a zero vesting duration.
Safe Division: In _getNextClaimableAmount, before performing the division, check if groupVestingDuration[addressGroup[_account]] is zero. If it is, handle this case by either returning zero or reverting with a clear error message.


### Improper Implementation of Whitelisting in Private Transfer Mode

**Severity**: Medium

**Status** : Resolved 

**Description**:

The `BaseToken` contract contains a feature known as "Private Transfer Mode," intended to restrict token transfers exclusively to whitelisted addresses (controlled via `isRecipientAllowed`). However, the current implementation of this feature has a critical vulnerability. The contract logic allows a transaction if either the sender or the recipient is whitelisted, rather than requiring both parties to be on the whitelist. This loophole enables non-whitelisted addresses to receive tokens through a whitelisted intermediary.
**Scenario:**
Contract State: The contract is set to "Private Transfer Mode."
Initial Transfer: A whitelisted address (whitelistedUser) receives tokens from an allowed sender (e.g., the contract owner).
Secondary Transfer: The whitelistedUser then transfers these tokens to a non-whitelisted address (nonWhitelistedUser).
Result: Despite nonWhitelistedUser not being on the whitelist, they successfully receive tokens, effectively bypassing the whitelisting mechanism.
Implication:
This flaw permits unauthorized transfer of tokens to non-whitelisted addresses, undermining the security and control intended by the private transfer mode. This could lead to unauthorized token circulation, potentially disrupting the token economy and compromising the integrity of the contract.

**Recommendation:**

To rectify this vulnerability, modify the _transfer function to enforce whitelisting checks for both the sender and the recipient in private transfer mode. This can be achieved by updating the conditional check to require both parties to be whitelisted

###  Pairs `feeIndex` and `groupIndex` can not be updated 0

**Severity**: Medium

**Status**: Resolved

In Contract PairInfos.sol, the method updatePair(...) has the following check:


```solidity
       if (_pair.feeIndex > 0) {
           p.feeIndex = _pair.feeIndex;
       }


       if (_pair.groupIndex > 0) {
           p.groupIndex = _pair.groupIndex;
       }

```

Here if owner wants to set feeIndex and/or groupIndex to 0, it will not be set without reverting leading to owner believing values are set properly.

**Recommendation**: 

Remove the check if feeIndex > 0 and/or groupIndex > 0.

## Low Risk

### Possibility of inflation of `totalAssets` through direct donation

**Severity**: Low

**Status**: Acknowledged 

**Description**

The deposit function from the Vault contract allows users to deposit a specified amount of assets into the vault. Upon a successful deposit, the user's asset amount is updated, and shares are minted based on the current total value of the vault. Users can withdraw their assets from the vault through the withdraw function. It calculates the amount of assets corresponding to the user’s shares and then performs the withdrawal, reducing the user’s share count. 
The `convertToShares` and `convertToAssets` functions allow the conversion between assets and shares based on the total value of the vault and the total supply of shares. This calculation is based on the value returned by the totalAssets function, which returns the current balance of assets held by the contract.

The vulnerability arises from the ability to inflate the totalAssets value through direct donations to the contract's address, bypassing the deposit function. An attacker can transfer assets directly to the contract, thereby increasing the balance of assets without the corresponding minting of shares. This inflation impacts the calculation of share value for subsequent legitimate deposits, leading to these users receiving fewer or even zero shares. The attacker can then withdraw their assets immediately after a user's legitimate transaction, obtaining the inflated value of the assets.

**Recommendation**: 

Implement a mechanism to track assets separately from the contract's direct balance.
Ensure that a user receives more than 0 shares for their deposit.


**Client Comment** : Our vault contract operates in two phases. In the "mint" phase, when a user deposits USDT, we mint Vault tokens at a 1:1 ratio and allocate them to the user. There is a minimum minting amount for each user, and they must reach a certain threshold to progress to the next stage, the "deposit" phase. During this phase, the Vault contract's USDT balance is calculated, and Vault tokens are minted for the user based on their deposit proportion. Similarly, there is a minimum deposit requirement. Therefore, transferring USDT to the Vault contract during the "mint" phase results in losses without any gains. In the "deposit" phase, although it may affect the proportion, this is considered normal logic. Attackers would incur greater losses with minimal gains. Additionally, our withdrawal logic allows withdrawals only at a designated time, not at any time. There are also limits on the deposit amounts for each user and an overall limit on the vault's total deposits. Considering the scenarios mentioned in the report, it seems unlikely to occur.



### Collateral limit may prevent the position owner from updating their position and avoiding liquidation

**Severity**: Low

**Status**: Acknowledged

**Description**

The `updateMargin` function in the Trading contract allows a position owner to either increase or decrease the margin of their open trade. The function retrieves the details of the open trade and performs checks to ensure the caller is the trader of the position and the amount to be added or subtracted is positive. If the margin is being increased, it checks if the new margin would exceed the collateral limit using pairInfo.isExceedGroupsCollateralLimit. If not, it transfers the additional amount from the trader.

However, in a rapidly declining market, the value of collateral can decrease quickly. If the collateral value approaches or exceeds the maximum allowed (as per isExceedGroupsCollateralLimit), traders who wish to add margin to their positions to avoid liquidation may be unable to do so. If traders cannot add margin to their positions, they face automatic liquidation if the margin falls below the maintenance margin.
The check for exceeding the group's collateral limit is not presented during the opening of a position. This inconsistency might allow positions to be opened that are already close to the collateral limit.

**Recommendation**: 

Introduce checks similar to isExceedGroupsCollateralLimit during the position opening phase. This ensures that new positions are not opened too close to the collateral limit, thereby reducing the risk of immediate liquidations in volatile markets.

### Lack of Validation in `addGroupCliff` Function

**Severity**: Low

**Status** : Acknowledged 

**Description**:

The `addGroupCliff` function in the `VestingSchedule` smart contract is intended to set the cliff and vesting duration for different groups. However, it lacks comprehensive validation for the input parameters `_cliff` and `_vestingDuration`. Specifically, the function does not enforce reasonable ranges for these parameters nor does it ensure that the total vesting duration is equal to or greater than the cliff duration. This oversight allows for the setting of impractical or illogical values, potentially leading to functional issues within the contract.
**Scenario**:
The contract accepts a vesting duration that is shorter than the cliff duration, which contradicts standard vesting logic. For instance, a cliff of 1 hour and a vesting duration of 30 minutes is accepted without error.
Extremely large values for either parameter could be set, leading to impractical or unmanageable vesting schedules.

**Recommendations**:

Input Validation: Implement checks to ensure that _cliff and _vestingDuration are within practical ranges. This could include upper limits to prevent excessively long vesting periods.
Cliff and Vesting Duration Relationship: Add a validation rule to ensure that _vestingDuration is always equal to or greater than _cliff. This aligns with standard vesting practices and avoids logical inconsistencies.


### No address(0) check

**Severity**: Low

**Status**: Resolved

**Description**

In Contract PythOracle.sol, the method rescueFunds() does not validate if _receiver is address(0) or not. If accidentally set to address(0), funds would be lost.

In Contract PairInfos.sol, the method initialize(...) transfers ownership to address _owner which is not validated to be address(0) or not.

In Contract Trading.sol, the method initialize(...) transfers ownership to address _owner which is not validated to be address(0) or not.

**Recommendation**: 

Add a check to validate if the given parameter is address(0) or not.




### Method `distributeRewardUSDT` can increment `currentBalanceUSDT` without sending USDT

**Severity**: Low

**Status**: Resolved

**Description**

In Contract TradingVault, method distributeRewardUSDT(...) has a boolean value to check if USDT needs to be transferred from msg.sender or not. 
In case _send is false and USDT is not already transferred, currentBalanceUSDT will be increased by _amount which can lead to wrong calculation of shares and assets.

**Recommendation**: 

Add a check to ensure that `currentBalanceUSDT` is equal to amount of USDT tokens in the contract.

## Informational

### Missing `PausableUpgradeable` init

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract Trading.sol, PausableUpgradeable is inherited but not initialized. 

**Recommendation**: 

Initialize the `PausableUpgradeable` in the `initialize()` method adding `__Pausable_init()`.



### Method `setNativeForKeeper` sets `nativeForCallback` as well with no events

**Severity**: Informational

**Status**: Resolved

**Description**


In Contract Trading.sol, the method setNativeForKeeper also sets nativeForCallback with emitting any event even though there is a method to set nativeForCallback. 

**Recommendation**: 

Remove the setting of `nativeForCallback` from the method `setNativeForKeeper`.


### Import `MILLAGE_DENOMINATOR`

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract FeeHelper, variable `MILLAGE_DENOMINATOR`  can be imported from ./libs/VarCanstant.sol file like in other contracts to maintain uniformity.

**Recommendation**: 

Import the variable as suggested.


### Typos 

**Severity**: Informational

**Status**: Resolved

**Description**


Contract VarCanstant.sol sould be VarConstant.sol.
`Cancle` in many of the contracts should be Cancel.
`factorAddress` in Vault.sol should be factoryAddress.
This require statement in VaultFactory should be updated.
```solidity
       require(_maxVaultCapAmount <= vaultCap, "Factory: vault usdt too high");
```
Function `dealRefereAward` should be `dealReferralAward`

**Recommendation**: 

Fix the typos.
