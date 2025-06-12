**Auditor**

[Kann Audits](https://x.com/KannAudits)

# Findings

## High Risk
### [C-01] Miscalculation in rewardswill give toomuchrewardstousers

**Severity**

Critical

**Description**

In syncRewards, the rewards are calculated by subtracting the last rewards to the total
yield. This is inaccurate because total yield keeps increasing, and last rewards are just the last amount
rewarded.
This means that over time, rewards will become much more inflated than their actual values, giving
away too many yield to the users. The first users to withdraw will effectively steal from others.

Proof of Concept:
Let’s assume that we get 10 ether of rewards per cycle, for simplicity.
The key parts of function syncRewards work as follow:

Cycle 1:
yieldEth has increased to 10 ether. lastRewardAmount is 0 (first run).
uint256 nextRewards = yieldEth - lastRewardAmount
uint256 nextRewards = 10 ether - 0 = 10 ether
lastRewardAmount is set to nextRewards, 10 ether.

Cycle 2:
yieldEth has increased to 20 ether. lastRewardAmount is 10.
uint256 nextRewards = 20 ether - 10 ether = 10 ether
lastRewardAmount is set to nextRewards, 10 ether.

Cycle 3, where things start to break:
yieldEth has increased to 30 ether. lastRewardAmount is 10.
uint256 nextRewards = 30 ether - 10 ether = 20 ether
Next rewards are calculated as twice the normal amount.

**Team Response**

Fixed.

## High Risk
### [H-01] Incorrect Accounting of currentWithheldETHinInstantWithdrawals

**Severity**

High

**Description**

In the withdraw(address recipient) function, when the withdrawal amount is less than
or equal to currentWithheldETH, the function enters the instant withdrawal path. In this case, the
contract does not call plumeStaking.withdraw(), and the funds are expected to be fully covered by the
ETH already held in currentWithheldETH. However, the code still sets withdrawn = amount and then
unconditionally adds this value back to currentWithheldETH. Immediately after, it subtracts amount
from currentWithheldETH, effectively leaving the value of currentWithheldETH unchanged.

```solidity
} else {
fee = amount * INSTANT_REDEMPTION_FEE / RATIO_PRECISION;
withdrawn = amount;
}
uint256 cachedWithheldETH = currentWithheldETH;
currentWithheldETH += withdrawn; // adding the withdraw amount
withHoldEth += fee;
currentWithheldETH -= amount; // deducting the amount
```

This logic is flawed because the ETH used to fulfill the withdrawal is not newly received; it is already
present in currentWithheldETH, and should simply be subtracted, not first added and then subtracted.
The current logic makes it appear as if no ETH was withdrawn, which results in incorrect accounting
of the withheld pool. If multiple such withdrawals occur, the system will consistently over-report the
available withheld ETH, potentially leading to over-withdrawals or failed withdrawals in the future
when actual funds are not available.

**Team Response**

Fixed.

## High Risk
### [H-02] User Funds Burned With No ETH Returned Due to Missing withdrawalRequests[msg.sender]Assignment After Unstake

**Severity**

High

**Description**

The unstaking process is executed in two steps:
The user initiates unstake().
If currentWithheldETH is insufficient, the contract attempts to unstake from the Plume validator.
However, during the fallback logic that handles validator unstaking, the contract returns amountUnstaked without saving it in the withdrawalRequests[msg.sender] mapping. This causes a critical failure in the withdrawal flow.
Users who unstake in this path will have their frxETH tokens burned but will receive no ETH in return because
no withdrawalRequest is saved.
The withdraw() function depends on withdrawalRequests[msg.sender] being populated.

This effectively results in loss of user funds with no means of recovery through the intended withdraw
flow.

**Team Response**

Fixed.

## Medium Risk
### [M-01] Incorrect increasing of end timestamp causes Over Extended delay in syncRewards()

**Severity**

Medium

**Description**

The syncRewards() function contains faulty logic that causes the rewardsCycleEnd to
be pushed one full cycle further than necessary. When the function is called late into the current cycle—for example at timestamp 180 with a rewardsCycleLength of 100—it calculates the next rewardsCycleEnd as 200, but then adds an extra 100, setting it to 300. This happens because of the condition:

```solidity
if (end - timestamp < rewardsCycleLength) {
end += rewardsCycleLength;
}
```

At timestamp 180, the current cycle (ending at 100) is already complete, and only 20 seconds remain
to complete the next one. However, instead of moving to 200 as expected, the function pushes the
end to 300, delaying the next sync by a full cycle. This causes up to nearly one full cycle of rewards to
be skipped and forces users to wait longer than necessary to receive rewards.

**Team Response**

Fixed.

## Medium Risk
### [M-02] _depositEther Does Not Increment Validator Index,Causing All Deposits to Funnel Into First Validator and Fail Once Full

**Severity**

Medium

**Description**

In the stPlumeMinter contract, the _depositEther() function is responsible for distributing incoming ETH deposits across available validators. This function loops through the validator set
to find suitable validators with enough capacity to stake the deposited ETH.
However, there is a critical issue: the index variable, initialized to 0, is never incremented inside the
while loop. As a result, the loop evaluates the same validator (validators[0]) in every iteration, preventing the function from ever progressing to the next validator in the list.
When the first validator reaches its staking capacity or becomes inactive, the getNextValidator() function
returns a capacity of 0. Since depositSize < minStakeAmount in such cases, the full remaining
amount is stored in the currentWithheldETH variable. This means the user’s deposit is held back and
never actually staked, despite the user receiving yield-bearing tokens.

**Team Response**

Fixed.

## Medium Risk
### [M-03]  DoS Vulnerability in withdraw() Due to Overcommitment of currentWithheldETH in Concurrent unstake() Calls

**Severity**

Medium

**Description**

The protocol maintains a global variable currentWithheldETH to represent the amount
of ETH reserved for user withdrawals after they call unstake(). However, this variable is only decreased
when withdraw() is called — not when unstake() is made. This creates a critical race condition: multiple
users can initiate unstake() relying on the same currentWithheldETH balance, leading to overcommitment.
If one user completes the withdrawal first, the remaining users are blocked from withdrawing, resulting in a denial of service (DoS).

Example Scenario
Initial State: currentWithheldETH = 10 ETH
Alice calls unstake(10 ETH) → succeeds (as 10 ETH is available)
Bob calls unstake(8 ETH) → also succeeds (still sees 10 ETH as available)

Both transactions pass the currentWithheldETH >= amount check, since the variable hasn’t been
decremented yet.

Alice calls withdraw() first:
Receives 10 ETH
currentWithheldETH becomes 0

Bob then calls withdraw():
Fails, as there is no remaining currentWithheldETH

Additionally, Bob cannot call unstake() again due to the presence of an active withdrawal request.
If no one deposits or triggers a stake rebalance before Bob’s cooldown period ends, Bob is temporarily
locked out of his funds until admin calls unstakeGov() or rebalance happens.

**Team Response**

Fixed.
