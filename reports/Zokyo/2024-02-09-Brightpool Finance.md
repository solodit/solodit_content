**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Certain ERC20 tokens do not return bool from approve which breaks the core logic
**Description**

The BrightPoolLenger and Uniswap Exchange contracts attempt to approve the ask or bid token for other contracts and require the approve function to return true. However, some ERC20 implementations (e.g., mainnet USDT) do not return a value. This will cause a revert when the implementation tries to decode the boolean return value of the call to approve.
if (!order.bid.token.approve (address (exchangeable), order.bid.amount)) revert Insufficient Funds();
However, some ERC20 implementations (e.g., mainnet USDT) do not return a value. This will cause a revert when the implementation tries to decode the boolean return value of the call to approve.

**Recommendation**

Consider using SafeApprove instead of approve.

**Re-audit comment**

Resolved

## Low Risk

### Validate array lengths
**Description**

In the contract Vesting.sol's constructor array of vestedPlans, receivers, amounts and timestamps is passed but there are not sufficient checks to verify the lengths of these arrays. It should be made sure that these arrays are of equal lengths otherwise it might result in erroneous assignment.

**Recommendation**

Ensure all these arrays are of equal lengths.

**Re-audit comment**

Resolved

## Informational

### Unreachable Branch
**Description**

In the function_internalDeposit in the contract Vesting.sol the condition if (receiver_ == address(0)) revert ZeroAddress() is unnecessary since at L114 ERC20's mint is called which reverts on 0 address mint, therefore checking the condition again is not needed.

**Recommendation**

The check can be removed.

**Re-audit comment**

Resolved
