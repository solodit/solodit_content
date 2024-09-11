

### **Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### `DepositPrivateSale` could be executed in a Public Sale period		

**Severity**: High

**Status**: Resolved

**Function**: execute_deposit_private_sale

**Line**: 327

**Description**:

Function `execute_deposit_private_sale` has a check that validates that it called not before the `private_start_time`, but there's nothing that ensures the function is not called after the private sale event

**Recommendation**: 

Add the if-case to check if the current time is past the private sale period.	

## Low Risk

### Duplicated functionality				

**Severity**: Low

**Status**: Unresolved

**Functions**: execute_deposit and execute_deposit_private_sale

**Description**:

Functions `execute_deposit` and `execute_deposit_private_sale` have similar functionality of checks and deposit features. This functionality could be combined into one function.

**Recommendation**: 

Combine the deposit functionality into one function.

**Comment from a client team**: 

execute_deposit_private_sale is for private sale, and execute_deposit is for the public. While these funcs have similar functionality, we want to split them for more clarity and feature improvements.

### Duplicated functionality					

**Severity**: Low

**Status**: Unresolved

**Functions**: execute_withdraw_funds and execute_withdraw_unsold_token

**Description**:

Functions `execute_withdraw_funds` and `execute_withdraw_unsold_token` have similar functionality to withdraw tokens by the admin after the sale event. The only difference could be seen is a `state.fund_denom` vs. `state.vesting` which could be a parameter to a new function.

**Recommendation**: 

Combine the deposit functionality into one function and call it from those two.	

**Comment from a client team**: 

`execute_withdraw_funds`, `execute_withdraw_unsold_token`. these functions also have similar feature but one is for withdrawing fund tokens while another is for withdrawing reward token. Due to the same reason of #2, we will use separate functions.	

## Informational

### Unused import						

**Severity**: Informational

**Status**: Resolved

**Import**: cw2::get_contract_version

**Line**: 9

**Description**:

The imported function is never used in the code.

**Recommendation**: 

Remove unused import.		




### Unused import						

**Severity**: Informational

**Status**: Resolved

**Import**: semver::Version

**Line**: 13

**Description**:

The imported function is never used in the code.

**Recommendation**: 

Remove unused import.	

### Use of deprecated function						

**Severity**: Informational

**Status**: Resolved

**Function**: cosmwasm_std::to_binary

**Lines**: 6

**Description**:

Function `cosmwasm_std::to_binary` is deprecated

**Recommendation**: Use `to_json_binary` function instead

### Unused variable						

**Severity**: Informational

**Status**: Resolved

**Function**: migrate

**Variable**: deps

**Description**:

Unused variable in a function.

**Recommendation**: 

Prefix unused argument name with an underscore ("_deps")
