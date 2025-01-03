**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

###  No initialization.

**Severity**: Critical

**Status**:  Resolved 

**Description**

In contract TangleswapFactory, in the initNonfungiblePositionManager,`nonfungiblePositionManager` must be initialized when NonfungiblePositionManager contract is deployed, and it should be be initialized only once, but it isn’t initialized when NonfungiblePositionManager contract deployed and the owner of the factory contract can update the it anytime.

**Recommendation**: 

Refactor the code to eliminate the centralization risk of being able to set the value multiple time, one possible way to achieve it’s to refactor it so that nonFungiblePositionManager is initalized only one time in the TangleSwapFactory at the moment of deployment of the NonFungiblePositionManager.

## Medium Risk

### Abuse of the owner power.

**Severity**: Medium

**Status**:  Resolved

**Description**


The Factory owner can change state variables that affect the behavior of the contracts.
These include calls to several functions like updating priceOracle, burn wallet  and burn fee percent.

**Recommendation**:

As this is known to be called a Centralization Risk. The recommended way to deal with this is to consider using multisig wallets or having a governance module.

**Note**: The client have stated the following: “Owner itself is a multisig wallet. When the TangleswapFactory contract is successfully deployed, it will be set to a multisig wallet by calling the setOwner method.”

## Low Risk

### No confirmation support interface

**Severity**: Low 

**Status**:  Resolved

**Description**

In contract TangleswapPool, in the setPriceOracle  there are no confirmations support interface for address. If the provided address doesn’t support the IPriceOracle interface,some method call will be reverted or the fallback function will be called which can lead to unpredictable behaviour.



**Recommendation**:



### Unnecessary operation

**Severity**: Low

**Status**:  Resolved

**Description**

In contract TangleswapPool, in the increaseObservationCardinalityNext, if the new observationCardinality is same with old observationCardinality, observationCardinalityNext of slot0 is not needed to update



**Recommendation**:

 
 
