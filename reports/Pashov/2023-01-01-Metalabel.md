**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Medium Risk

### [M-01] Insufficient input data validation for `configureSequence`

**Impact:**
High, as some of those can result in a DoS or too big of a royalty payment

**Likelihood:**
Low, as it requires a configuration error or a malicious actor

**Description**

An authorized address for a node can call `Collection::configureSequence` where most of the input is not validated properly. The `_sequence` parameter of the method is of type `SequenceData` which fields are not validated. Missing checks are the following:

1.  `sealedBeforeTimestamp` is bigger than `block.timestamp`
2.  `sealedAfterTimestamp` is always bigger than `sealedBeforeTimestamp`
3.  The difference between `sealedBeforeTimestamp` and `sealedAfterTimestamp` is at least `2 days ` for example
4.  The difference between `sealedBeforeTimestamp` and `sealedAfterTimestamp` is not more than `500 days` for example
5.  The difference between `sealedBeforeTimestamp` and `block.timestamp` is not more than `10 days` for example

Also in `DropEngine::configureSequence` the `royaltyBps` is not validated that it is not more than 100% (a value of 10000). I suggest you add a lower royaltyBps upper bound.

**Recommendations**

Add sensible constraints and validations for all user input mentioned above.

### [M-02] Using the `transfer` function of `address payable` is discouraged

**Impact:**
Medium, as sequence won't be usable as mints will revert

**Likelihood:**
Medium, as it happens any time the recipient is a smart contract or a multisig wallet that has a receive function taking up more than 2300 gas

**Description**

The `mint` function in `DropEngine` uses the `transfer` method of `address payable` to transfer native asset funds to an address. This address is set by a node owner and is possible to be a smart contract that has a `receive` or `fallback` function that takes up more than the 2300 gas which is the limit of `transfer`. Examples are some smart contract wallets or multi-sig wallets, so usage of `transfer` is discouraged.

**Recommendations**

Use a `call` with value instead of `transfer`. There is no risk from reentrancy in the `mint` method as it has a check for the caller to be an EOA. When this is done you can remove the `payable` keyword from the `revenueRecipient` variable.

### [M-03] Role transfer actions done in a single-step manner are dangerous

**Impact:**
High, as important protocol functionality would become unusable

**Likelihood:**
Low, as it requires an admin/owner error

**Description**

This is a common problem where transferring a role or admin rights to a different address can go wrong if this address is wrong and not actually controlled by any user. This is taken into consideration in `NodeRegistry` where the node ownership transfer is a two-step operation. Not the same approach is used in `AccountRegistry` though, where the contract inherits from `Owned` which has a single-step ownership transfer pattern and also the `transferAccount` logic in it is also using a single-step pattern.

**Recommendations**

Use a two-step ownership/rights transfer pattern in both the `AccountRegistry` ownership and in the `transferAccount` method, you can reuse the approach you used in `NodeRegistry`.

### [M-04] Records minted to an address that is a smart contract that can't handle ERC721 tokens will be stuck forever

**Impact:**
High, as records will be stuck forever

**Likelihood:**
Low, as it requires the engine to allow smart contracts as minters and that contracts should not support handling of ERC721 tokens

**Description**

Both `mintRecord` methods in `Collection` use the `_mint` method of `ERC721` which is missing a check if the recipient is a smart contract that can actually handle ERC721 tokens. If the case is that the recipient can't handle ERC721 tokens then they will be stuck forever. For this particular problem the `safe` methods were added to the ERC721 standard and `Solmate` has added the `_safeMint` method to check handle this problem in a minting context. This is actually not a problem in `DropEngine` because it allows only EOAs to mint, but since users can freely implement Engines then this is a valid problem.

**Recommendations**

Prefer using `_safeMint` over `_mint` for ERC721 tokens, but do this very carefully, because this opens up a reentrancy attack vector. It's best to add a `nonReentrant` modifier in the method that is calling `_safeMint` because of this.

## Low Risk

### [L-01] `resolveId` will return the 0 ID even if an address does not have an account

This is error prone and requires all clients of the `resolveId` function to do checks for `!= 0`, as we can see in `NodeRegistry` where this method is called. A better and safer approach is to just revert in the `resolveId` method when the `subject` does not have an account, then if the client of the method wants to catch the error he can do this.

### [L-02] Inconsistent `tokenData.owner` validation

The `mintRecord` functionality in `Collection` allows the caller to mint tokens to any address since the `to` argument is not validated in any way. The problem is that the `getTokenData` inherited function has the following check:

```solidity
require(data.owner != address(0), "NOT_MINTED");
```

which is not actually true, because a token could be minted to the zero address, even due to a mistake. I suggest to add zero-address checks for the `to` argument in both `mintRecord` functions in `Collection` for consistency.

## Informational

### [I-01] Use a more abstract name for function parameter

The `broadcast` and `broadcastAndStore` methods in `ResourceFactory` have a `waterfall` parameter but some other type of resource might be used - for example a Collection or a Split. Rename parameter from `waterfall` to `resource` in both methods.

### [I-02] `NodeData.nodeType` is used as an existence check and nothing else

The only thing `nodeType` is used for in the system is to check if a node exists. This can be done with a simple boolean or if you actually need a "type" then use an enum where the first value is `NON_EXISTENT` for example.

### [I-03] NatSpec docs are incomplete

In almost all methods the NatSpec documentation is incomplete as it is missing parameters and return variables in it. NatSpec documentation is essential for better understanding of the code by developers and auditors and is strongly recommended. Please refer to the [NatSpec format](https://docs.soliditylang.org/en/v0.8.17/natspec-format.html) and follow the guidelines outlined there.

### [I-04] Unused import

The `IERC721` import in `Collection` is not used. Remove it from the code.

### [I-05] Copy-pasted comments in `WaterfallFactory`

Almost all methods reference a `split` instead of a `waterfall` in the contract, I expect they were copy-pasted from `SplitFactory`. Replace `split` with `waterfall` in all of the comments in `WaterfallFactory`.

### [I-06] Missing `override` keyword

Most methods in contracts that are implemented and inherited from an interface are missing the `override` keyword. Go through each method and apply the keyword where it is right to do so. Some examples are the `mintRecord` and `royaltyInfo` methods.

### [I-07] Use a safe pragma statement

Always use stable pragma statement to lock the compiler version and to have deterministic compilation to bytecode. Keep in mind `0.8.13` has a bug when using assembly, even though the codebase does not do that.

### [I-08] Typos and grammatical errors

A number of typos in the codebase that need to be fixed:

`AccountTransfered` -> `AccountTransferred`

`has be` -> `has been`

`Inheritting` -> `Inheriting`

`Otherise` -> `Otherwise`

`managaeable` -> `manageable`

`so long as their is` -> `as long as there is`

`entites` -> `entities`

`Otherise` -> `Otherwise`

`additonal` -> `additional`

`seqeuence` -> `sequence`

`funcionality` -> `functionality`

`reliquish` -> `relinquish`
