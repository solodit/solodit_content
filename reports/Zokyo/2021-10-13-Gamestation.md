**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Solidity files has no license declaration.

**Recommendation**:

Specify license in every Solidity file.


## Informational

### Useless require in contract wGameStationToken function _beforeTokenTransfer in line 62. The condition is already checked in the modifier whenNotPaused().

**Recommendation**:

Delete the line.

### There is the possibility to use modifier whenNotPaused from the Pausable contract in the contract GameStationToken function _beforeTokenTransafer instead of required in line 42.

**Recommendation**:

Add modifier whenNotPaused to the _ beforeTokenTransafer function.
