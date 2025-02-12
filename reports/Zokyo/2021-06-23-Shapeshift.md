**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Commented code in production

**Description**

TokenDistribution.sol, line 122.
There is a commented check which can influence a process of claiming. Rather the account
itself can perform a claim (so this check should be uncommented), or anyone can perform a
claim on behalf of another account (just with correct merkle proof).

**Recommendation**:

Review commented “require” statement and confirm, that it should be removed or included to
the code.

**Post-audit**

Functionality verified by the customer.

## Medium Risk

### Solidity version update

**Description**

Issue is classified as Medium, because it is included to the list of standard smart contracts’
vulnerabilities. Currently used version (0.6.1) is not the last in the line, so it contradicts
standard checklist.

**Recommendation**:

You need to update the solidity version to the latest one in the branch - consider 0.6.12, 0.7.6
or 0.8.4.

### No security checks for the final epoch calculation

**Description**

TokenDistribution.sol, line 51.
With incorrectly handled parameters there is a possibility to get underflow during the epoch
calculation, since no SafeMath applied in this place nor necessary checks performed.

**Recommendation**:

Add necessary “require” statements to mitigate the case when “currentRewardRate /
rewardReductionPerEpoch_” equals 0 or add SafeMath.

## Low Risk

### Methods should be declared as external

**Description**

In order to decrease the gas usage, next methods should be declared as external:
TokenDistributor.initialize() (TokenDistributor.sol#27-52)
TokenDistributor.getClaimsStartTime() (TokenDistributor.sol#61-63)
TokenDistributor.getNextEpochStart() (TokenDistributor.sol#66-74)
TokenDistributor.getTimeUntilNextEpoch() (TokenDistributor.sol#76-84)
TokenDistributor.getNextEpochRewardsRate() (TokenDistributor.sol#106-111)

**Recommendation**:

Declare methods as external.

### Unnecessary Migration

**Description**

Migration.sol contract can be removed to increase the readability of the code and in order to
avoid errors during migrations.

**Recommendation**:

Remove Migration.sol.


### Extra variable

**Description**


TokenDistributor.sol, line 49: variable currentRewardRate can be completely omitted. It is
initialized just once, it is used just once and it is never changed.

**Recommendation**:

Remove currentRewardRate or set as constant.


## Informational

### Condition never met

**Description**

TokenDistributor.sol, line 108, getNextEpochRewardsRate()
if (epoch == 0) return MAX_BPS;
The condition is never met. epoch is always set to at least 1.

**Recommendation**:

Сheck the value of the epoch and the condition and/or remove the condition.
