**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk


###  User’s fund can be transferred out without consent

**Severity**: Critical

**Status**: Resolved

**Description**


In contract Swap, the function settleFutureProfit(address token, uint256 amount, address from) can transfer funds from any user to itself. Anyone can call settleFutureProfit(...) with address of any other user who has approved the spender as Swapl and the tokens can be moved without the consent of the user. A bot can also monitor the approvals on public networks/mem pools and call settleFutureProfit(...) leading to the draining of user funds.  Also, the user does not get any liquidity token in return, making the tokens immovable from the contract. 

Step 1: List token in Swap.sol
Step 2: User approves Swap.sol to spend tokens
Step 3: Anyone can call settleFutureProfit(token, amount, address_from_step_2) and transfer tokens of the address from step 2 to Swap.sol.


**Recommendation**: 

Fix the logic in the contract so that no user can transfer tokens of another address and provide liquidity tokens in case the user transfers their own tokens so that tokens can be retrieved.


## Low Risk

### Unchecked possible zero address  in Liquidity.sol

**Severity**: Low

**Status**: Resolved

**Description**

In contract Liquidity.sol, at line 21 inside the constructor, the _token parameter can be zero and is not checked. In case the address is zero, the Liquidity token would not function properly and there is no fallback method to set the correct implementation after deployment.

**Recommendation**: 

Add a sanity check for the to address variable to not be zero and revert otherwise.

### Unchecked possible zero address in VariableBorrow.sol

**Severity**: Low

**Status**: Resolved

**Description**

In contract VariableBorrow.sol, at lines 93 and 94 inside the constructor, the _swap and _oracle parameters can be zero and are not checked. 

**Recommendation**: 

Add a sanity check for the to address variable to not be zero and revert otherwise.

### Lock solidity pragma

**Severity**: Low

**Status**: Unresolved

**Description**

Contracts should be deployed with the same compiler version and flags that they have been tested the most with. Lock the pragma to a specific version, since not all the compiler versions support all the features.

**Recommendation**: 

Lock pragma to version at least 0.8.12 to support all the features.

**Note #1**: Minor change has been made, but it did not fix/address the issue.

### Incorrect value comparison for uint8 type in VariableBorrow.sol

**Severity**: Low

**Status**: Resolved

**Description**

In contract VariableBorrow.sol, at line 521, 522 and 523 in function updateProtocolRevenue inside the require statements uint types are checked to be greater than or equal to 0, which is redundant as uint types can never be less than 0. The checks are meant to ensure that those values are in a certain range e.g [0, 100].

**Recommendation**: 

Remove the greater than or equal comparison and leave only the less than or equal to the range upper bound

### Unchecked possible zero address in VariableBorrow.sol

**Severity**: Low

**Status**: Resolved

**Description**

In contract VariableBorrow.sol, at line 539 in function setOracle the _oracle parameter can be zero and is not checked. This can lead to reverted transactions throughout the contract as the oracle variable is used in multiple instances.

**Recommendation**: 

Add a sanity check for the to address variable to not be zero and revert otherwise.

### Unchecked possible zero address in Swap.sol

**Severity**: Low

**Status**: Resolved

**Description**

In contract Swap.sol, at line 933 in function setPriceFeed the setPriceFeed parameter can be zero and is not checked. This can lead to reverted transactions throughout the contract as the priceFeed variable is used in multiple instances.

**Recommendation**:

Add a sanity check for the to address variable to not be zero and revert otherwise.

### Centralization risk

**Severity**: Low

**Status**: Unresolved

**Description**

There are multiple instances where the contract owner can change protocol parameters that affect the behavior of the contracts. These include calls to several functions like updating asset details, protocol revenue and general setters/mutators. 

**Recommendation**: 

Consider using multisig wallets and having a governance module

**Fix#1**: 

No response  to address that the project owners are planning to use multisig wallets to operate the contracts or not.


### Unchecked return value

**Severity**: Low

**Status**:  Resolved

**Description**

In contract VariableBorrow.sol - return value of _updateInterest is not checked in almost all its occurrences.

**Recommendation**: 

Wrap the result of the call in a require statement or handle return according to the desired behavior for each particular case.
**Note #1**:   _updateInterest    no longer returns value


### Argument might be risky

**Severity**: Low

**Status**: Unresolved

**Description**

In contract VariableBorrow.sol - in methods borrow and repay the argument to mostly referring to msg.sender adds a dimension that can be exploited if things are not perfectly implemented in Router contract. The case in which the to is not msg.sender exists if router = msg.sender. But implementation of the router contract is not the concern of this audit's scope. If the router is exploited by an attacker to bypass the require check, severe consequences might take place that will lead to attackers exploiting others investors' collateral. In this contract presumably we have a securely implemented router, hence severity is low, but a simple coding refactor is recommended.

**Recommendation**:

It is a better coding practice and more secure to isolate those external methods in this discussed scenario. Create methods exposed for EOA callers without to for borrow and repay. Also, implement other methods restricted for the router callers that include the to address. As for the logic itself, implement it in internal methods to be invoked by the external methods, suggested saving gas of course.
**Note #1**:  No change done to address this.

## Informational

### Change public functions to external where possible to save gas

**Severity**: Informational

**Status**: Resolved

**Description**

In contract Flashloan.sol, at line 18 function flashLoan is declared as public but never used for internal calls, no reason to be public.

**Recommendation**: 

Change function from public to external

### Remove hardhat console imports from the code

**Severity**: Informational

**Status**: Unresolved

**Description**

In contracts Swap.sol and VariableBorrow.sol the hardhat/console.sol library is imported and never used. 

**Recommendation**: 

Remove all occurrences of hardhat/console.sol imports from production code

### Shadowing inherited state variables

**Severity**: Informational

**Status**: Resolved

**Description**

In contract Liquidity.sol, in function lpName, the variable symbol is a shadowing variable with the same name from inherited contract ERC20.

**Recommendation**: 

Rename function parameter symbol.

### Shadowing inherited state variables

**Severity**: Informational

**Status**: Resolved

**Description**

In contract Liquidity.sol, in function lpSymbol, the variable symbol is a shadowing variable with the same name from inherited contract ERC20.

**Recommendation**: 

Rename function parameter symbol.

### Use of reason strings instead of custom errors

**Severity**: Informational

**Status**: Unresolved

**Description**

Inside the contracts the reason strings are used inside require statements as revert messages. An alternative would be to use custom errors. This contributes to reducing contract size and reducing overall gas cost.

**Recommendation**: 

Change reason strings to custom errors.


### Prefixed variable name

**Severity**: Informational

**Status**: Resolved

**Description**

In contract Swap.sol, at line 102 the variable of type IBorrowForSwap is prefixed with the “$” sign. This is not considered naming best practice in Solidity and is not ideal for readability.

**Recommendation**: 

Remove the “$” prefix


### Save gas by rearranging variables in the struct Pool declaration

**Severity**: Informational

**Status**: Resolved

**Description**


The structs used in the contract use storage slots in EVM. The arrangement of variables defined in the struct affects the storage slots required. All reads and writes to storage in Solidity are handled in 32-byte increments. Solidity tightly packs variables where possible, storing them within the same 32 bytes. The best use of storage available slots is possible by re-arranging the variables in the struct Pool such that multiple variables can be stored in the same storage slot.

**Recommendation**: 

Re-organize variables struct Pool declaration for efficient use of storage slot. Suggested declaration as below:
