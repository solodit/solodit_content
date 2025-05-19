**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Organisation record can be re-written

**Description**

MGD.sol, createOrganisation()
Bool flag for the organisation name taken (orgTaken mapping) is never set. Thus, it makes
orgTaken mapping useless. The issue is marked as Medium only because the organisation
creation can be performed only by admin.

**Recommendation**:

Review the functionality, remove unused mapping or add storage setter.

### Use fixed version of Solidity

**Description**

The project utilises Solidity ^0.8.0 version. Though, the standard auditor s checklist states for
the usage of the fixed version of the Solidity. So it is preferable to use the 0.8.6 version of
Solidity (the latest stable release).

**Recommendation**:

Use a fixed version of the Solidity (e.g. the latest stable release 0.8.6).

### Unused mapping

**Description**

flaggedNFTS (contracts/MGD.sol#19)
The mapping is never used, though is kept on the contract and can be set through the setter.
The comment states, that the mapping is used by the frontend app, though its usage should
be verified.

**Recommendation**:

Verify the functionality.

### Explicitly mark visibility of state

**Description**

contracts/OrderBook.sol#20, withdrawAddress
The storage variable has no visibility set. Such an issue is included to the standard auditor s
checklist. Missed visibility may lead to incorrect interaction with the smart contract.

**Recommendation**:
Explicitly mark visibility of state.

## Low Risk

### Missing zero check during initialization

**Description**

contracts/OrderBook.sol#48, withdrawAddress
The variable is actively used in the contract, though is not checked against the zero address.
Since there is no ability to change the value and since the initialization is performed only in the
initializer the issue is classified as Low.

**Recommendation**:

Add check that the address is not zero.

### Provide an error message for require statements

**Description**

contracts/MGD.sol
Lines: 61, 77, 271, 312, 313, 395, 409, 410, 421, 422, 433, 434
contracts/OrderBook.sol
Lines: 52, 97
All mentioned require statements do not contain error messages. Such an approach affects
the further contract functioning, since any troubleshooting will require a lot of additional
actions.

**Recommendation**:

Provide error messages for required statements.

### Compares to a boolean constant

**Description**

contracts/MGD.sol#240:
flaggedNFTS[_tokenId] == true
contracts/MGD.sol#52 55:
require(protocolAdmins[_msgSender()] == true)
contracts/MGD.sol#61 64
require(_org.orgAdmin[_msgSender()] == true || protocolAdmins[_msgSender()] == true)
contracts/OrderBook.sol#209
require(order.orderValid == true,Orderbook ERR: order not valid)
Boolean constants can be used directly and do not need to be compared to true or false. Thus
it will perform a minimal gas usage optimization.

**Recommendation**:

Remove comparison to the bool constant and use the statement directly.

## Informational

### Event invocations have to be prefixed by emit

**Description**

contracts/OrderBook.sol#257
It is recommended to explicitly use emit keyword for the event to distinguish the functions
calls and events emitting per recommendation from the Solidity checklist.

**Recommendation**:

Event invocations have to be prefixed by emit . Add prefixed by emit .

### “Magic" numbers

**Description**

contracts/MGD.sol#213, updateMetadata()
contracts/MGD.sol#360, royaltiInfo()
Variables should be set as constants with proper naming and documentation. That will
increase the code readability and give a chance to verify the functionality in case these
numbers should be adjusted.

**Recommendation**:

Move variables to constants.

### Use an interface instead of an abstract contract

**Description**

For now MGDcontract.sol has no functional value as an abstract contract. Consider usage of
the interface. Such a move will save gas and decrease the contract size.

**Recommendation**:

Use an interface instead of an abstract contract.

### Extra variable may be omitted

**Description**

contracts/MGD.sol#432, setListingFee()
contracts/MGD.sol#420, setMintFee()
contracts/MGD.sol#408, unlockPlatform()
Extra variable mgdCon may be omitted as unnecessary.

**Recommendation**:

Remove extra variable.

### pausePlatform() have confusing naming

**Description**

contracts/MGD.sol#394, pausePlatform()
The function is named as a pausable function, though it works as a trigger function, because it
changes the state based on the previous value. Consider splitting it into 2 functions (pause()
and unpause()) or renaming e.g. triggerPause().

**Recommendation**:

Review the functionality, consider function renaming or split, correct the docstring for the
function. Also consider usage of the standard OpenZeppelin Pausable contract.

### Useless variable

**Description**

contracts/MGD.sol#344, royaltyInfo() _data
_data the variable is not written anywhere. The function receives and sends _data unchanged
or written.

**Recommendation**:

Revise the need for it, keep it in the contract, or remove it from the project.
