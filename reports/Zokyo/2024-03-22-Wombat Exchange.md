**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### The latest `Value` is not saved in the `Write()` method

**Severity**: Medium

**Status**: Resolved

**Description**

In Library DynamicFeeHelper, the method `write(...)` is called to write data after each swap.
Since the timepoint for a block can be written only once, this method has the following condition for early return:
```solidity
PointHistory memory lastPoint = pointHistories[lastIndex];
       if (lastPoint.pointTimestamp == blockTimestamp) {
           // Early return if we've already written a timepoint this block
           lastPoint.value = value; 
           return lastIndex;
       }
```
Here, `lastPoint.value` is being updated but `lastPoint` is of type `memory` so the value will not be persisted in the storage. Hence latest value will not be saved.

Since this `value` is the first step of the data flow and the swap method can be called multiple times in a block, this not being persistent can lead to unexpected results in further calculation. 

**Recommendation**: 

Update the `value` for `lastIndex` properly so it is persistent.

## Low Risk

### Method `toLogScale` should be internal 

**Severity**: Low

**Status**: Acknowledged

**Description**

In Library DynamicFeeHelper, the method `toLogScale(...)` has the following logic:
```solidity
// bound the result
       if (result > type(int32).max) {
           return type(int32).max;
       } else if (result < type(int32).min) {
           return type(int32).min;
       } else {
           return int32(result);
       }
```
Here the result is greater than `type(int32).max`, then returned value is `type(int32).max` which incorrect if `toLofScale()` method is used directly. Any external contract relying on this value can consider the upper and lower bounds as the correct result and process accordingly which will lead to wrong calculations.

**Recommendation**: 

Update to make this method internal and to be used only through `safetoLogScale()` method.

### Method `_getHaircutRate` should check if the price anchor is set or not

**Severity**: Low

**Status**: Acknowledged

**Description**

In Contract VolatilePool.sol, the method `_getHaircutrate(...)` calculates the volatility of `fromAsset` and `toAsset` if they are not price anchor.

Since `priceAnchor` is not set in the `initialize()` method, there is a possibility of it not being assigned and this will lead to volatility calculation even for `priceAnchor` asset if it is either `fromAsset` or `toAsset` which will lead to wrong haircut rate.

**Recommendation**: 

Ensure that the price anchor is set before `getHaircutrate` can be calculated.

## Informational

### Variable `lastSwapTimestamp` getting updated inside the loop

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract `VolatilePool.sol`, the method `_postSwapHook()`  has the following code:

```solidity
for (uint256 i; i < assetCount; ++i) {
           . . .
           marketPricesLast[asset] = marketPrice;
           lastSwapTimestamp = block.timestamp;
       }
```
Since `lastSwapTimestamp` needs to be updated once in the `_postSwapHook` method, there is no need to update it in the loop.

**Recommendation**: 

Update the variable `lastSwapTimestamp` outside the loop.

### Unnecessary condition check

**Severity**: Informational

**Status**: Resolved

**Description**

In Library `RepegHelper`, the method `_getProposedPriceScales` has the following check:
```solidity
       require(normalizedAdjustmentStep <= WAD);
```
Since the `normalizedAdjustmentStep` can not be more than 0.2e18 as per the logic of the method `_getNormalizedAdjustmentStep`, this check is not needed here.

**Recommendation**: 

Remove this unnecessary check.

### Use existing values to save gas

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract VolatilePool.sol, the method `_getHaircutRate()` has the following logic:

```solidity

       uint256 fromLiability = fromAsset.liability();
       uint256 toLiability = toAsset.liability();
       uint256 rFromAsset = fromLiability > 0 ? uint256(fromAsset.cash()).wdiv(fromAsset.liability()) : WAD;
       uint256 rToAsset = toLiability > 0 ? uint256(toAsset.cash()).wdiv(toAsset.liability()) : WAD;

```

Here `fromLiability` and `toLiability` are initialized but later on `fromAsset.liability()` and `toAsset.liability()`  is used as well.

**Recommendation**: 

Use already initialized variables.

### Unsafe cast

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract VolatilePool.sol, the method `_getHaircutRate()` has the following logic:
```solidity

return
           poolData.haircutRate +
           DynamicFeeHelper.getVolatilityHaircutRate(dynamicFeeConfig, volatility.toInt256()) +
           DynamicFeeHelper.getImbalanceHaircutRate(dynamicFeeConfig, int256(rFromAsset), int256(rToAsset));
```
Here, `rFromAsset` and `rToAsset` are cast to int256 unsafely. 

**Recommendation**: Use `.toInt256()` for casting `rFromAsset` and `rToAsset`.


### Unnecessary cast

**Severity**: Informational

**Status**: Resolved

**Description**

In Contract VolatilePool.sol, the method `_postSwapHook()` has the following logic:

```solidity
           int256 value = DynamicFeeHelper.safeToLogScale((marketPrice * 1e18) / priceLast, dt);


           DynamicFeeHelper.write(dynamicFeeData[asset], uint40(block.timestamp), int32(value));
```
Here, the value returned is of type int32 but unnecessarily cast to int256 and again cast to int32 on the next line.

**Recommendation**: 

Remove the unnecessary cast and get the return value as int32.

### Pending To-do(s)

**Severity**: Informational

**Status**: Resolved

**Description**

There are pending To-do(s) in all the audited contracts. It is advised to resolve or remove those To-do(s) before proceeding with production deployment.

**Recommendation**: 

Remove/Resolve the pending to-do(s).

### Unused constants

**Severity**: Informational

**Status**: Resolved

**Description**


In the library RepegHelper, the following constant is unused.

```solidity
   int256 private constant WAD_I = 10 ** 18;
```

**Recommendation**: 

Remove the unused constant.

### Typo/Wrong comment

**Severity**: Informational

**Status**: Resolved

**Description**


RepegHelper, l#80 has a typo.
RepegHelper, l#207 has a wrong comment. It should be r* >= 1 as per whitepaper.
DynamicFee, l#46 has a typo.


### Unused imports

**Severity**: Informational

**Status**: Resolved

**Description**

Following import in VolatilePool.sol is not used.
```solidity
import '../../wombat-governance/libraries/LogExpMath.sol';
```


The following imports in RepegHelper are not used.
```solidity
import '../interfaces/IRelativePriceProvider.sol';

import '../pool/PoolV4Data.sol';
```
The following import in DynamicFeeHelper is not used.
```solidity
import '../interfaces/IAsset.sol';
```


**Recommendation**: 

Removed unused imports.

### Use specific Solidity compiler version

**Severity**: Informational

**Status**: Acknowledged

**Description**

Audited contracts use the following floating pragma:
```solidity
pragma solidity ^0.8.5;
```
It allows to compile contracts with various versions of the compiler and introduces the risk of using a 
different version when deploying than during testing.

**Recommendation**: 

Use a specific version of the Solidity compiler.
