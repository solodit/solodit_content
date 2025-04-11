**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### TODO in pre-production code

**Description**

Line 84
TODO statements in pre-production code for allocations calculations. This statement needs
clarification, especially from the point of view of the non-audited code potentially added.

**Recommendation**:

Clarify the statement and finish the logic.

### Missing total amount calculation

**Description**

Line 83, No calculations and checks for the total amount added through the addAllocations()
method.
There are no checks if the total amount exceeds the current supply, or if the new portion of
allocations added to the same vesting type exceeds the allocations, or if the next vesting type
amount added will exceed the total supply. This is the crucial point because there is no logic
for changing the frozen wallet with the allocation.

**Recommendation**:

Finish the total amount checking logic.

**Post-audit**:
After the conversation with the Shield Finance team, auditors verified the functionality and its
coverage with tests. The issue is marked as resolved.


## Medium Risk

### Missing mint method

**Description**

Line 95, _mint() function overloading
Token’s initializer already provides mint for the full maximum supply (line 48, mint for
getMaxTotalSupply()). Though the token can enable burn functionality, so more place for
minting appears. Though, overloaded _mint() function is called from nowhere but from the
initializer, which is called only once. Looks like either public mint() function is absent, or
overload _mint() method is unnecessary.

**Recommendation**:

Clarify the minting logic and the necessity of _mint() and mint() methods.


## Low Risk

### Extra variable

Line, 95, _mint() function.
Since there is overloaded totalSupply() method, prefix “super.totalSupply()” can be omitted so
as the local variable. Require statement may be simplified to “getMaxTotalSupply() >=
(totalSupply() + amount)”. For now such statements are confusing and misleading,

**Recommendation**: 

simplify _mint() function.

### Commented code

**Description**

Line, 228, 260, commented line.

**Recommendation**: 

remove commented code to prevent misleading.
