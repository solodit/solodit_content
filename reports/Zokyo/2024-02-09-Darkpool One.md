**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Certain ERC20 tokens do not return bool from approve which breaks the core logic

**Severity**: High

**Status**:  Resolved

**Description**

The BrightPoolLenger and UniswapExchange contracts attempt to approve the ask or bid token for other contracts and require the approve function to return true. However, some ERC20 implementations (e.g., mainnet USDT) do not return a value. This will cause a revert when the implementation tries to decode the boolean return value of the call to approve.
```solidity
if (!order.bid.token.approve(address(exchangeable), order.bid.amount)) revert InsufficientFunds();
```
However, some ERC20 implementations (e.g., mainnet USDT) do not return a value. This will cause a revert when the implementation tries to decode the boolean return value of the call to approve.

**Recommendation**: 

Consider using SafeApprove instead of approve.

## Low Risk

### Validate array lengths

**Severity** - Low

**Status** - Resolved

**Description**

In the contract Vesting.sol’s constructor array of vestedPlans , receivers , amounts and timestamps is passed but there are not sufficient checks to verify the lengths of these arrays.
It should be made sure that these arrays are of equal lengths otherwise it might result in erroneous assignment.

**Recommendation**:

Ensure all these arrays are of equal lengths.



## Informational

### Unreachable Branch

**Severity** - Informational

**Status** - Resolved

**Description**

In the function `_internalDeposit` in the contract Vesting.sol the condition if (`receiver_ == address(0)`) revert `ZeroAddress()` is unnecessary since at L114 ERC20’s mint is called which reverts on 0 address mint , therefore checking the condition again is not needed.

**Recommendation**:

The check can be removed.
