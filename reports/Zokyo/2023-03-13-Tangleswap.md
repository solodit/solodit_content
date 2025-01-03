**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Exposed to reentrancy


**Severity**: High

**Status**: Resolved

**Description**

FixRange.sol & DynamicRange.sol - emergencyWithdraw is exposed to reentrancy due to safeTransfer. An attacker can reenter this method by having a external call implemented in his onERC721Received that will call other functions from the same contract context or different contract context.  Other way is to reenter other methods like withdraw or collectReward.  This can be highly severe because the state is not updated before the reentrancy.

**Recommendation**: 

Apply a wide reentrancy guard using best practices pattern and modifiers.


### Set is not updated after emergencyWithdraw

**Severity**: High

**Status**: Resolved

**Description**

FixRange.sol & DynamicRange.sol - emergencyWithdraw does not remove the tokenId from the enumerable set tokenIds. This will lead to several issues, including a Denial of Service issue to collectRewards on the user's other legitimate tokenIds.

**Recommendation**: 

Update tokenIds by removing that token from the enumerable set on the emergencyWithdraw transaction.

## Medium Risk

### Unhandled edge case

**Severity**: Medium

**Status**: Resolved

**Description**

Base.sol - In constructor, in setRewardPool - the case in which token0 = token1 which is believed not to be desired (i.e. according to understanding of the contract). This case is not prevented from occuring and not even considered here, token0 < token1.

**Recommendation** 

add a requirement statement to ensure that token0 != token1.

### Centralization Risk

**Severity**: Medium

**Status**: Resolved

**Description**

Admin enjoys too much authority. The general theme of the repo is that admin has power to call several functions like emergencyWithdraw, general setters/mutators. Some functions can be more highly severe to be left out controlled by one wallet more than other functions; depending on the intentions behind the project. Recommendation Apply governance / use multisig wallets.

## Informational

### Lock Solidity version

**Severity**: Informational

**Status**: Resolved

**Description**

All Contracts, Lock the pragma to a specific version, since not all the EVM compiler versions support all the features, especially the latest one’s which are kind of beta versions, So the intended behavior written in code might not be executed as expected. Locking the pragma helps ensure that contracts do not accidentally get deployed using, for example, the latest compiler, which may have higher risks of undiscovered bugs. Recommendation fix version to 0.8.9



### Unnecessary public view visibility

**Severity**: Informational

**Status**: Resolved

**Description**

In FixRange.sol - tokenStatusLastTouchAccRewardPerShare is unnecessarily public while it can be limited to external. It is one of the best practices to limit the access modifier to what is needed.


### Costly error messages

**Severity**: Informational

**Status**: Acknowledged

**Description**

Error messages in some require statements are long and considered costly in terms of gas.

**Recommendation**:

Write shorter error messages

Use custom errors which is the goto choice for developers since solidity v0.8.4 details about this shown here: soliditylang.


### Transactions can be excluded due to block gas limit while calling collectAllTokens()

**Severity**: Informational

**Status**: Resolved

**Description**

In the contract DynamicRange, the loop in line 396 iterates over all the tokens owned by the user and collects the rewards. This for loop has no limit on the number of interactions. If the user owns a large number of tokens and calls the functions, the transaction can possibly exceed the block gas limit. Thus, the transaction will never be included in the block. 

**Recommendation**: 

Users can alternatively call collect function for each token individually. Limit the number of iterations in the loop by adding from and to parameters to the function call. Using these 2 parameters, the user can decide the range in the array for which the tokens should be collected. 
**Note #1**: This issue was fixed by providing a method called collect that allows user to collect pending reward one NFT at a time, this covers a majority of use case.
Second, the client also have  the intention to put contraints on the frontend to collect no greater than NFT rewards at a time, to mitigate the out of gas risk. Only those who do not respect the constraints on the frontend will run into “out of gas” problem by invoking scripts to collect large batch of NFTs’ rewards will get into this problem.

