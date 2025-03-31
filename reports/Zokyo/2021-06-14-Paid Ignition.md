**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Potentially broken un-finalization

**Description**

IgnitionIDO.sol, 60, revert_finalize().
Previous pool’s total amount is restored with the fbck_finalize value. Though, fbck_finalize is
used as a buffer for storing the unsold amount from the last finalized pool. In the general
case, the pool we want to un-finalize does not match the last finalized pool (e.g. we have
finalized pool 1 and pool 2, and now we want to un-finalize pool 1). So the fbck_finalize can
store value from another finalization operation.

**Recommendation**:

Use mapping to get finalization values for every pool, or allow the un-finalization to only the
last finalized pool - add the condition that the next pool is NOT finalized.

### Unclear behavior

**Description**

IgnitionCore.sol, recover_token().
There is no documentation or explanation of what token is recovered (is it a pool token or
payment token etc). And since that, there is no clarity if pool’s tokenTotalAmount should be
increased (in case pool token is recovered).

**Recommendation**:

Verify the functionality, add the documentation and add the total pool token amount
re-calculation if needed.

### Incorrect max check

**Description**

IgnitionCore.sol, buyTokensETH() and buyTokensQuoteAsset().
The check if the max raised is exceeded is performed without including the currently
purchased value. So the max amount can be exceeded at least once.

**Recommendation**:

Include the currently purchased value into the check, that max amount is exceeded.

## Medium Risk

### Undocumented “magic” number

**Description**

IgnitionCore.sol, 37.
Variable status_boolean is initialized just once during the construction and is not changed
anywhere. Consider declaring it as a constant in order to save gas. Also, the variable has
“magic value” - undicumented hardcoded number. Consider adding the comment or
documentation to the constant.

**Recommendation**:

Declare the variable as constant and add documentation.

## Low Risk

### Optimization in calculation

**Description**

IgnitionIDO.sol, line 53
poolTokens[_poolAddr][_pool].tokenTotalAmount is set to the result of subtraction, though,
due to the logic of previous calculations, it can be set directly to
poolTokens[_poolAddr][_pool].soldAmount.
IgnitionIDO.sol, line 238. poolTokens[_poolAddr][_pool].tokenTotalAmount can be set to
soldAmount instead of re-calculation

**Recommendation**:

Optimize the calculation.

### Library is not used in the project

**Description**

Library DateTimePool.sol is not used in the project. Review the logic or remove the library.

**Recommendation**:

Remove unused library.

### Pointless return

**Description**

IgnitionCore.sol, setPrivAndAutoTxPool() and setQuoteAsset()
IgnitionPools.sol, addTokenToPool(), setRate(), setBaseTier(), setStartDate(), setEndDate(),
setTokenTotalAmount(), transferPool(), setAdminOnPoolToken(), disablePool()
IgnitionaIDO.sol: addWhitelist(), buyTokensQuoteAsset(), buyTokensETH(), recover_token(),
withdrawToken(), withdraw(), redeemTokens()
These functions return boolean value. Though they always return true or revert the
transaction. Thus it makes the return value pointless.

**Recommendation**:

Remove the return interface.

### Unused functions in the library

**Description**

Library BitOpMapPool.sol contains functions setPkgDtTknPool() and setPkgDtWhiteList() which
are not used within the project. In case if it is native and not external library, consider
removing the unused functions.

**Recommendation**:

Remove unused functions.
