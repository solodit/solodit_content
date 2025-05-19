**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Max amount may exceed totalSupply

**Description**

Although these functions can be called only by an authorized person, the total of
maxAmounts should be calculated in order not to exceed totalSupply or balance of
_lgePairAddress .
Also notice, that max amount for each round is a limit for each recipient, but not a total limit
for a whole round. Consider adding more checks.

**Recommendation**:

Add an internal function which calculates sum of max amounts and reverts if total is
exceeded. Call this function in both createLGEWhitelist and modifyLGEWhitelist functions after
setting new values to amountsMax. Or confirm the functionality and calculations.

## Low Risk

### Not restricted round start

**Description**

LGEWhitelisted.sol, line 145, _applyLGEWhiteList function
For now anyone can start the purchases rounds (send tokens to _lgePairAddress, thus
triggering _applyLGEWhiteList ).

**Recommendation**:

Confirm the functionality or consider restricting for owner/whitelister only.

### Parameters lack zero check

**Description**

BlockBank.sol, line 16
Function parameters recipient and amount should be checked for misleading transactions
prevention (e.g. in case of the frontend validations failure).

**Recommendation**:

Add “require” statements to check that parameters are not zeros before executing the
function.

### Address lacks zero-check

**Description**

LGEWhitelisted.sol, line 61

Missing “require” statement for comparison “pairAddress” function parameter with zero
address. It is needed to prevent misleading transactions or a transaction with incorrect
arguments (e.g. because of frontend validation failure).

**Recommendation**:

Add “require” statement before executing function.

## Informational

### Use storage pointer

**Description**

LGEWhitelisted.sol, line 86 “modifyLGEWhitelist” function
There are too many usages for _lgeWhitelistRounds[index]. In order to provide gas savings
during the operation, consider usage of storage pointer.

**Recommendation**:

Use storage pointer instead.

### Missing zero address check

**Description**

LGEWhitelisted.sol, line 97, modifyLGEWhitelist function
Missing checking addresses from “addresses” array parameter.

**Recommendation**:

Add “require” statement in for-loop and check every address in the array.

### Variables names too similar.

**Description**

BlockBank.sol, line 10
Constant TOTAL_SUPPLY is too similar to BEP20._totalSupply and may lead to
misunderstanding during development.

**Recommendation**:

Consider using a different name, i.e SUPPLY_CAP.

### Lacks TOTAL_SUPPLY check in mint function

**Description**

BEP20.sol, line 213
Mint function lacks check, that _totalSupply + amount will not exceed TOTAL_SUPPLY.
Although this function can be called by owner only, this check should be provided.

**Recommendation**:

Consider creating an override method mint in BlockBand.sol, which will check that
_totalSupply + amount does not exceed TOTAL_SUPPLY.

### Unnecessary Ownable inheritance

**Description**

Contract inherits Ownable contract, though there is no any function to use the abilities of the
Ownable contract.

**Recommendation**:
Remove Ownable inheritance
