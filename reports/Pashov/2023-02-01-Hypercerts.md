**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## High Risk

### [C-01] Users can split a token to more fractions than the `units` held at `tokenID`

**Impact**
High, as it breaks an important protocol invariant

**Likelihood**
High, as those types of issues are common and are easily exploitable

**Description**

The `_splitValue` method in `SemiFungible1155` does not follow the Checks-Effects-Interactions pattern and it calls `_mintBatch` from the ERC1155 implementation of OpenZeppelin which will actually do a hook call to the recipient account as a safety check. This call is unsafe as it can reenter the `_splitValue` method and since `tokenValues[_tokenID]` hasn't been updated yet, it can once again split the tokens into more fractions and then repeat until a huge amount of tokens get minted.

**Recommendation**

Follow the Checks-Effects-Interactions pattern

```diff
-_mintBatch(_account, toIDs, amounts, "");
-
-tokenValues[_tokenID] = valueLeft;
+tokenValues[_tokenID] = valueLeft;
+
+_mintBatch(_account, toIDs, amounts, "");
```

## High Risk

### [H-01] Calling `splitValue` when token index is not the latest will overwrite other claims' storage

**Impact:**
High, as it can lead to loss of units for an account without any action on his side

**Likelihood:**
Medium, because it can happen only with a token that has a non-latest index

**Description**

The logic in `_splitValue` is flawed here:

```solidity
uint256 currentID = _tokenID;
...
toIDs[i] = ++currentID;
...
for (uint256 i; i < len; ) {
    valueLeft -= values[i];

    tokenValues[toIDs[i]] = values[i];

    unchecked {
        ++i;
    }
}
...
_mintBatch(_account, toIDs, amounts, "");
```

Let's look at the following scenario:

1. Alice mints through allowlist, token 1, 10 units
2. Bob mints through allowlist, token 2, 100 units
3. Alice calls `splitValue` for token 1 to 2 new tokens, both 5 units

Now we will have `tokenValues[toIDs[i]] = values[i]` where `toIDs[i]` is `++currentID` which is 2 and `values[i]` is 5, so now `tokenValues[2] = 5` which is overwriting the `tokenValues` of Bob. Also, later `_mintBatch` is called with Bob's token ID as a token ID, which will make some of the split tokens be of the type of Bob's token.

**Recommendations**

Change the code the following way:

```diff
- maxIndex[_typeID] += len;
...
- toIDs[i] = ++currentID;
+ toIDs[i] = _typeID + ++maxIndex[typeID];
```

## Medium Risk

### [M-01] Unused function parameters can lead to false assumptions on user side

**Impact:**
Low, because the caller still controls the minted token

**Likelihood:**
High, because users will provide values to those parameters and their assumptions about their usage will always be false

**Description**

The `units` parameter in `mintClaimWithFractions` is used only in the event emission. This is misleading as actually `fractions.length` number of fractions will be minted. If `units != fractions.length` this can have unexpected consequences for a user. The same is the problem with the `account` parameter in both `mintClaim` and `mintClaimWithFractions` - it is not used in the method and actually `msg.sender` is the account to which tokens are minted and is set as the token creator. Again if `account != msg.sender` this is unexpected from a user standpoint and while in the best case scenario leads to a not so great UX, in the worst case it can lead to faulty assumptions for value received by the `account` address.

**Recommendations**

Remove the `units` parameter from `mintClaimWithFractions` and also use `account` instead of `msg.sender` in the `_mintValue` call in `mintClaim` and `mintClaimWithFractions`.

### [M-02] Input & data validation is missing or incomplete

**Impact:**
High, because in some cases this can lead to DoS and unexpected behaviour

**Likelihood:**
Low, as it requires malicious user or a big error on the user side

**Description**

Multiple methods are missing input/data validation or it is incomplete.

1. The `splitValue` method in `SemiFungible1155` has the `maxIndex[_typeID] += len;` code so should also do a `notMaxItem` check for the index
2. The `createAllowlist` method accepts a `units` argument which should be the maximum units mintable through the allowlist - this should be enforced with a check on minting claims from allowlist
3. The `_createAllowlist` of `AllowlistMinter` should revert when `merkleRoot == ""`
4. The `_mintClaim` and `_batchMintClaims` methods in `SemiFungible1155` should revert when `_units == 0` or `_units[i] == 0` respectively.

**Recommendations**

Add the checks mentioned for all inputs and logic.

## Low Risk

### [L-01] Comment has incorrect and possible dangerous assumptions

The comment in the end of `SemiFungible1155` assumes constant values are saved in the contract storage which is not the case. Both constants and immutable values are not stored in contract storage. Update comment and make sure to understand the workings of smart contracts storage so it does not lead to problems in upgrading the contracts in a later stage.

### [L-02] Missing event and incorrect event argument

The `_mergeValue` method in `SemiFungible1155` does not emit an event while `_splitValue` does - consider emitting one for off-chain monitoring. Also the `TransferSingle` event emission in `_createTokenType` has a value of 1 for the `amount` argument but it does not actually transfer or mint a token so the value should be 0.

### [L-03] Prefer two-step pattern for role transfers

The `Upgradeable1155` contract inherits from `OwnableUpgradeable` which uses a single-step pattern for ownership transfer. It is a best practice to use two-step ownership transfer pattern, meaning ownership transfer gets to a "pending" state and the new owner should claim his new rights, otherwise the old owner still has control of the contract. Consider using a two-step approach.

### [L-04] Contracts pausability and upgradeability should be behind multi-sig or governance account

A compromised or a malicious owner can call `pause` and then `renounceOwnership` to execute a DoS attack on the protocol based on pausability. The problem has an even wider attack surface with upgradeability - the owner can upgrade the contracts with arbitrary code at any time. I suggest using a multi-sig or governance as the protocol owner after the contracts have been live for a while or using a Timelock smart contract.

## Informational

### [I-01] Transfer hook is not needed in current code

The `_beforeTokenTransfer` hook in `SemiFungible1155` is not needed as it only checks if a base type token is getting transferred but the system in its current state does not actually ever mint such a token, so this check is not needed and only wastes gas. Remove the `_beforeTokenTransfer` hook override in `SemiFungible1155`.

### [I-02] Unused import, local variable and custom errors

The `IERC1155ReceiverUpgradeable` import in `SemiFungible1155` is not actually used and should be removed, same for the `_typeID` local variable in `SemiFungible1155::mergeValue`, same for the `ToZeroAddress`, `MaxValue` and `FractionalBurn` custom errors.

### [I-03] Merge logic into one smart contract instead of using inheritance

The `SemiFungible1155` contract inherits from `Upgradeable1155` but it doesn't make sense to separate those two since the logic is very coupled and `Upgradeable1155` won't be inherited from other contracts so it does not need its own abstraction. Merge the two contracts into one and give it a good name as the currently used two names are too close to the `ERC1155` standard.

### [I-04] Incorrect custom error thrown

The code in `AllowlistMinter::_processClaim` throws `DuplicateEntry` when a leaf has been claimed already - throw an `AlreadyClaimed` custom error as well. Also consider renaming the `node` local variable there to `leaf`.

### [I-05] Typos in comments

`AlloslistMinter` -> `AllowlistMinter`

### [I-06] Missing `override` keyword for interface inherited methods

The `HypercertMinter` contract is inheriting the `IHypercertToken` interface and implements its methods but the `override` keyword is missing on the overriden methods. Add the keyword on those as a best practice and for compiler checks.

### [I-07] Bit-shift operations are unnecessarily complex

I recommend the following change for simplicity:

```diff
-    uint256 internal constant TYPE_MASK = uint256(uint128(int128(~0))) << 128;
+    uint256 internal constant TYPE_MASK = type(uint256).max << 128;

    /// @dev Bitmask used to expose only lower 128 bits of uint256
-    uint256 internal constant NF_INDEX_MASK = uint128(int128(~0));
+    uint256 internal constant NF_INDEX_MASK = type(uint256).max >> 128;
```

### [I-08] Redundant check in code

The `_beforeValueTransfer` hook in `SemiFungible1155` has the `getBaseType(_to) > 0` check which always evaluates to `true` so it can be removed.

### [I-09] Incomplete or wrong NatSpec docs

The NatSpec docs on the external methods are incomplete - missing `@param`, `@return` and other descriptive documentation about the protocol functionality. Also part of the NatSpec of `IAllowlist` is copy-pasted from `IHypercertToken`, same for `AllowlistMinter`. Make sure to write descriptive docs for each method and contract which will help users, developers and auditors.

### [I-10] Misleading variable name

In the methods `mintClaim`, `mintClaimWithFractions` and `createAllowlist` the local variable `claimID` has a misleading name as it actually holds `typeID` value - rename it to `typeID` in the three methods.

### [I-11] Solidity safe pragma best practices are not used

Always use a stable pragma to be certain that you deterministically compile the Solidity code to the same bytecode every time. The project is currently using a floatable version.

### [I-12] Magic numbers in the codebase

In `SemiFungible1155` we have the following code:

```solidity
if (_values.length > 253 || _values.length < 2) revert Errors.ArraySize();
```

Extract the `253` value to a well-named constant so the intention of the number in the code is clear.
