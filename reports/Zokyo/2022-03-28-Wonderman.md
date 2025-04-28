**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### The WondermanToken contract have a pragma version of ^0.8.0, that means it can be compiled with any compiler bigger then 0.8.0, however the openzeppelin contracts that it inherits also have ^0.8.0 expect the Address contract which have the pragma version of ^0.8.1, because the solidity compiler in the truffle.js is also declared as ^0.8.0, if the version used in compilation will be 0.8.1 the compilation will fail because of the Address contracts that gets inherited.

**Recommendation**:

Change the pragma version of the WondermanToken to ^0.8.1 or a fixed bigger version like
0.8.3 etc..

### There is a typo in the contract name, the contract file is called WondermaToken.sol and the contract is called WonermanToken, because there is only 1 letter difference we assume it’s a typo, also it’s best practice to name the contract the file with the same name.

**Recommendation**:

Change the contract name from WondermanToken to WondermanToken.
