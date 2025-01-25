**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

###  Lack of a restriction on the recoverERC20() function in the pause status

**Severity** - Low

**Status** - Resolved

**Description**

Regarding the documentation, the recoverERC20() function is not supposed to be called while paused.
However, the function is missing a whenNotPaused modifier and it makes the function to be called in the pause status.

**Recommendation** 

Add a check to make sure that the return value is true and if not, revert the function.

## Informational

### Lack of return value check

**Severity** - Informational

**Status** - Resolved

**Description**

The recoverERC20() function is missing a check for the return value of the token’s transfer.

**Recommendation** 

Add a check to make sure that the return value is true and if not, revert the function.


### Floating pragma

**Severity** - Informational

**Status** - Resolved

**Description**

The contract uses pragma solidity ^0.8.27.
This may result in the contracts being deployed using the wrong pragma version, which is different from the one they were tested with.

**Recommendation** 

It is recommended to lock the pragma version.


### Duplicated zero address check

**Severity** - Informational

**Status** - Resolved

**Description**

The constructor() has a zero address check for the _delegate variable passed.
However, _delegate value is already checked in the both Ownable and OAppCore contracts which the GalaxyGames contract inherits from.
(https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol#L40
https://github.com/LayerZero-Labs/devtools/blob/main/packages/oapp-evm/contracts/oapp/OAppCore.sol#L29)

**Recommendation** 

It is recommended to remove the zero address check for the _delegate variable passed.


### Unnecessary use of nonReentrant modifier

**Severity** - Informational

**Status** - Resolved

**Description**

The burn() and recoverERC20() functions have the nonReentrant modifiers but the functions could not have reentrancy attacks because the functions can only be called by the owner and _burn() function doesn’t have an implementation to cause the reetrancy.
Unnecessary use of modifiers will result in unnecessary gas consumption.

**Recommendation** 

Remove the nonReentrant modifier in the functions and delete the ReentrancyGuard import in the contract.

