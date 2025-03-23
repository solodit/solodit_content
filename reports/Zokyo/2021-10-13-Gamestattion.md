**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### The owner will be able to remove the token in the case when the liquidity is not zero and when the owner will try to call the adminWithdraw() for the deleted token it would be  impossible to withdraw that token from the contract balance.

**Re-audit**

The issue related to removing the supported tokens was fixed in the function
removeSupportedToken(). The function adminWithdraw() was deleted.

## Medium Risk

### In the function addSupportedToken() there is no check if the added token is already on the list. There is a possibility to call the function with the same address of the token and rewrite the info about the current token.

### Solidity fileі has no license declaration.

**Recommendation**:

Specify license in every Solidity file.

## Low Risk

### In the constructor, there is no check if the length of the array's tokensToSupport_ tokensInfo_ is the same. It can be the reason for the incorrect behavior.

### In the function withdraw(), addLiquidity(), adminWithdraw() there is no check if the requesting token is supported.

**Recommendation**:

Add checking if the token is supporting:
require(supportedTokens.contains(tokenAddress_), “Token is not supported”)
