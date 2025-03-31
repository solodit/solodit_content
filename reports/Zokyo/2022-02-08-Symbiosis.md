**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### In contract MetaRouterV2, function metaRouteV2, lines 87 & 67, there are some external calls to an arbitrary address with an arbitrary data, an malicious actor could carefully craft the input arguments to call transferFrom function from an ERC20 that was previously approved by the user.

**Recommendation**:
Whitelist all the addresses that metaRouter will do external calls to (firstDexRouter,
secondDexRouter, relayRecipient).

### In contract MetaRouterV2, function metaMintSwap, line 106 & 129, there is an external call to an arbitrary address with an arbitrary data, made through the swap function, an malicious actor could carefully craft the input arguments to call transferFrom function from an ERC20 that was previously approved by the user.

**Recommendation**:
Whitelist the address _router from the _swap function.

## Medium Risk

### In contract Synthesis, line 138, getSyntRepresentation function could in some cases return the zero address.

**Recommendation**:
Add a checker using the require function to check for the zero address.

### In contract Synthesis, line 171, getSyntRepresentation function could in some cases return the zero address.

**Recommendation**:
Add a checker using the require function to check for the zero address.

## Low Risk

### In contract Wrapper, you have both a deposit function and the receive functionality active, the deposit function is a payable function used for sending native tokens(ex: eth) and receive/mint the wrapped erc20 version (ex: weth), the problems arise when some users may be inclined to send the native tokens directly to the contract and not through the deposit function, if they do that they will not receive any wrapped tokens.

**Recommendation**:
Eliminate the receive() function.

## Informational

### In contract Portal at lines 84 & 89, the require functions does not contain a revert message.

**Recommendation**:
Add a revert message in the require functions at the specified lines.

### In contract SyntFabric at line 80, the require function does not contain a revert message.

**Recommendation**:
Consider updating to “pragma solidity 0.8.11;”.

### In contract Synthesis at line 32, the require function does not contain a revert message.

**Recommendation**:
Add a revert message in the require function at the specified line.

### In contract Synthesis at line 465, the require function does not contain a revert message.

**Recommendation**:
Add a revert message in the require function at the specified line.
