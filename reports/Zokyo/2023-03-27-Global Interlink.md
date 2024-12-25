**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Tx sender can steal the released tokens of any whitelisted user

**Severity**: Critical

**Status**: Resolved

**Description**

In contract token_vesting.move, function `release` is having the role of releasing the vested funds, first the function is checking that the parameter ‘receiver’ is a whitelisted address and after that is proceeding to calculate the amount that will be vested, adjust the storage and transfer the tokens, however there is one problem, the user that is checked for being whitelisted and is having his amount calculated is the ‘receiver’ variable, but the account that will actually receive the tokens is the function caller, there is where the problem is at L#225, the tokens in the end are not transferred to the receiver address but to the account that have called the function, which can be the receiver or not 

**Recommendation**: 

Implement proper sanity check by making sure the tx sender have the same address with the receiver or do not pass the receiver variable through the function parameters and just assign it the address of the caller

### Whitelisted table balance and contract balance can be different resulting in locked funds for users

**Severity**: High

**Status**: Resolved

**Description**

At this moment, the whitelisted address are added through the add_to_whitelist method and the funds of the pool are added through the deposit functionality, however these 2 steps pattern can become problematic for the contract integrity and the vested users, as the contract admin can add in the whitelist table tokens amount that do not reflect the true token balances in the contract, and users could be stuck with uncorrelated/hypothetical funds because there would be nothing for them to release from vesting.

**Recommendation**: 

Implement a check to never being able to add more hypothetical total funds in the table then the ones in contract balance or every time admin add a new user to the whitelist also transfer the necessary vested funds for that user directly in the same function logic.



### Incorrect balance check

**Severity**: High

**Status**: Resolved

**Description**

In contract utility.move, in function `merge_and_split` at line 31 there’s an assertion between the base coin value and amount parameter. This is meant to ensure that there are enough remaining funds in the base coin balance, however there can be a case when the base coin remaining balance is equal to the amount parameter, so the assertion fails. This should not happen if the two values are equal.

**Recommendation**: 

Change the assertion from greater than to greater than or equal


## Medium Risk


### Insufficient sanity checks for treasury creation

**Severity**: Medium

**Status**: Resolved

**Description**

In contract token_vesting.move, in function `create_treasury` there are no sanity checks for the `total_lock_in_days`, `vesting_period_in_days` and `initial_lock_in_days` variables. It would be ideal to add checks to prevent these values from being zero or being invalid combinations. Also, the `vesting_period_in_days` and `initial_lock_in_days` parameters are not checked to make sure they are not greater than the variable `total_lock_in_days` which could lead to unexpected behavior if the vesting period 	is longer than the total lock period. Even if these values are set in a function that’s protected by admin rights, there is a chance that a mistake is made and not noticed before it leads to a possible issue.

**Recommendation**: 

Add sanity checks to make sure that these parameters are set to the correct values.




### Constant defined by a value that has not yet been published in the documentation

**Severity**: Medium

**Status**: Resolved

**Description**

In contract utility.move the const EPOCH_SECONDS is supposed to be the time duration of an epoch in seconds. This const will affect the value returned by the functions `get_current_timestamp` and `get_timestamp_from_now_by_days`. The time duration of an epoch is present in the documentation just as an example, and there is no certainty that the value will remain the same.

**Recommendation**: 

Be aware that the value of the const EPOCH_SECONDS comes from an example and not from the official documentation and there is no certainty that the value will remain the same and update the code accordingly.

## Low Risk

### The name of the function can cause a misunderstanding

**Severity**: Low

**Status**: Resolved

**Description**

In contract utility.move, the function name "get_current_timestamp" could be misunderstood as a Unix timestamp, leading to potential confusion for developers who may rely on this function. It would be better to use a more specific name to avoid any confusion. By modifying the function name to "get_time_from_genesis", it will be clearer to developers that the function is not returning a Unix timestamp, but rather the time since the genesis block. This change will help to reduce confusion and potential errors in code that relies on this function.

**Recommendation**: 

Modify the function name to "get_time_from_genesis" to more accurately reflect the function's purpose of calculating the time since the genesis block. This name will be more specific and easier for developers to understand, avoiding any potential confusion.


### Insufficient sanity check for add_to_whitelist

**Severity**: Low

**Status**: Resolved

**Description**

In contract token_vesting.move, in function `add_to_whitelist` there are no sanity checks for the whitelist_address and whitelist_amount vectors, there is one check that is making sure the vectors have the same length, however both of the vectors could have length 0 which will break the function invariant as this function will execute successful but the store will not be modified in any way.

**Recommendation**: 

Add sanity checks to make sure that the arrays are not empty..
