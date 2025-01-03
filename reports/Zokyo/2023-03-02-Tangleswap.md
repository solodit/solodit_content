**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Exposed to possible DoS due to gasLimit

**Severity**: High

**Status**: Resolved

**Description**

TangleStaking.sol - in _unstake


For loop in unstake is costly since it includes a storage operation also. The high gas cost is an issue by itself but what makes this highly severe is that it is susceptible to Denial of Service. An investor might be denied to receive his token if he holds too many staked tokens in TangleStaking and seemingly there is no way around it.

**Recommendation**: 

Might be a better option to use enumerable set to represent the token ids in stakedNftById. Or adopt a different operation to remove the token Id if preserving order is unnecessary.

## Medium Risk

### No validation of apy

**Severity**: Medium

**Status**: Resolved

**Description**

In setApys, there is no validation check for the input value. This issue severity is higher than it is usually due to the impact a mistake in new value can have on the tokenomics of the ecosystem.

**Recommendation**: 

require statement to check input new value.


### Centralization Risk

**Severity**: Medium

**Status**: Resolved

**Description**

There are multiple instances where the contract owner can change protocol parameters that affect the behavior of the contracts. These include calls to several functions like updating rewarding details and general setters/mutators. More severely though is the action to with draw ERC20 asset from the contract, this alleviates the concern, hence recommendation on how to deal with it should be followed.

**Recommendation**: 

Consider using multisig wallets or having a governance module

### Use SafeERC20

**Severity**: Medium

**Status**: Resolved

**Description**

It is recommended to utilize SafeERC20 in order to do the transfers of tokens (i.e. as safeTransfer). Devs preferred to utilize transfer wrapped in a require statement which appears safe, but it is not the proper way to deal in case we have "Bad ERC20" that SafeERC20 takes account for in its implementation.

**Recommendation**: 

Refactor the usage of transfer with safeTransfer.

## Low Risk

### ERC721 transfer does not respect check-effect-interact pattern

**Severity**: Low

**Status**: Resolved

**Description**

In _stake and _unstake methods the transfer of nft might represent a re-entrancy risk because of the hook of onERC721Received, it would be best to protect from this possibility by implementing a proper check-effect-interact pattern by making the external calls to the transfer functions as the final call in the contract.


**Recommendation**: 

Refactor to implement a proper check-effect-interact pattern

### Lost ERC20 rewards

**Severity**: Low

**Status**: Resolved

**Description**

In unstakeNft - if totalFunding is less than amount or reward, the investor is worthy to receive. The investor would not receive rewards, which is obvious anyway, but stakeNftRewardInfo record for that investor for the staked NFT is deleted. Hence, no chance for the investor to receive the rewards even later.

## Informational 


### Error objects to save gas

**Severity**: Informational

**Status**: Acknowledged

**Description**

Starting from solidity 0.8.4 (project uses 0.8.7) - solidity developers are recommended to use custom error objects to save gas and also better error information as explained here in solidity blog.

**Recommendation**: 

Use custom errors instead of require statements to revert.

### Limit access modifier

**Severity**: Informational

**Status**: Resolved

**Description**

Most public methods are not invoked internally in contract, hence it is recommended to limit the access modifier to external for those methods.

### No way back for allowList

**Severity**: Informational

**Status**: Resolved

**Description**

In setAllowList - this method sets the allow list on nft contracts to true. On the other hand there is no way in this contract to revert that change and set it to false. This issue can be regarded as a note as in case this feature is intentional the issue shall be irrelevant.

**Recommendation**: 

a method needed to enable admin to remove from allowList 

### Setter applied with no event emitted

**Severity**: Informational

**Status**: Resolved

**Description**

In setAllowList & setApys - according to the general theme of the project, it is noticed that events are emitted after calling the setter methods in this contract. While for setAllowList no event describing this mutation taking place is emitted.

**Recommendation**: 

emit event for setAllowList and setApys.

