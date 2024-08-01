**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Informational

### Unused imports

**Severity**: Informational

**Status**: Resolved

In Contract ChainZoomToken.sol, the following imports are unused.
```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
```
ERC20 is already available from the import ERC20Permit.sol
Ownable.sol is not needed as onlyOwnable modifier is not used anywhere.

**Recommendation**: remove the unused imports and modify the contract as follows.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;


import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";


contract ChainZoomToken is ERC20Permit {
   constructor() ERC20("ChainZoom", "ZOOM") ERC20Permit("ChainZoom") {
       _mint(msg.sender, 200000000 * 10 ** decimals());
   }
}
```
### Floating Pragma Version in Solidity Files

**Severity**: Informational

**Status**: Resolved

**Description**

The Solidity files in this codebase contain pragma statements specifying the compiler version in a floating manner (e.g., ^0.8.20 ). Floating pragmas can introduce unpredictability in contract behavior, as they allow the automatic adoption of newer compiler versions that may introduce breaking changes.

**Recommendation**: 

To ensure stability and predictability in contract behavior, it is recommended to: Specify a Fixed Compiler Version: Instead of using a floating pragma, specify a fixed compiler version to ensure consistency across different deployments and prevent the automatic adoption of potentially incompatible compiler versions.
