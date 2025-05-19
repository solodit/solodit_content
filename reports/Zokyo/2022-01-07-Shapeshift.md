**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Function totalVaultBalance() doesn't use firstVaultId variable

**Description**

Cycle “for (uint256 i = 0; i <= _lastVaultId; i++)” always starts from 0 but must start from the
firstVaultId variable.

**Recommendation**:
Change “uint256 i = 0” to “uint256 i = firstVaultId”

**Re-audit**:
Fixed.

## Low Risk

### Functions _migration and _withdraw have unused statements migrator != address(this) and withdrawer != address(this)

**Description**

Function _withdraw can be called by functions: _migrate, withdraw(address,address),
withdraw(address, address, uint256), withdraw(address, address, uint256, uint256, uint256).
They all set the withdrawer as _msgSeder().
The same situation is with migrator. Function _migrate can be called by functions:
migrate(address), migrate(address, uint256), migrate(address, uint256, uint256, uint256). They
all set the migrator as _msgSeder().

**Comment**:
2 unused statements were removed but one was left on line 365.
Commit 3b65405d97655236b3174f4318ad1c413f01e3f2.

**Re-audit**:
Fixed.
