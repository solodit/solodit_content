**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### `fee1e9` can be set to a higher value than expected

**Severity**: Low	

**Status**: Acknowledged

**Description**

The function `setFee()` in the `StableSwapPool.sol` and `XYKPool.sol` contracts checks the new fee to ensure that it is equal or lower than `0.1e9`:

```solidity
function setFee(uint256 fee1e9_, uint256 decayRate_) external authenticate { 
       require(fee1e9 <= 0.1e9);
       fee1e9 = uint32(fee1e9_);
       emit FeeChanged(fee1e9 * uint256(1e8)); 
   }
```

It can be seen that the `require` checks the value of `fee1e9` instead of the input parameter `fee1e9_`.

**Recommendation**:

Fix the `require` statement:
```solidity
require(fee1e9_ <= 0.1e9);
```

**Client comment**: Acknowledged. This is a typo. Fee1e9 is managed by authorized entities, so the possibility of DOS is minimal.

### Possible Denial of Service due to  `lastWithdrawTimestamp != block.timestamp` when calling Bladeswap__execute()

**Severity**: Low	

**Status**: Acknowledged

**Description**

When a call to the `Bladeswap__execute()` function in the `StableSwapPool.sol` and `XYKPool.sol` contracts is executed there is a `require` that checks if `lastWithdrawTimestamp != block.timestamp`:
```solidity
if (lastWithdrawTimestamp != block.timestamp) {
           feeMultiplier = 1e9;
}
```

If `lastWithdrawTimestamp` is not equal to `block.timestamp` then `feeMultiplier` is set to 1e9.

Then, the function execution reaches a point where a call to `_exchange_for_t1()`, `_exchange_for_t0()` or `_exchange_for_lp()` is executed. These 3 functions receives an input (the last one) as `fe1e9 * feeMultiplier`.

The mentioned functions later call `_exchange()`:
```solidity
function _exchange(
       int256 a_0,
       int256 a_1,
       int256 a_k,
       int256 b_1,
       int256 d_k,
       int256 fee
   ) internal view returns (int256) {
       int256 b_k = a_k - d_k;
       require(b_k >= 2);

       if (a_k <= b_k) {
           b_1 -= (SignedMath.max(((a_k * b_1) / b_k) - a_1, 0) * fee) / 1e18;
       } else if (a_k > b_k) {
           b_1 -= (SignedMath.max(b_1 - ((b_k * a_1) / a_k), 0) * fee) / 1e18;
       }

       int256 b_0 = _y(b_k, b_1);

       if (a_k <= b_k) {
           b_0 +=
               (SignedMath.max(((a_k * b_0) / b_k) - a_0, 0) * fee) /
               (1e18 - fee);
       } else if (a_k > b_k) {
           b_0 +=
               (SignedMath.max(b_0 - ((b_k * a_0) / a_k), 0) * fee) /
               (1e18 - fee);
       }

       return b_0 - a_0 + 1;
   }
```


If `fe1e9` is set to `1e9`, which can not be done by calling the `setFee()` function but directly in the constructor, the `fee` passed as last parameter in `_exectute`, will be the result of 1e9*1e9, which is equal to 1e18. This leads to a scenario where the denominator for the computation of b_0 will be equal to 0 and a division by 0 will revert causing a denial of service.

**Recommendation**:

Implement checks that `int256(uint256(fee1e9 * feeMultiplier))`  does not exceed 1e17.

**Client comment**: `fee1e9` is managed by authorized entities, so the possibility of DOS attack is not significant.

### Fee minimum value can get bypassed

**Severity**: Low

**Status**: Acknowledged

**Description**

The function `setFee()` in the `StableSwapPool.sol` and `XYKPool.sol` contracts allows certain addresses to set a new value for the `fee1e9` variable. This function implements a require statement for ensuring that the value is lower or equal to `0.1e9`.

```solidity
function setFee(uint256 fee1e9_, uint256 decayRate_) external authenticate { 
       require(fee1e9 <= 0.1e9);
       fee1e9 = uint32(fee1e9_);
       emit FeeChanged(fee1e9 * uint256(1e8)); 
 }
```

But when the fee is first set in the constructor when the contract is deployed there are no checks for ensuring that the value set is equal or less than  `0.1e9` so the condition can be bypassed.

```solidity
constructor(
       IVault vault_,
       string memory _name,
       string memory _symbol,
       Token t0,
       Token t1,
       uint32 fee1e9_,
       uint32 decay,
       uint32 a_
   ) SingleTokenGauge(vault_, toToken(this), this) {
       A = a_;


       _2a = int256(uint256(2 * a_));
       _4a = int256(uint256(4 * a_));
       _2a_minus_1 = int256(uint256(2 * a_ - 1));
       _8a = int256(uint256(8 * a_));
       decayRate = decay;
       fee1e9 = fee1e9_; // @audit check can be bypassed 
       index = 1e18;
       lastIndex = 1e18;
```

If fees were accidentally set higher than the threshold in the constructor, would result in a minor denial of service and would require the developers to set fees to zero (where users could get free fees due to back-running) then reset the fees using setFees where an attacker could front run this (a classic sandwich attack).

**Recommendation**:

Add the same `require` statement in the constructor to ensure that the maximum value set for `fee1e9` is `0.1e9` as well:

`require(fee1e9 <= 0.1e9);`

**Client comment**: The reason we verify fee1e9 is to prevent authorized entities from changing fees to insanely high levels in the future. There is no need to verify fee1e9 in the constructor, because there is no economic stake yet, and users can choose not to interact with pools with abnormal fees.

### Missing sanity checks for important variables

**Severity**: Low	

**Status**: Acknowledged

**Description**

There are some important variables in the `StableSwapPool.sol` and `XYKPool.sol` contract that are not checked before being set in the `constructor()` function and also in the functions used for changing their values.

- `setDecay()`:
```solidity
function setDecay(uint256 decayRate_) external authenticate {
       decayRate = uint32(decayRate_);
       emit DecayChanged(decayRate);
   }
```

- `setFee()`:

```solidity
function setFee(uint256 fee1e9_, uint256 decayRate_) external authenticate { 
       require(fee1e9 <= 0.1e9);
       fee1e9 = uint32(fee1e9_);
       emit FeeChanged(fee1e9 * uint256(1e8)); 
   }
```


**Recommendation**:

Add checks to ensure that the values set are between the desired thresholds.

### `lastWithdrawTimestamp` used instead of `lastTradeTimestamp`.

**Severity**: Low	

**Status**: Acknowledged

**Description**

The `Bladeswap__execute()` function in `StableSwapPool.sol` and `XYKPool.sol` implements an ‘if statement’ which checks if `lastWithdrawTimestamp` is equal to `block.timestamp`.
```solidity
if (lastWithdrawTimestamp != block.timestamp) {
           feeMultiplier = 1e9;
 }
```

`lastWithdrawTimestamp` is never initialized so it will always be 0 and this check will never be met. It could have been confused with `lastTradeTimestamp`.

**Recommendation**:

Consider using `lastTradeTimestamp` instead of `lastWithdrawTimestamp` is it the desired behavior.

**Client comment**: `lastWithdrawTimestamp` is the desired behavior, and we acknowledge it is not being set. This however does not cause a serious problem.

## Informational

###  Missing Event Emission at `setFeeToZero()`

**Severity**: Informational

**Status**: Acknowledged

**Description**

Event emission is a crucial practice in blockchain development for tracking and monitoring contract state changes. In this case, the setFeeToZero() function in `StableSwapPool.sol` and `XYKPool.sol` modifies the contract's fee variables (feeMultiplier and fee1e9) but doesn't emit an event to this change.


```solidity
function setFeeToZero() external onlyVault { 
       feeMultiplier = 0;
       fee1e9 = 0;
}
```

**Recommendation**: 

Consider emitting an event after setting the fee to zero.


**Client comment**: `setFeeToZero` can only be called by another permission contract, and the contract will only call this method counterfactually. In other words, it will always revert after calling this method. Event emission can be skipped as it will never happen.


### Centralization Risks

**Severity** - Informational

**Status** - Acknowledged

**Description**

System critical state can be modified (setFee and deploy pools in factory contracts) by an “authenticated” address . It should be made sure that these authenticated addresses are multisigs with a timelock to ensure that privileges are not centralised to one address.

**Recommendation**:

Make sure authenticated addresses are multisigs with a timelock.

**Client comment**: Acknowledged and we’ve set timelock contract already.

### Wrong amount in event emission

**Severity**: Informational	

**Status**: Acknowledged

**Description**

The function `setFee` in the `StableSwapPool.sol` and `XYKPool.sol` contracts emits a `FeeChanged` event every time a new fee is set. This event emits the new amount but is wrongly set.
```solidity
emit FeeChanged(fee1e9 * uint256(1e8));
```

The event emits `fee1e9 * uint256(1e)` as the new fee amount instead of emitting `fee1e9`.

The same event is emitted in the constructor.

**Recommendation**:

Change the event amount to: 
```solidity
emit FeeChanged(fee1e9); 
```

**Client comment**: Acknowledged. This is a typo. It should be fee1e9 * 1e9 instead. We’ll calculate accordingly when we need to index it in off-chain.

### Unused parameter

**Severity**: Informational	

**Status**: Acknowledged

**Description**

The function `setFee` in the `StableSwapPool.sol` and `XYKPool.sol` contracts takes `decayRate_` as parameter but it is not needed as it is not used.

```solidity
function setFee(uint256 fee1e9_, uint256 decayRate_) external authenticate { 
       require(fee1e9 <= 0.1e9);
       fee1e9 = uint32(fee1e9_);
       emit FeeChanged(fee1e9 * uint256(1e8)); 
  }
```


**Recommendation**:

It’s recommended that unused parameters are left unnamed to be more compatible with interfaces. This can also assist in improving readability for example:
```solidity
function setFee(uint256 fee1e9_, uint256) external authenticate {...}
```
**Client comment**: This is intentionally added to make it compatible with some interfaces


### Use custom errors instead of `require`

**Severity**: Informational	

**Status**: Acknowledged

**Description**

In Solidity v0.8.4, a significant improvement was implemented with the incorporation of custom errors. These errors offer a more efficient method, in terms of gas usage, for communicating failure reasons within your smart contracts compared to using revert strings. This advancement helps in decreasing both deployment and runtime expenses.

`StableSwapPool.sol` implements a require statement that can be optimized by using custom errors.

**Recommendation**:

Consider replacing require statements in favor of custom errors for the purposes of gas optimizations.

### Incomplete Deployment Scripts and Compilation Issues Around Unit Tests

**Severity**: Informational	

**Status**: Acknowledged

**Description**

To ensure a smooth deployment, it is essential that all components are thoroughly completed and tested beforehand, including deployment scripts. Failure to do so may result in significant issues during deployment. Additionally, it should be noted that the tests did not compile, indicating potential underlying problems that need to be addressed before proceeding.

### Reading a storage variable in the event


**Severity**: Informational

**Status**: Acknowledged

**Description**

In the `setDecay()` and `setFee()` function, `DecayChanged()` and `FeeChanged()` events read storage variables even if local variables exist for the same value.

**Recommendation**: 

Read `decayRate_` and `fee1e9_` instead of `decayRate` and `fee1e9` in the events.

### Setting unused local variables

**Severity**: Informational

**Status**: Acknowledged

**Description**

In the `_excessInvariant()` function, a_0 and a_1 are not set but not used at all.

**Recommendation**: 

Remove the line setting a_0 and a_1

### Missing Natspec

**Severity**: Informational

**Status**: Acknowledged

**Description**

The contract exhibits a lack of documentation, hindering its readability, maintainability, and auditability.

**Recommendation**: 

Consider adding Natspec to the contracts.

### Inefficient Read from storage slot in `setFee` function

**Severity**: Informational

**Status**: Acknowledged

**Location**: XYKPoolFactory.sol/StableSwapPoolFactory.sol

**Description**

The `setFee` function in `XYKPoolFactory.sol/StableSwapPoolFactory.sol` reads the value of fee1e9 from storage twice, which can be optimized for efficiency by reading the value of the calldata `fee1e9_` instead of fee1e9 from the storage slot.

**Recommendation**: 

Modify the setFee function to directly use the value of fee1e9_ passed as a function parameter instead of assigning it to the storage variable fee1e9 before using it. This will eliminate the need to read the value from storage twice and improve gas efficiency.

**Client comment**: This function is called very infrequently, by authorized entities only, therefore gas cost is not a concern here. Also, solc would probably be able to optimize them


### Variable Shadowing Issue in `getPools` Function

**Severity**: Informational

**Status**: Acknowledged

**Location**: XYKPoolFactory.sol/StableSwapPoolFactory.sol

**Description**

The return variable pools in the getPools function shadows the state variable pools, which can lead to confusion and potential errors in the code.

**Recommendation**: 

It is recommended to rename the return variable pools to a different name to avoid shadowing the state variable. This will improve code readability and reduce the risk of unintended errors caused by variable shadowing.

