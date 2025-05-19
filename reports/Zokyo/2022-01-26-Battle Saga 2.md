**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings


## High Risk

### Logic issue.

**Description**

StakingB_1.sol Line 512
“lastRewardBlock” must be initialized when the pool is set. In other case first reward will be
counted as a reward in range [0; last block number] that is a numerous amount.

**Recommendation**:

Initialize lastRewardBlock in pool setting.

### Logic issue.

**Description**

StakingB_1.sol Line 614-615
First “updatePool” function executes “pool.lastRewardBlock = block.number;” and than
“pool.accTokenPerShare += (((block.number - pool.lastRewardBlock) * pool.rate...” where in
case of first operation “block.number” becomes equal to “pool.lastRewardBlock” their
sunstraction in next operation will always cause zero, that is the coefficient for the whole
equation. Pending rewards will never be increased.

**Recommendation**:

Place the first operation after the second.

### Risk of underflow.

**Description**

StakingB_1.sol Line 579
Each user can use the unstake function even if they have nothing to unstake, so everyone can
decrease the “amountOfUsers” variable that can cause an underflow and won’t allow other
users to unstake.

**Recommendation**:

Check if “userInfo[msg.sender]” has not been already deleted before decreasing an
“amountOfUsers”.

### Risk of overflow.

**Description**


accTokenPerShare calculation utilizes 1e12*1e18*1e32 that is equal to 1e62. Max size of
unsigned integer (256) is 2**256-1 that is ~1e77. With high pool rate and low
rateDenominator such amount can cause an overflow.

**Recommendation**:

Handle or prevent an overflow exception.

## Medium Risk

### View function will never return value.

**Description**

StakingB.sol Line 46
“stakesInfo” will never return value because it creates an array [from; to) but tries to iterate
through [from; to].

**Recommendation**:

Change the format of array to
“ s = new Stake[](_to - _from+1)” or
“for (uint256 i = _from; i < _to; i++)”

## Low Risk

### Excess functionality.

**Description**

StakingB_1.sol Line 632
Calling pool.token will cause the same result as the tokenAddress() function.

**Recommendation**:

Delete tokenAddress() function.

## Informational

### Improper use of access modifiers.

**Description**

The whole functions in contract use “public” access modifiers but most of them can be called
only externally.

**Recommendation**:

Use “external” access modifier in functions that can be called only externally.

### The Solidity version should be updated.

**Description**

Best practices for Solidity development and auditors standard checklist requires strict and
explicit usage of the latest stable version of Solidity, which is 0.8.11 at the moment.

**Recommendation**:

Consider updating to “pragma solidity 0.8.11;”.

### Code structure is messed up.

**Description**

StakingB_1.sol Line 524
Storage and events must be separated in order to make code structure more readable.

**Recommendation**:

Move the “amountOfUser” variable to the other variables.

### No message in “require” statements.

**Description**

StakingB_1.sol Line 546
StakingB.sol Lines 80, 95, 97, 99, 102, 103, 115, 117
To hold the exception error messages must be added.

**Recommendation**:

Add an error message to each exception.

### Code could be simplified.

**Description**

StakingB_1.sol Line 609
“pool.lastRewardBlock = block.number” can be executed before condition so this operation
shouldn’t be repeated twice in the code.

**Recommendation**:

Use just one realisation of “pool.lastRewardBlock = block.number” before condition.

### Use constants.

**Description**

StakingB.sol Lines 17, 18, 19
StakingB_1.sol Lines 518
If the value of the storage variable can’t be changed it can be set as constant in order to save
gas.

**Recommendation**:

Set offered storage as constant
