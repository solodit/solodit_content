**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Medium Risk

### [M-01] The protection check for `maxRecordsPerTransaction` can be gamed

**Impact:**
Medium, because a protocol invariant can be broken and the code gives a false sense of security

**Likelihood:**
Medium, because the attack is easy to do and we have seen such attacks in the past

**Description**

Let's look at the following example scenario:

1. Collection creates a drop where `maxRecordsPerTransaction` is 1 and total supply is 10
2. It is expected that many people will be able to mint
3. A malicious actor writes a script that loads different wallets with enough value and bundles transactions for 10 mints
4. Only the malicious actor minted, no one else did

Even though there was some kind of a protection against bots/snipers the result was still that only 1 account got to minting.

**Recommendations**

Document that the `maxRecordsPerTransaction` check does not protect the protocol from sniping attacks. To protect from them you can decide to use an off-chain process for pre-registrations of addresses that will be put into a Merkle tree and then validated on `mint`.

### [M-02] Insufficient input validation opens up multiple attack vectors

**Impact:**
High, as it can overflow a balance and re-mint burned NFTs

**Likelihood:**
Low, as it requires a malicious/compromised owner account or an owner input error

**Description**

The `adminTransferFrom` method does not validate that the `from` argument shouldn't have a value of `address(0)`. Now if `from == address(0)` multiple attack vectors open:

1. Burned NFTs can be "re-minted", and also that happens without changing `totalSupply`
2. If `from` balance is zero, this will underflow `_balanceOf[from]`
3. The 0 ID token (which shouldn't exist) can be "minted" with this method

**Recommendations**

Add a check and assert that the `from` argument is not `address(0)`.

### [M-03] Owner can front-run sequence configurations by setting fee to 100%

**Impact:**
High, as if it goes unnoticed it can rug the `revenueRecipient` address

**Likelihood:**
Low, as it requires a malicious/compromised owner account

**Description**

The `setPrimarySaleFeeBps` is callable at any time by the contract owner address and will update the fee variable immediately. Now if a user is trying to call `configureSequence`, the owner can front-run the user call, update the fee to 100% and since there is this code in `configureSequence`

```solidity
dropData.primarySaleFeeBps = primarySaleFeeBps;
```

Now the whole mint payment for this sequence drop will go to the contract owner. He can also execute this attack and front-run each `configureSequence` call to get all mints' ETH value.

**Recommendations**

Since the user provides `dropData.primarySaleFeeBps`, check that he expected the same fee as the one that is currently set in `DropEngineV2` and if the current one is bigger revert the transaction. Also it is generally recommended to not allow fee to go up to 100% - lower the upper limit to a sensible number.

## Low Risk

### [L-01] User can mint, burn and then re-mint his `Memberships` NFT

The `mintMemberships` method allows multiple reuses of the same Merkle tree leaf, which means a user can mint, then burn, then mint again. This way he can spam events and also increase `totalMinted` and `totalSupply`. Add a check that forbids reuse of the same leaf in the Merkle tree.

### [L-02] Anyone can mint your membership for you

The `mintMembership` method uses a Merkle tree that has the `mints[i].to` value in the leaf, instead of `msg.sender` - this means anyone can mint your membership for you. This means any user can influence the ID of the NFT which might not be desired. Prefer using `msg.sender` instead of a user-supplied value in the leaf generation.

### [L-03] No upper limit validation on user-supplied values

The `decayStopTimestamp` and `priceDecayPerDay` properties of `DropData` in `DropEngineV2` do not have an upper limit validation. If too big values are set (due to an error for example) this can DoS the minting process. Add sensible upper limits for both values.

### [L-04] Merkle tree leaf generation is single-hashed and might lead to a second preimage attack if code changes

Merkle trees whose leafs are just single-hashed are vulnerable to [second preimage attack](https://flawed.net.nz/2018/02/21/attacking-merkle-trees-with-a-second-preimage-attack/). The correct way is to double-hash them as [OpenZeppelin suggests](https://github.com/OpenZeppelin/merkle-tree#standard-merkle-trees). The problem exists in both `ControllerV1` and in `Memberships` but in the latter the transaction would revert because of the max 1 balance check and in the former it will just setup a new Metalabel, but it can mess with the `subdomains` mapping.

## Informational

### [I-01] Duplicated custom error

The `NotAuthorized` custom error is duplicated throughout the codebase. Declare it once and reuse it, same for duplicated interface which should be extracted in separate files.

### [I-02] NatSpecs are incomplete

@param and @return fields are missing throughout the codebase. NatSpec documentation is essential for better understanding of the code by developers and auditors and is strongly recommended. Please refer to the [NatSpec format](https://docs.soliditylang.org/en/v0.8.17/natspec-format.html) and follow the guidelines outlined there. Also the NatSpec of `createMemberships` is incorrect, same for `RevenueModuleFactory`.

### [I-03] Redundant code

The following imports are unused and can be removed `DropEngineV2` and `DropData` in `ControllerV1`, `MerkleProofLib` in `Memberships`. The `onlyAuthorized` modifier in `_mintAndBurn` can be removed because the functions that are calling this method already have it. Also `baseTokenURIs` and `mintAuthorities` are unused storage variables in `DropEngineV2` and should be removed.

### [I-04] Typos and grammatical errors in the comments

`Information provided with publishing a new release` -> Information provided when publishing a new release

`Lanch` -> `Launch`

`offchain` -> `off-chain`

`inheritted` -> `inherited`

`contorller` -> `controller`

`Tranfer` -> `Transfer`

`addidtional` -> `additional`

`iniitalization` -> `initialization`

`additonal` -> `additional`

`A an` -> `An`

`Admin can use a merkle root to set a large list of memberships that can /// minted by anyone with a valid proof to socialize gas` -> `that can be minted`

`Token URI computation defaults to baseURI + tokenID, but can be modified /// by a future external metadata resolver contract that implements IEngine` -> `ICustomMetadataResolver` instead of `IEngine`

`having to a separate storage` -> `having to do a separate storage`

`programatically` -> `programmatically`

`psuedo` -> `pseudo`

### [I-05] Missing `override` keyword

`DropEngineV2` inherits `configureSequence` method from `IEngine` but is missing the `override` keyword.
