**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Signature is replayable

**Severity**: Medium

**Status**: Resolved

**Description**

WalletAccount.sol - Function `validateUserOp()` (i.e. in BaseAccount.sol) is susceptible to replay attacks. A rogue caller is able because of this vulnerability to reuse an already executed signature to authorize a new action without the consent of the signer to have that action repeated.
The implementation of `validateUserOp()` actually validates the nonce (i.e. line 46) which is supposed to mitigate that issue:
// BaseAccount.sol
```solidity
function validateUserOp(UserOperation calldata userOp, bytes32 userOpHash, uint256 missingAccountFunds)
    external override virtual returns (uint256 validationData) {
44  _requireFromEntryPoint();
45  validationData = _validateSignature(userOp, userOpHash);
46  _validateNonce(userOp.nonce);
```
But WalletAccount.sol is supposed to implement `_validateNonce()` by overriding it since it is not implemented in BaseAccount.sol. Therefore the contract is exposed to replay attack because of the lack of validation of the nonce.
Recommendation - Override _validateNonce() in WalletAccount.sol by rejecting used nonces. 
**Fix**: Issue becomes irrelevant because the function is only callable by EntryPoint contract (out of audit scope). It is worth noting that the EntryPoint includes the required mechanism in NonceManager contract shown as follow:
```solidity
   function _validateAndUpdateNonce(address sender, uint256 nonce) internal returns (bool) {

        uint192 key = uint192(nonce >> 64);
        uint64 seq = uint64(nonce);
        return nonceSequenceNumber[sender][key]++ == seq;
    }
```

## Low Risk

### Lack of event emission

**Severity**: Low

**Status**: Resolved

**Description**

WalletAccount.sol - Function `withdrawDepositTo()` is an important function that is called by privileged admin in order to withdraw valuable assets.
```solidity
   function withdrawDepositTo(address payable withdrawAddress, uint256 amount) public onlyOwner {
        entryPoint().withdrawTo(withdrawAddress, amount);
    }
```
This function should emit event to help the stakeholders in the project to follow up with such an important action.

**Recommendation**

Emit event on calling `withdrawDepositTo()`.

### Use the Latest Version Of OpenZeppelin Contracts

**Severity** - Low

**Status** - Acknowledged

**Description**

OpenZepplin regularly updates their contracts in order to fix security issues that are present in the older version , it is always advisable to use the latest version of the OZ contracts. COTI currently uses ^4.2.0 and the latest version of the OZ contracts is ^5.0.0 . See all the past issues for past versions here  https://github.com/OpenZeppelin/openzeppelin-contracts/security which includes a signature malleability in the ECDSA which is being used by the VerifyingPaymaster.sol. 
https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-4h98-2769-gh6h

**Recommendation**:

Update to newer versions of OpenZeppelin contracts.

**Client comment**: 

We decided to use openzeppelin 4.9.2. This version is used by @account-abstraction package.


### No 0 address check in ‘withdrawDepositTo’ function

**Severity**: Low

**Status**: Resolved

**Location**: WalletAccount.sol

**Description**

`withdrawDepositTo` function in the provided code allows the withdrawal of an amount to a specified address. However, there is no check to ensure that the withdrawAddress is not the zero address (0x0). Sending funds to the zero address effectively destroys them, as the zero address is not controlled by any party and cannot execute transactions. Implementing a zero address check can prevent accidental loss of funds due to user error.

**Recommendation**:

**Implement Zero Address Check**:

Add a condition in the withdrawDepositTo function to ensure that the withdrawAddress is not the zero address. The updated function would look like this:
```solidity
require(withdrawAddress != address(0), "withdrawDepositTo: withdraw to the zero address");
```

## Informational

### Inefficient Access of Array Lengths

**Severity**: Informational 

**Status**: Resolved 

**Location**: WalletAccount.sol


**Explanation**:

In the given Solidity code, the lengths of the arrays dest, value, and func are accessed multiple times within the executeBatch function. Specifically, dest.length is read in each iteration of the for loop. Accessing the length property of an array in Solidity is not a simple memory read but involves a certain cost of gas each time it is executed. This repeated access can lead to unnecessary gas consumption, especially in loops.
Rationale:
Caching commonly used values in memory can significantly reduce gas costs. In this case, caching the lengths of the dest, value, and func arrays outside the loops will avoid repeated costly lookups on the blockchain. This is particularly beneficial in loops where the length property is accessed in every iteration, as the gas savings will be multiplied by the number of iterations.
**Recommendation:**
Cache Array Lengths: Store the lengths of dest, value, and func in local variables before the loops begin. Use these variables in the loop conditions and other relevant places in the function. E.g
```solidity
uint256 destLength = dest.length;
uint256 valueLength = value.length;
uint256 funcLength = func.length;
```

### Inefficient Use of public Function Visibility

**Severity**: Informational 

**Status**: Resolved

**Explanation**:

The current implementation of the smart contracts uses public visibility for certain functions that are exclusively called externally. In Solidity, the public visibility allows a function to be called both internally (within the contract) and externally (from other contracts or transactions). However, when a function is only intended to be called externally, declaring it as external instead of public can result in reduced gas costs. This is because external functions can access their parameters more efficiently from calldata.
Rationale:
Using external visibility for functions that are only called from outside the contract is a best practice in Solidity for optimizing gas usage. The key reasons are:
Calldata Efficiency: external functions can read inputs directly from calldata, which is cheaper in terms of gas than reading from memory (used by public functions).
Clarity and Intent: Declaring a function as external clearly communicates that it is not meant to be called internally, which can improve the readability and maintainability of the code.

**Recommendation:**

Identify functions currently marked as public that are only called externally and change their visibility to external.

### Use of Post-Increment Operator in Loop (i++)

**Severity**: Informational 

**Status**: 

**Location**: WalletAccount.sol 


**Explanation:**

In the provided Solidity code, the loop iteration variable i is incremented using the post-increment operator (i.e., i++). In Solidity, the post-increment operation i++ returns the original value of i before incrementing. This requires additional temporary storage and can be more expensive in terms of gas usage.
Rationale:
Solidity, like many programming languages, provides two types of increment operations: post-increment (i++) and pre-increment (++i). The pre-increment operation ++i increments the variable i and then returns the incremented value. This is generally more efficient in a loop where the value returned by the increment operation is not used elsewhere, as it avoids the need for temporary storage of the original value.

**Recommendations:**

Replace Post-Increment with Pre-Increment: Modify the loop incrementation from i++ to ++i. This small change can save gas, especially in loops that are executed numerous times, as is often the case in contract functions.

### Unnecessary overhead of safe math that could be avoided

**Severity**: Informational

**Status**: Resolved

WalletAccount.sol -
```solidity
   function executeBatch(address[] calldata dest, uint256[] calldata value, bytes[] calldata func) external {
        _requireFromEntryPointOrOwner();
        require(dest.length == func.length && (value.length == 0 || value.length == func.length), "wrong array lengths");
        if (value.length == 0) {
            for (uint256 i = 0; i < dest.length; i++) {
                _call(dest[i], 0, func[i]);
            }
        } else {
            for (uint256 i = 0; i < dest.length; i++) {
                _call(dest[i], value[i], func[i]);
            }
        }
    }
```
**Recommendation** 

Avoid that by putting unchecked { i++; } at the end of the body of the for loops.
**Fix**:  Client carried out the suggested optimization in commit  85c1a4b



### Floating pragma

**Severity**: Informational

**Status**: Resolved

**Description**

The audited contracts use the following floating pragma:
```solidity
pragma solidity ^0.8.12;
```
It allows for the compilation of contracts with various compiler versions and introduces the risk of deploying with a different version than the one used during testing.

**Recommendation** 

Use a specific version of the Solidity compiler.


### Assumed Security Coverage by out of scope components

**Severity**: Informational

**Status**: Acknowledged

**Description**

Developers within the scoped contracts make the assumption that the external entrypoint component inherently provides security coverage for critical aspects. However, this assumption introduces a substantial vulnerability, as evidenced by the following issues:

Contract Takeover via entrypoint Compromise: In the event that the entrypoint component is compromised, malicious actors can exploit this external vulnerability to take control of the entire contract, specifically the "WalletAccount" contract, by upgrading its implementation contract. This scenario exposes a significant security risk, as unauthorized parties could gain control over the contract's functionality and potentially cause financial losses or unauthorized access.
Signature Replay Vulnerability: The contract WalletAccount is susceptible to signature replay attacks as described in this report in a separate issue. This is limited though as it takes place when the caller is entrypoint. A compromised entrypoint is able to reuse valid signatures and perform unauthorized actions or it can unintentionally do so.

**Recommendation**: 

While the entrypoint component falls outside the audit scope, it is recommended for developers to explicitly address and mitigate the assumed security coverage gap. Consider implementing additional security measures within the scoped contracts to independently safeguard against potential vulnerabilities.
**Fix**: As EntryPoint comes from the trusted authority in our space, it is very likely to be secure. For interested stakeholders, the specifications are described in the following EIP https://eips.ethereum.org/EIPS/eip-4337 


### Zero Address Checks

**Severity** - Informational

**Status** - Resolved

**Description**

There are instances in the codebase where if the address parameter is mistakenly set to 0 address then it might be problematic for the system , moreover the owner can not be updated once it has been set. Following are the instances from the code →
```solidity
L50  (WalletAccount.sol)
L95  (WalletAccount.sol)
```
**Recommendation**:

Introduce 0 address checks for the above.

