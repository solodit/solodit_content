**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### In contract FOX Yieldy, function decreaseAllowance at lines 300-314, when allowed value is smaller then subtracted value will put 0 for allowed value. It can lead to undefined behavior.

**Recommendation**:

Add a require statement to check if _subtractedValue is smaller than allowed value.

### In contract Staking, the variable TOKE_REWARD_HASH, declared at line 23, and assigned at line 84, is not used after assignment.

**Recommendation**:

Remove the variable and constructor parameter.

## Informational

### In contract FOX Yieldy, function rebase at lines 97-126, if circulating supply is 0 the call to _storeRebase will always revert. Move circulating supply check from _storeRebase at the beginning of function Rebase.

### In contract FOX Yieldy, function transfer at lines 210-220, should verify if sender has enough value.

### In contract FOX Yieldy, function transfer at lines 210-220, should verify if address of _to is not zero address.

### In contract FOX Yieldy, function transferFrom at lines 244-259, to prevent underflow panic, add a require for allowedValue[_from][msg.sender] - _value to be greater than 0. It’s easier to understand when a custom message is provided.

### In contract LiquidityReserve, given that at line 11 the SafeERC20 is used for the IERC20 and in order to be consistent with other ERC20 transfer calls, use safeTransferFrom instead of transferFrom.


