**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Low Risk

### [L-01] Using an `immutable` value as a gas limit might be dangerous

The `pluginCallGasLimit` is a value that is set in the constructor of `ERC20Plugins` and is `immutable`. This value is used to cap the gas allowed for an external low-level call to use. The problem with this is that previous Ethereum hardforks have changed gas costs of commonly used opcodes (for example with `EIP-150`) - this happening again can result in the `pluginCallGasLimit` value being insufficient to execute the operations needed by the contract. Consider making the value mutable or removing it altogether.

### [L-02] It's possible to use a flawed compiler version

Solidity version 0.8.13 & 0.8.14 have a security vulnerability related to assembly blocks that write to memory, which are present in `ERC20Plugins` contract. The issue is fixed in version 0.8.15 and is explained [here](https://soliditylang.org/blog/2022/06/15/solidity-0.8.15-release-announcement/). While the problem is with the legacy optimizer (not the yul IR pipeline that you are using), it is still correct to enforce latest Solidity version (0.8.21) or at least the one enforced in other popular library code as OpenZeppelin (0.8.19). Applies for the whole codebase.

```diff
-pragma solidity ^0.8.0;
+pragma solidity ^0.8.19;
```