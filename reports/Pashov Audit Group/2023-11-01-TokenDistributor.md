**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Medium Risk

### [M-01] DoS attack on initialization is possible

**Severity**

**Impact:**
High, as the initialization can be blocked

**Likelihood:**
Low, as it a front-running type of an attack with no benefit for the attacker

**Description**

The `initializeDistributor` method in `TokenDistributor` has the following check:

```solidity
require(token.balanceOf(address(this)) == _totalTokensToDistribute, "totalTokensToDistribute must match token balance of contract");
```

This gives the expectation that the owner will pre-transfer let's say 10 tokens to the contract and then set the `_totalTokensToDistribute` argument to 10 when calling `initializeDistributor`. The problem with this is that if a malicious user front-runs the owner call with a transfer of 1 wei worth of `token` to the contract, the check and the transaction will revert as the balance will not be equal anymore.

**Recommendations**

Change the code in the following way:

```diff
- require(
-    token.balanceOf(address(this)) == _totalTokensToDistribute,
-    "totalTokensToDistribute must match token balance of contract"
- );
+ token.safeTransferFrom(msg.sender, address(this), _totalTokensToDistribute);
```

and do not pre-transfer tokens to the contract.

## Low Risk

### [L-01] Insufficient input validation

In `initializeDistributor` multiple parameters are insufficiently validated:

- `_maxEthContributionPerAddress` is only checked that is > 0, but this seems like too low of a limit
  , maybe use 1e18
- `_distributionStartTimestamp` is checked that it is `>= block.timestamp` but it isn't checked that it isn't too further away in the future
- `_distributionEndTimestamp` is checked that it is `> _distributionStartTimestamp` but it isn't checked that it isn't too further away in the future

For `_maxEthContributionPerAddress` consider using a bigger lower limit, like `1e18` for example. For `_distributionStartTimestamp` check that it isn't more than 1 week or month, or year away in the future, depending on your use case, and for `_distributionEndTimestamp` check that it isn't more than (again) a week or a month or a year away after the start timestamp.