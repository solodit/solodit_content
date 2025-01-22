
**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk



### Missing gap to Avoid Storage Collisions

**Severity**: Medium

**Status**: Resolved

**Description**

The SymbiosisMiddleware contract is intended to be an upgradeable smart contract, but do not have a __gap variable.
In upgradeable contracts, it's crucial to include a _gap to ensure that any additional storage variables added in future contract upgrades do not collide with existing storage variables. This is especially important when inheriting from multiple upgradeable contracts.

**Recommendation**: 

Include a _gap as the last storage variable to SymbiosisMiddleware contract to reserve space for future storage variables and prevent storage collisions. This is a common practice to ensure compatibility and avoid issues when upgrading the contract in the future.

### Missing call to _disableInitializers

**Severity**: Medium

**Status**: Resolved

**Description**

An uninitialized contract can be taken over by an attacker. This applies to both a proxy and its implementation contract, which may impact the proxy. To prevent the implementation contract from being used, you should invoke the _disableInitializers() function in the constructor to automatically lock it when it is deployed, more information can be found here.

**Recommendation**: 

Consider calling _disableInitializers()in the constructor.



### Incorrect Operator Validation in registerOperator Function

**Severity**: Medium

**Status**: Resolved

**Description**:
In the registerOperator function of the SymbiosisMiddleware contract, there is a critical bug in the operator validation process. The function is incorrectly checking if the msg.sender (the contract owner) is registered in the OPERATOR_REGISTRY instead of verifying the operator parameter. This bug allows the registration of potentially invalid operators, as long as the contract owner is registered in the OPERATOR_REGISTRY.

**Current implementation:**
```solidity
if (!IRegistry(OPERATOR_REGISTRY).isEntity(msg.sender)) { // BUG HERE
   revert NotOperator();
}
```
**Recommendation:**
```solidity
Replace the incorrect validation check with the following code:
if (!IRegistry(OPERATOR_REGISTRY).isEntity(operator)) {
   revert NotOperator();
}
```

This change ensures that the function verifies the validity of the operator being registered, rather than the contract owner.

## Low Risk

### Lack of Two-Step Ownership Transfer 

**Severity**: Low

**Status**: Resolved

**Description**

The SymbiosisMiddleware contracts does not implement a two-step process for transferring ownership. In its current state, ownership can be transferred in a single step, which can be risky as it could lead to accidental or malicious transfers of ownership without proper verification.

**Recommendation**: 

Implement a two-step process for ownership transfer where the new owner must explicitly accept the ownership. It is advised to use OpenZeppelin’s Ownable2StepUpgradeable.

### The owner can renounce ownership

**Severity**: Low

**Status**: Resolved

**Description**

The OwnableUpgradeable contracts includes a function named renounceOwnership() which can be used to remove the ownership of the contract. 
If this function is called on the SymbiosisMiddleware contract, it will result in the contract becoming disowned. This would subsequently break several critical functions of the protocol that rely on onlyOwner modifier.



 **Recommendation**: 
 
 override the function to disable its functionality, ensuring the contract cannot be disowned e.g.
```solidity
function renounceOwnership() public override onlyOwner { 
revert ("renounceOwnership is disabled"); 
}
```


### Risk of Centralisation

**Severity**: Low

**Status**: Acknowledged

**Description**

The current implementation grants significant control to the owner through multiple functions that can alter the contract's state and behavior. This centralization places considerable trust in a single entity, increasing the risk of potential misuse.
If the owner's private key is compromised, an attacker could execute any function accessible to the owner, potentially leading to fund loss, contract manipulation, or service disruption.

**Recommendation**: 

To enhance security and reduce the risk of a single point of failure, it is recommended to implement a multi-signature wallet for executing owner functions. 

**Client comments**: 
- The owner of the contract will be the multisig address;
- We have added a slashing mechanism to reduce the risk of an economic attack on our protocol from operators.

### Excessive Loops in SymbiosisMiddleware Functions 

**Severity**: Low

**Status**: Acknowledged

**Description**

The contract contains functions, such as getOperatorCollaterals and getOperatorStake, that perform extensive iterations over the vaults EnumerableSet. These loops directly iterate over the total number of vaults and invoke external calls within each iteration. The gas cost for executing these functions increases linearly with the number of vaults in the set, potentially leading to out-of-gas errors or exceeding block gas limits in scenarios with a large number of vaults. This inefficiency could disrupt contract operations and prevent users from successfully executing these functions in production environments.


**Recommendation**: 

Introduce mappings or indexed storage to facilitate direct access to operator-specific or collateral-specific data.

**Client comment**:  Since there will be a limited number of registered vaults in the Middleware contract (highly likely < 20), loop through them in the view function should not be a problem

### Missing Event Emission

**Severity**: Low

**Status**: Resolved

**Description**

functions in the SymbiosisMiddleware contract do not emit events to log state changes. This omission reduces the transparency and traceability of these operations, which can hinder off-chain monitoring.

**Recommendation**: 

Introduce event definitions for vault and operator registration/unregistration actions. Emit these events at the end of each respective function to ensure that all state changes are logged on-chain.


## Informational

### Lack of Operator Existence Verification in getOperatorCollaterals and getOperatorStake Functions

Severity: Informational

Status: Resolved

**Description**:

The getOperatorCollaterals and getOperatorStake functions in the SymbiosisMiddleware contract do not verify whether the provided operator address is registered within the system before performing operations. This oversight could lead to misleading results and potential security risks.


**Recommendation**:

Modify both functions to include a check for operator registration before proceeding with the stake calculations. This can be done by using the operators EnumerableSet:
```solidity
function getOperatorCollaterals(address operator) public view returns (address[] memory, uint256[] memory) {
   require(operators.contains(operator), "Operator not registered");
   // ... rest of the function ...
}


function getOperatorStake(address operator, address collateral) public view returns (uint256 amount) {
   require(operators.contains(operator), "Operator not registered");
   // ... rest of the function ...
}
```
Alternatively, if you want to maintain the current function signature, you could return zero values for unregistered operators:



```solidity
function getOperatorCollaterals(address operator) public view returns (address[] memory, uint256[] memory) {
   if (!operators.contains(operator)) {
       return (new address[](0), new uint256[](0));
   }
   // ... rest of the function ...
}


function getOperatorStake(address operator, address collateral) public view returns (uint256 amount) {
   if (!operators.contains(operator)) {
       return 0;
   }
   // ... rest of the function ...
}
```
