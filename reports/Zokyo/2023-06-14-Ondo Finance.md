**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Anyone can claim excess

**Severity**: High

**Status**: Resolved

**Description**

In the RWAHubNonStableInstantMints contract, there is no check for the msg.sender to be authorized to claim the excess of the deposit. The only check there is to be KYC'd. Therefore, anyone who went through the KYC could get a claim excess

**Recommendation**: 

add a check for the msg.sender to be authorized by the depositor.user to claim the excess or at least mint to the depositor user, not to msg.sender

## Medium Risk

### Incorrect variable used to compare

**Severity**: Medium

**Status**: Resolved

**Description**

In the contract RWAHubOffChainRedemptions the function requestRedemptionServicedOffchain is using the `minimumRedemptionAmount` to compare the `amountRWATokenToRedeem` with while the `minimumOffChainRedemptionAmount` is unused

**Recommendation**: 

make sure `minimumOffChainRedemptionAmount` is needed
At the contract deployment the initial mints of the RWAHubNonStableInstantMints and RWAHubInstantMints are paused by default which may lead to excess call to `unpause` function before it actually starts working

inialize the instantMintPaused with false if mints need to be ready right after deployment

## Low Risk

### Initial mints are paused

**Severity**: Low

**Status**: Acknowledged

**Description**

At the contract deployment the claimExcess function of the RWAHubNonStableInstantMints is paused by default which may lead to an excess call to `unpause` function before it actually starts working

**Recommendation**: 

inialize the claimExcessPaused with false if claims need to be ready right after deployment

### Claim excess is paused

**Severity**: Low

**Status**: Acknowledged

**Description**

At the contract deployment the claimExcess function of the RWAHubInstantMints is paused by default which may lead to an excess call to `unpause` function before it actually starts working

**Recommendation**: 

initialize the instantRedemptionPaused with the false if claims need to be ready right after deployment
​
### Instant redemption is paused

**Severity**: Low

**Status**: Acknowledged

**Description**

At the contract deployment the claimExcess function of the RWAHubInstantMints is paused by default which may lead to an excess call to `unpause` function before it actually starts working

**Recommendation**: 

initialize the instantRedemptionPaused with the false if claims need to be ready right after deployment

## Informational

### Unused function parameter

**Severity**: Informational

**Status**: Resolved

**Description**

In the contract ommf.sol on the line 588 there is declared a variable which is not used in the code.

**Recommendation**: 

use "_" instead of the named argument

### `wommf` should be immutable

**Severity**: Informational

**Status**: Resolved

**Description**

In the contract ommfManager.sol on the line 25 there is a state variable which value is assigned during in the constructor and not defined elsewhere in the code.

**Recommendation**: 

add the `immutable` attribute to state variables that never change or are set only in the constructor

### `rwaOracle` should be immutable

**Severity**: Informational

**Status**: Resolved

**Description**

In the contract Pricer.sol on the line 45 there is a state variable which value is assigned during in the constructor and not defined elsewhere in the code.

**Recommendation**: 

add the `immutable` attribute to state variables that never change or are set only in the constructor

### `collateral` should be immutable

**Severity**: Informational

**Status**: Resolved

**Description**

In the contract RWAHub.sol on the line 25 there is a state variable which value is assigned during in the constructor and not defined elsewhere in the code.

**Recommendation**: 

add the `immutable` attribute to state variables that never change or are set only in the constructor

### `decimalsMultiplier` should be immutable

**Severity**: Informational

**Status**: Resolved

**Description**

In the contract RWAHub.sol on the line 66 there is a state variable which value is assigned during in the constructor and not defined elsewhere in the code.

**Recommendation**:

add the `immutable` attribute to state variables that never change or are set only in the constructor



### SPDX license not specified

**Severity**: Informational

**Status**: Resolved

**Description**

The ommf.sol contract doesn't contain the SPDX license header

**Recommendation**: 

add the SPDX license to ommf.sol
