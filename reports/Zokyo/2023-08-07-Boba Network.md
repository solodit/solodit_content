**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Using oracle method that could return 0
**Description**

In contract Boba Deposit Paymaster, function "getTokenValueOfEth" is translating a given value amount of ethereum into the underlying token amount, for that the business logic is using oracles to query for values and different token decimals, oracles will be chosen by the admin of the contract however we expect that most preferred oracles will be BobaStraw, as BobaStraw is a fork of chainlink as it is saying in it's documentation, we noticed that the used function to query the oracles is the function "latestAnswer" which based on it's implementation does not revert if the price returned it's staled or if it can not retrieve the price it will simply return 0 which will break all the logic of the application resulting in a token cost of and and free transactions.

**Recommendation**

Use function "latestRoundData" instead of "latestAnswer" and ensure you are receiving fresh data from it by sanity checking updatedAt and answeredInRound returned values.

**Re-audit comment**

Resolved

## Low Risk

### Method could return 0 based on oracles returned decimals feeds
**Description**

In contract Boba Deposit Paymaster, function "getTokenValueOfEth" is translating a given value amount of ethereum into the underlying token amount, for that the business logic is using oracles to query for values and different token decimals, oracles will be chosen by the admin of the contract, based on the function comments there is no check over the variable requiredAmount to ensure it's not 0 as priceRatio shouldn't normally exceed ethBought value, that statement is not always true and it is dependent of the values returned by the oracles, as the oracles are chosen by the owner, there is the possibility that one oracles will returned a price feed on 8 decimals and another one on 18 decimals and so on, and that will ruin the assumption that required Amount can not be 0.

**Recommendation**

Add a sanity check to ensure that the function can not return the value at the end.

**Re-audit comment**

Resolved

## Informational

### Floating Pragma
**Description**

Throughout the codebase, the contracts that are unlocked at version ^0.8.12, and they should always be deployed with the same compiler version. By locking the pragma to a specific version, contracts are not accidentally getting deployed by using an outdated version that can introduce unintended consequences.

**Recommendation**

Lock the compiler version to a specific. Known bugs are featured here.

**Re-audit comment**

Acknowledged.
Comment: The client acknowledges the finding, but did not make any changes.

### Outdated Dependencies
**Description**

The codebase is using an outdated Open Zeppelin version "@openzeppelin/contracts": "^4.2.0". Below are issues tied to outdated OZ versions. This lists errors such as DOS, improper initialization, etc.

**Recommendation**

Update Open Zeppelin dependencies to a more recent and secure version.

**Re-audit comment**

Resolved

### `getUserOpHashes()` Can Cause Unexpected Behavior
**Description**

In the contract, 'EntryPointWrapper.sol', 'getUserOpHashes() on line 196 may cause unexpected behavior. If the wrong 'entryPoint address is input to the function parameters, then it will revert. There could be a check to ensure the right EntryPoint is used or removed from the parameters. Even if it is implied for one contract per chain, this might lower risk of accidental error.

**Recommendation**

1. Add a simple check to ensure the right entrypoint is used to ensure all hashes are the same. As the return value will hash something else with the wrong 'Entrypoint address. If this check is supposed to be off-chain, then proper documentation should be completed for developers to understand the importance.
2. Remove `IEntryPoint entryPoint from the function parameter as it is already defined inside the constructor.

**Re-audit comment**

Acknowledged.
Comment: The client acknowledges the finding but did not make any changes as there will only be one contract.

### Lack of NatSpec in 'EntryPointWrapper.sol'
**Description**

In the contract, EntryPointWrapper.sol', the codebase lacks documentation that might be useful to developers. For example: documentation for structs such as 'FailOpStatus` might be useful for developers without having to guess the intended functionality.

**Recommendation**

We recommend that documentation is added throughout EntryPointWrapper.sol to ensure external reviewers and developers understand intended meaning.

**Re-audit comment**

Resolved

### Bridge Cannot Handle ERC-20 Fee on Transfer
**Description**

In the contract Teleportation.sol is used to transfer funds from L1 to L2. The function `teleportAsset()` sends and emits tokens based on the input of _amount'. If a token had a fee attached on transfer the contract can expect potential unexpected consequences if the fee is not exempt for this contract.

**Recommendation**

We recommend checking the balance received by the contract before updating/or emitting a value that is potentially incorrect. This additional logic will allow fee on transfer tokens to be bridged across without users losing funds.

**Re-audit comment**

Resolved.
Comment: The client does not expect this to be an issue as they will not support fee on transfer tokens.

### SafeMath Not Needed
**Description**

In the contract, 'Teleportation.sol', the codebase uses SafeMath. This causes unnecessary overflow and underflow checks that are reverted by default. By removing this library and using built in arithmetic, users can save gas for not unnecessary checks.

**Recommendation**

We recommend removing SafeMath as the compiler version is above 0.8.0.

**Re-audit comment**

Resolved

### `sourceChainId` Can Be Converted to Save One Slot
**Description**

In the contract, Teleportation.sol', the variable 'sourceChainId can be converted to save gas. 'sourceChainId may benefit from being reduced from uint256 to smaller size such as uint32. By changing sourceChainId', the slot size will be lowered by one.

**Recommendation**

If converting 'sourceChainId to uint32 is within the protocol specification, then we recommend making this change.

**Re-audit comment**

Resolved
