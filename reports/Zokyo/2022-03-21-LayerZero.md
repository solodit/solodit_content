**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Subtraction might revert.

**Description**

Line 181. Storage variable ‘eqFeePool’ collects fees in function swapRemote() and subtracts
fees in function swap(). However, in case swap() is executed before swapRemote() has been
executed, subtraction ‘eqFeePool = eqFeePool.sub(s.eqReward);’ might revert.

**Recommendation**:

Make sure that subtraction won’t revert and prevent function call.

## Low Risk

### Variables lack validation.

**Description**

Function setBridgeAndFactory(). Function parameters '_bridge' and '_factory' should be
validated not to be zero addresses.

**Recommendation**:

Add validation that function parameters are not zero addresses.

### Variable lacks validation.

**Description**

Lines 199, 220. Variable ‘pending’ should be validated not to be zero before transferring
rewards to reduce gas spendings and transfer of zero value.

**Recommendation**:

Transfer rewards only in case ‘pending’ is greater than 0.

## Informational

### Use a storage constant.

**Description**

Lines 201, 234, 277, 403, 425, 585, 604. Value ‘10000’ is used across these lines directly and
should be moved to storage constant.

**Recommendation**:

Move value ’10000’ to storage constant and use it instead.

### Variable lacks validation.

**Description**

Function setRouter(). Function parameter '_router' should be validated not to be zero address.

**Recommendation**:

Add validation that the function parameter is not zero address.

### Variables lack validation.

**Description**

constructor(). Parameters ‘_bonusEndBlock’ should be validated to be greater than
‘_startBlock’ and ‘startBlock’ should be validated to be greater than block.number.

**Recommendation**:

Add corresponding validations.
