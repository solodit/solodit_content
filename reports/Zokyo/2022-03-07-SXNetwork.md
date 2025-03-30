**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### Contract WSX have a centralization risk in the adminWithdrawal function, the admin can withdraw any amount of native tokens that are available without the burning of the underlying erc20 tokens.

**Recommendation**:
Implement a multi-sig mechanism for the admin address.

**Team Response**:
Will be implemented after initial launch

### Contract SXVault, possible to have a centralization risk in the withdraw function, the admin can withdraw any balance from the vault.

**Recommendation**:

Implement a multi-sign mechanism for the admin address.

**Team Response**:

Will be implemented after initial launch

## Informational

### Contract EIP712Base contains a typo at line 46 in function name “getDomainSeperator”.

**Recommendation**:
Rename the function to ““getDomainSeparator”

### Contract EIP712Base contains a type at line 20 in internal variable named ““domainSeperator””.

**Recommendation**:

Rename the internal variable name to “domainSeparator”.

### In contract WSX, at line 38 and line 54, the Deposit and Withdrawal events are emitted without using the ‘emit’ keywords.
The emit keywords was introduced since solidity v0.4.21- 8.3.2018, and is the recommend way
of explicitly calling events to make them stand out from regular function calls.

**Recommendation**:
Call the events using the recommend way through the emit keyword.

### In contract WSX, function adminWithdraw uses the modifier onlyAdmin which use the global variable msg.sender, but inside the function at line 65, is used the _msgSender internal function.

**Recommendation**:
Use the global variable msg.sender in both places to have the same code context.
