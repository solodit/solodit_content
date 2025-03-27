**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### Versioning

**Description**

Provided contracts use version 0.8.4^. 0.8.4 is not the actual version of Solidity.

**Recommendation**:
Consider using the actual version of Solidity(0.8.11). Also it must be stable (without ^).


## Informational

### DevtMine.sol::Lines 21-26. DevtMine::33-36 DevtLpMine::14-15

**Description**


The code can be simplified
You can use the “days” keyword to get an amount of seconds in a day (86400). It allows you to
delete useless “DAY” variables. The same thing is with “ONE” variable. You can use the “ethers”
keyword to get a number with 18 decimals.

**Recommendation**:

Consider using “days” and “ether” keywords.

### Type casts

**Description**


Provided contracts use type casts. This is quite an expensive and unsafe operation in Solidity.

**Recommendation**:

If there is no solution without type casts just take the risks.
