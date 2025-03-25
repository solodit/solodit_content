**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Deposits and shares may get stuck.

**Description**

Funds may be stuck in case a user transfers his share to another user or smart-contract, which
doesn’t not interface for transferring shares.
Consider the next scenarioc
j User1 deposits tokens to TimeLockPool and receives sharesj
^j User1 transfers his shares to User2j
j Both users wait for a maturity periodj
\j User1 tries to withdraw his deposits, but the transaction fails since he doesn’t have enough
shares. This way, his deposits are stuckj
Zj User2 tries to withdraw, using shares which he received from User1, but the transaction
fails because his array of deposits is empty. This way, shares are stuck until User2 transfers
them back to User1. In case, User2 is a smart-contract, which does not support an
interface for transferring shares, funds may get stuck.

**Recommendation**:

Consider rewriting values of deposits of both users during transfer.
PostAudit. Client verified that only non-transferable pools will be used.

## Medium Risk

### Token saver can withdraw staked and reward tokens.

**Description**

Line 24-27, function saveToken. There are no restrictions on which tokens can be transferred.

**Recommendation**:

Add a restriction on transferring staked and reward tokens.
PostAudit. Client verified that ability to transfer tokens will be delegated to multisig wallet.

### distributeRewards() and claimRewards() may revert.

**Description**

Lines 65, 70. There is no restriction in the constructor on rewardToken to be a zero address. In
this case, both functions distributeRewards() and claimRewards() will always revert.
This can also cause a denial of service during calling function
LiquidityMiningManager.distributeRewards() (Line 110). There is a loop for distributing
rewards to all the pools. However, in case at least one of the pools has a zero address
rewardToken, the whole distribution will fail, blocking pools from receiving reward tokens.

**Recommendation**:

Either add a validation in the constructor, that rewardToken can’t be a zero address, or add a
validation in distributeRewards() and claimRewards(), so that the functions won’t revert.
## Low Risk

### Missing check for zero address.

**Description**

Line 24-27, function saveToken. There is no check for value _receiver for zero address. This
can cause accidental token loss.

**Recommendation**:

Add a validation, that _receiver is not a zero address.

### Unsafe cast from uint256 to uint64.

**Description**

Lines 57, 58. Variables block.timestamp and duration may exceed maximum of uint64.

**Recommendation**:

Use the function toUint64 from SafeCast library. Variables maxLockDuration,
MIN_LOCK_DURATION(Lines 16-17) and variables _duration, duration(Lines 46, 49) can also be
marked as uint64 to make sure that the duration doesn't exceed maximum.

## Informational

### Use onlyRole modifier.

**Description**

Line 15. Creating a custom modifier for checking role is unnecessary since AccessControl has
a modifier onlyRole.

**Recommendation**:

Use onlyRole modifier instead of custom modifier.

### Too low dust.

**Description**

claimRewards(), line 80. The contract utilizes low dust (1). Consider usage of a more realistic
amount (e.g. 1e8).

**Recommendation**:

Consider increasing the dust amount.

### Constant may be utilised.

**Description**

Contract uses “magic” number 1e18 as a definition of accuracy. Public constant may be
utilized for this purpose.

**Recommendation**:

Consider constant usage.

### Overcomplicated construction.

**Description**

Since AbstractRewards contract is not used anywhere but within BasePool, thus construction
with function pointers seems overcomplicated.

**Recommendation**:

Consider construction simplification

### Use onlyRole modifier.

**Description**

Lines 30, 35. Creating a custom modifier for checking role is unnecessary since AccessControl
has a modifier onlyRole.

**Recommendation**:

Use onlyRole modifier instead of custom modifier.

### Variable can be marked as constant.

**Description**

Line 14. Variable MAX_POOL_COUNT is never changed.

**Recommendation**:

Make the variable MAX_POOL_COUNT constant.
