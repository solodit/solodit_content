**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### In contract "Controller", in function "addRegistrar", at lines 65 and 67, "validators.length" was used instead of "registrars.length".This leads to improper deletion of some values from the "registrars".

**Recommendation**:

Through iteration from line 65, replace “validators.length” with “registrars.length”.


## Medium Risk

### In contract bridge.sol, functions send and burn do not implement the re-entrancy protection properly, you will think it is not necessary to specify the nonRenetrat modifier to the send and burn functions because those both functions calls the deductFee function which have that protection active, but however before calling the deductFee function, there is another external call to the processedPayment function at line 423, here an malicious user, could create a token that will have a modified allowance function and with that will be possible to re-entar the send or burn functions, we are aware that the tokens addresses need to be pre-approved by an administrator but the malicious code could hide in plain sight and the malicious actor could use this technique to extract liquidity from his community using bridge.sol contract.

**Recommendation**:

Add the nonReentrant modifier to all the public/external functions that are doing external
calls or are changing the contract state and remove it from the internal/private functions, the
internal/private functions will be protected from re-entracy because the only way to call them
is through an public/external function anyway, so there is no need to add nonReentrant
modified on the private/internal function, but it is a strong need to add them on all the
external/public ones that are doing external calls or are changing the contract state.

### In contract bridge.sol, function processedPayment, line 424, is a call to the transferFrom function from a ERC20 contract, because the address of the token contract is a token with user input, this can lead to a malicious attack, not all the tokens have requires inside their transfer or transferFrom functions that will make them fail if something is not ok during execution because the ERC20 standard does not requires it and this is the reason why the transfer functions have a return true statement.

**Recommendation**:

Add SafeERC20 library in the contract and use safeTransferFrom instead of transferFrom to
make interactions with any tokens safe.

### In contract tokenLock.sol, function processedPayment, there’s a transferFrom call from a ERC20 contract without checking whether the call was succesful.

**Recommendation**:

Add SafeERC20 library in the contract and use safeTransferFrom instead of transferFrom to
make interactions with any tokens safe.

## Informational

### When running the coverage tool with the solidity optimizer disabled it results in a stack too deep error when compiling. This happens because the EVM is limited to assigning 16 slots for local variables. Running with optimizer solves the issue but most of the external plugins, like solidity-coverage, does not support the optimizer, to have better access to tolling and plugins we will recommend to fix the issue.
![image](https://github.com/user-attachments/assets/48fa4873-3dd9-4b37-b5c7-d354eac6a064)

**Recommendation**:

Refactor functions that have many local variables and try to split them in smaller and more
manageable functions.

### In contract settings.sol there’s an assembly block at lines 62-64. This also applies to the bridge.sol file where there’s the same assembly block inside the constructor.

**Recommendation**:

Extract the assembly block in a separate method and add a disable comment for the no-inline-
assembly warning. This way you don’t mix assembly methods with the code logic.

### When iterating consider declaring the length variable as a local variable instead of reading it each time inside the loop. This helps save gas costs, as the most expensive operations are storage reads/writes, so by reading the variable only once it reduces the gas cost. For example in contract settings.sol, at line 67 the for loop can be refactored as follows. Do this for all for loop occurrences.

### There are multiple versions of solidity in different files. For example in tokenLock.sol the version declared is pragma solidity ^0.8.2,but the declared compiler version in hardhat.config is 0.8.0.

**Recommendation**:

Use consistent versioning.

### There are multiple contracts where global variables have no explicit visibility declarations. For example in the settings.sol file at line 17 there’s no visibility declared.

**Recommendation**:

Add explicit visibility declarations in all contracts and consider refactoring global variables in
groups, such as public and private groups.

### There is mixed usage of error messages when reverted inside functions. Some error messages are incomplete or plain abbreviations that are explained inside errorsMsgs.txt. That’s not the preferred way to have error messages.

**Recommendation**:

Remove the .txt file and add full length error messages whenever there’s require.

### Most of the .sol files in the contract directory are lower case, so are the names of the contracts. It’s preferred that contract names start with capital letters and follow a camelCase style.

**Recommendation**:

Refactor the contract names and if you wish also rename the .sol files to match the contract
casing. Also most of the code is not formatted correctly or using inconsistent indentation. Use
a tool such as solhint and format/indent the code correctly as it’s hard to follow. Also, for
further details check: https://docs.soliditylang.org/en/v0.8.11/style-guide.html

### In contract “Wrapped Deployer”, the variable “lossessEnabled” is initialized when it is declared at line 11 and there is no way to modify it.

**Recommendation**:

The variable should be initialized through the constructor and a function should be added to
allow the variable to be modified.
