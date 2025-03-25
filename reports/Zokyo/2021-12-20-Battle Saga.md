**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### Missed messages in exceptions

**Description**


Token.sol. In case of discrepancy in requirements the error will not return a description of
exception. It will make it harder to reveal a cause of revert.

**Recommendation**:

Use messages in “require” statements to add a description of exceptions.

## Informational

### Constant can be used

**Description**

Token.sol, Line 29. Token amount calculation can be moved to the public constant in order to
save gas during the deployment.

**Recommendation**:

Consider using the public constant.

### Centralization risk

**Description**

Token.sol. Contract uses onlyOwner modifier to control access to admin’s functionality. In case
of losing access to the owner's address or sharing access with an unwanted person, admin
can lose access to admin’s functionality.

**Recommendation**:

Consider using multisig as an owner’s address.

**Post-audit**:

By the client, ownable functionality will be used single time after the listing
