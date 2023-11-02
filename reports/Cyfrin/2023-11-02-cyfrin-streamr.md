**Lead Auditors**
[Hans](https://twitter.com/hansfriese)

---

# Findings

## High Risk


### `VoteKickPolicy._endVote()` might revert forever due to underflow

**Severity:** High

**Description:** In `onFlag()`, `targetStakeAtRiskWei[target]` might be less than the total rewards for the flagger/reviewers due to rounding.

```solidity
File: contracts\OperatorTokenomics\StreamrConfig.sol
22:     /**
23:      * Minimum amount to pay reviewers+flagger
24:      * That is: minimumStakeWei >= (flaggerRewardWei + flagReviewerCount * flagReviewerRewardWei) / slashingFraction
25:      */
26:     function minimumStakeWei() public view returns (uint) {
27:         return (flaggerRewardWei + flagReviewerCount * flagReviewerRewardWei) * 1 ether / slashingFraction;
28:     }
```

- Let's assume `flaggerRewardWei + flagReviewerCount * flagReviewerRewardWei = 100, StreamrConfig.slashingFraction = 0.03e18(3%), minimumStakeWei() = 1000 * 1e18 / 0.03e18 = 10000 / 3 = 3333.`
- If we suppose `stakedWei[target] = streamrConfig.minimumStakeWei()`, then `targetStakeAtRiskWei[target] = 3333 * 0.03e18 / 1e18 = 99.99 = 99.`
- As a result, `targetStakeAtRiskWei[target]` is less than total rewards(=100), and `_endVote()` will revert during the reward distribution due to underflow.

The above scenario is possible only when there is a rounding during `minimumStakeWei` calculation. So it works properly with the default `slashingFraction = 10%`.

**Impact:** The `VoteKickPolicy` wouldn't work as expected and malicious operators won't be kicked forever.

**Recommended Mitigation:** Always round the `minimumStakeWei()` up.

**Client:** Fixed in commit [615b531](https://github.com/streamr-dev/network-contracts/commit/615b5311963082c79d05ec4072b1abeba4d1f9b4).

**Cyfrin:** Verified.

### Possible overflow in `_payOutFirstInQueue`

**Severity:** High

**Description:** In `_payOutFirstInQueue()`, possible revert during `operatorTokenToDataInverse()`.

```solidity
uint amountOperatorTokens = moduleCall(address(exchangeRatePolicy), abi.encodeWithSelector(exchangeRatePolicy.operatorTokenToDataInverse.selector, amountDataWei));
```

If a delegator calls `undelegate()` with `type(uint256).max`, `operatorTokenToDataInverse()` will revert due to uint overflow and the queue logic will be broken forever.

```solidity
   function operatorTokenToDataInverse(uint dataWei) external view returns (uint operatorTokenWei) {
       return dataWei * this.totalSupply() / valueWithoutEarnings();
   }
```

**Impact:** The queue logic will be broken forever because `_payOutFirstInQueue()` keeps reverting.

**Recommended Mitigation:** We should cap `amountDataWei` before calling `operatorTokenToDataInverse()`.

**Client:** Fixed in commit [c62e5d9](https://github.com/streamr-dev/network-contracts/commit/c62e5d90ce8f8c084fe3917f499c967c85a3873b).

**Cyfrin:** Verified.

### Wrong validation in `DefaultUndelegationPolicy.onUndelegate()`

**Severity:** High

**Description:** In `onUndelegate()`, it checks if the operator owner still holds at least `minimumSelfDelegationFraction` of total supply.

```solidity
   function onUndelegate(address delegator, uint amount) external {
       // limitation only applies to the operator, others can always undelegate
       if (delegator != owner) { return; }

       uint actualAmount = amount < balanceOf(owner) ? amount : balanceOf(owner); //@audit amount:DATA, balanceOf:Operator
       uint balanceAfter = balanceOf(owner) - actualAmount;
       uint totalSupplyAfter = totalSupply() - actualAmount;
       require(1 ether * balanceAfter >= totalSupplyAfter * streamrConfig.minimumSelfDelegationFraction(), "error_selfDelegationTooLow");
   }
```

But `amount` means the DATA token amount and `balanceOf(owner)` indicates the `Operator` token balance and it's impossible to compare them directly.

**Impact:** The operator owner wouldn't be able to undelegate because `onUndelegate()` works unexpectedly.

**Recommended Mitigation:** `onUndelegate()` should compare amounts after converting to the same token.

**Client:** Fixed in commit [9b8c65e](https://github.com/streamr-dev/network-contracts/commit/9b8c65ea31b6bf15fe4ec913a975782f27c0c9a0).

**Cyfrin:** Verified.

### `onTokenTransfer` does not validate if the call is from the DATA token contract

**Severity:** High

**Description:** `SponsorshipFactory::onTokenTransfer` and `OperatorFactory::onTokenTransfer` are used to handle the token transfer and contract deployment in a single transaction. But there is no validation that the call is from the DATA token contract and anyone can call these functions.

The impact is low for `Sponsorship` deployment, but for `Operator` deployment, `ClonesUpgradeable.cloneDeterministic` is used with a salt based on the operator token name and the operator address. An attacker can abuse this to cause DoS for deployment.
Furthermore, it is even possible for an attacker to specify malicious modules and params in the initialization that will increase the severity further.

We see that this validation is implemented correctly in other contracts like `Operator`.
```solidity
       if (msg.sender != address(token)) {
           revert AccessDeniedDATATokenOnly();
       }
```

**Impact:** Attackers can prevent the deployment of `Operator` contracts, and in the worst case, victims could be deceived by `Operator` contracts with malicious modules.

**Recommended Mitigation:** Add a validation to ensure the caller is the actual DATA contract.

**Client:** Fixed in commit [8b13df4](https://github.com/streamr-dev/network-contracts/commit/8b13df49900c640df51e22f7c5a78fcad761f7cb).

**Cyfrin:** Verified.

### Malicious target can make `_endVote()` revert forever by forceUnstaking/staking again

**Severity:** High

**Description:** In `_endVote()`, we update `forfeitedStakeWei` or `lockedStakeWei[target]` according to the `target`'s staking status.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
179:     function _endVote(address target) internal {
180:         address flagger = flaggerAddress[target];
181:         bool flaggerIsGone = stakedWei[flagger] == 0;
182:         bool targetIsGone = stakedWei[target] == 0;
183:         uint reviewerCount = reviewers[target].length;
184:
185:         // release stake locks before vote resolution so that slashings and kickings during resolution aren't affected
186:         // if either the flagger or the target has forceUnstaked or been kicked, the lockedStakeWei was moved to forfeitedStakeWei
187:         if (flaggerIsGone) {
188:             forfeitedStakeWei -= flagStakeWei[target];
189:         } else {
190:             lockedStakeWei[flagger] -= flagStakeWei[target];
191:         }
192:         if (targetIsGone) {
193:             forfeitedStakeWei -= targetStakeAtRiskWei[target];
194:         } else {
195:             lockedStakeWei[target] -= targetStakeAtRiskWei[target]; //@audit revert after forceUnstake() => stake() again
196:         }
```

We consider the target is still active if he has a positive staking amount. But we don't know if he has unstaked and staked again, so the below scenario would be possible.

- The target staked 100 amount and a flagger reported him.
- In `onFlag()`, `lockedStakeWei[target] = targetStakeAtRiskWei[target] = 100`.
- During the voting period, the target called `forceUnstake()`. Then `lockedStakeWei[target]` was reset to 0 in `Sponsorship._removeOperator()`.
- After that, he stakes again and `_endVote()` will revert forever at L195 due to underflow.

After all, he won't be flagged again because the current flagging won't be finalized.

Furthermore, malicious operators would manipulate the above state by themselves to earn operator rewards without any risks.

**Impact:** Malicious operators can bypass the flagging system by reverting `_endVote()` forever.

**Recommended Mitigation:** Perform stake unlocks in `_endVote()` without relying on the current staking amounts.

**Client:** Fixed in commit [8be1d7e](https://github.com/streamr-dev/network-contracts/commit/8be1d7e3ded2d595eaaa16ddd4474d3a8d31bfbe).

**Cyfrin:** Verified.

## Medium Risk


### In `VoteKickPolicy.onFlag()`, `targetStakeAtRiskWei[target]` might be greater than `stakedWei[target]` and `_endVote()` would revert.

**Severity:** Medium

**Description:** `targetStakeAtRiskWei[target]` might be greater than `stakedWei[target]` in `onFlag()`.

```solidity
targetStakeAtRiskWei[target] = max(stakedWei[target], streamrConfig.minimumStakeWei()) * streamrConfig.slashingFraction() / 1 ether;
```

For example,
- At the first time, `streamrConfig.minimumStakeWei() = 100` and an operator(=target) has staked 100.
- `streamrConfig.minimumStakeWei()` was increased to 2000 after a reconfiguration.
- `onFlag()` is called for target and `targetStakeAtRiskWei[target]` will be `max(100, 2000) * 10% = 200`.
- In `_endVote()`, `slashingWei = _kick(target, slashingWei)` will be 100 because target has staked 100 only.
- So it will revert due to underflow during the reward distribution.

**Impact:** Operators with small staked funds wouldn't be kicked forever.

**Recommended Mitigation:** `onFlag()` should check if a target has staked enough funds for rewards and handle separately if not.

**Client:** Fixed in commit [05d9716](https://github.com/streamr-dev/network-contracts/commit/05d9716b8e19668fea70959327ab8e896ab0645d). Flag targets with not enough stake (to pay for the review) will be kicked out without review. Since this can only happen after the admin changes the minimum stake requirement (e.g. by increasing reviewer rewards), the flag target is not at fault and will not be slashed. They can stake back again with the new minimum stake if they want.

**Cyfrin:** Verified.

### Possible front running of `flag()`

**Severity:** Medium

**Description:** The `target` might call `unstake()/forceUnstake()` before a flagger calls `flag()` to avoid a possible fund loss. Also, there would be no slash during the unstaking for `target` when it meets the `penaltyPeriodSeconds` requirement.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
65:     function onFlag(address target, address flagger) external {
66:         require(flagger != target, "error_cannotFlagSelf");
67:         require(voteStartTimestamp[target] == 0 && block.timestamp > protectionEndTimestamp[target], "error_cannotFlagAgain"); // solhint-disable-line not-rely-on-time
68:         require(stakedWei[flagger] >= minimumStakeOf(flagger), "error_notEnoughStake");
69:         require(stakedWei[target] > 0, "error_flagTargetNotStaked"); //@audit possible front run
70:
```

**Impact:** A malicious target would bypass the kick policy by front running.

**Recommended Mitigation:** There is no straightforward mitigation but we could implement a kind of `delayed unstaking` logic for some percent of staking funds.

**Client:** Our current threat model is a staker who doesn't run a Streamr node. They could be a person using Metamask to do all smart contract transactions via our UI, or they could be a complex flashbot MEV searcher. But if they're not running Streamr nodes, they should be found out, flagged, and kicked out by the honest nodes.

While an advanced bot could stake and listen to `Flagged` events, if they're found out and flagged before their minimum stay (`DefaultLeavePolicy.penaltyPeriodSeconds`) is over, their stake would still get slashed even if they front-run the flagging. We aim to select our network parameters so that it will be very likely that someone staking but not actually running a Streamr node would get flagged during those `penaltyPeriodSeconds`. Then front-running the flagging wouldn't save them from slashing.

**Cyfrin:** Acknowledged.

### Operators can bypass a `leavePenalty` using `reduceStakeTo()`

**Severity:** Medium

**Description:** Operators should pay a leave penalty when they unstake earlier than expected.
But there are no relevant requirements in `reduceStakeTo()` so they can reduce their staking amount to the minimum value.

- An operator staked 100 and he wants to unstake earlier.
- When he calls `forceUnstake()`, he should pay `100 * 10% = 10` as a penalty.
- But if he reduces the staking amount to the minimum(like 10) using `reduceStakeTo()` first and calls `forceUnstake()`, the penalty will be `10 * 10% = 1.`

**Impact:** Operators will pay a `leavePenalty` for the minimum amount only.

**Recommended Mitigation:** The penalty mechanism should be implemented consistently.

**Client:** Fixed in commit [72323d0](https://github.com/streamr-dev/network-contracts/commit/72323d0099a85c8c7a5d59335492eebcc9cc66bb).

**Cyfrin:** Verified

### In `Operator._transfer()`, `onDelegate()` should be called after updating the token balances

**Severity:** Medium

**Description:** In `_transfer()`, `onDelegate()` is called to validate the owner's `minimumSelfDelegationFraction` requirement.

```solidity
File: contracts\OperatorTokenomics\Operator.sol
324:         // transfer creates a new delegator: check if the delegation policy allows this "delegation"
325:         if (balanceOf(to) == 0) {
326:             if (address(delegationPolicy) != address(0)) {
327:                 moduleCall(address(delegationPolicy), abi.encodeWithSelector(delegationPolicy.onDelegate.selector, to)); //@audit
should be called after _transfer()
328:             }
329:         }
330:
331:         super._transfer(from, to, amount);
332:
```

But `onDelegate()` is called before updating the token balances and the below scenario would be possible.

- The operator owner has 100 shares(required minimum fraction). And there are no undelegation policies.
- Logically, the owner shouldn't be able to transfer his 100 shares to a new delegator due to the min fraction requirement in `onDelegate()`.
- But if the owner calls `transfer(owner, to, 100)`, `balanceOf(owner)` will be 100 in `onDelegation()` and it will pass the requirement because it's called before `super._transfer()`.

**Impact:** The operator owner might transfer his shares to other delegators in anticipation of slashing, to avoid slashing.

**Recommended Mitigation:** `onDelegate()` should be called after `super._transfer()`.

**Client:** Fixed in commit [93d6105](https://github.com/streamr-dev/network-contracts/commit/93d610561c109058c967d1d2f49ea91811f28579).

**Cyfrin:** Verified.

### Centralization risk

**Severity:** Medium

**Description:** The protocol has a `DEFAULT_ADMIN_ROLE` with privileged rights to perform admin tasks that can affect users. Especially, the owner can change the fee/reward fraction settings and various policies.

Most admin functions don't emit events at the moment.

**Impact:** While the protocol owner is regarded as a trusted party, the owner can change many settings and policies without logging. This might lead to unexpected results and users might be affected.

**Recommended Mitigation:** Specify the owner's privileges and responsibilities in the documentation.
Add constant state variables that can be used as the minimum and maximum values for the fraction settings.
Log the changes in the important state variables via events.

**Client:** Logging added to StreamrConfig in commit [c530ec5](https://github.com/streamr-dev/network-contracts/commit/c530ec5092c1d6567357ef9993a0f029f113a0d8). Better documentation and more logging added to other contracts in commit [c343850](https://github.com/streamr-dev/network-contracts/commit/c343850136a97573349d7cb226733c9fd5729eb6). Those commits partially mitigate risks associated with leaking of the admin key.

In StreamrConfig, there isn't much difference in the power to change the config values, and in replacing the whole contract (it's upgradeable). Some maximum and minimum limits exist currently, but their main point is to sanity-check new values, especially the initial values. "Binding our hands" with tighter limits wouldn't thus really change anything, at best it would signal an intent.

Before using these admin powers to change config values or amend the contracts using upgrades, wider review (community, auditors) will be needed, to avoid unexpected side-effects that may affect users. The day-to-day is not designed to require any admin intervention. Admin powers are only needed for unforeseen circumstances (e.g. hotfixing bugs) or planned policy changes. There is no foreseeable need for such changes at the moment.

**Cyfrin:** Acknowledged.

## Low Risk

### Unsafe use of `transfer()/transferFrom()` with `IERC20`
Some tokens do not implement the ERC20 standard properly but are still accepted by most code that accepts ERC20 tokens. For example Tether (USDT)'s `transfer()` and `transferFrom()` functions on L1 do not return booleans as the specification requires, and instead have no return value. Consider using OpenZeppelin’s `SafeERC20`'s `safeTransfer()/safeTransferFrom()` instead

```solidity
File: contracts\OperatorTokenomics\Operator.sol
264:         token.transferFrom(_msgSender(), address(this), amountWei);
442:         token.transfer(msgSender, rewardDataWei);

File: contracts\OperatorTokenomics\Sponsorship.sol
187:         token.transferFrom(_msgSender(), address(this), amountWei);
215:         token.transferFrom(_msgSender(), address(this), amountWei);
261:         token.transfer(streamrConfig.protocolFeeBeneficiary(), slashedWei);
273:         token.transfer(operator, payoutWei);

File: contracts\OperatorTokenomics\OperatorPolicies\QueueModule.sol
86:           token.transfer(delegator, amountDataWei);

File: contracts\OperatorTokenomics\OperatorPolicies\StakeModule.sol
128:         token.transfer(streamrConfig.protocolFeeBeneficiary(), protocolFee);
```

**Client:** We will only use DATA token in our system. It doesn't have the above methods. So: `transfer` it is.

**Cyfrin:** Acknowledged.

## Informational Findings

### Redundant requirement
The first requirement is redundant because the second one is enough.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
156:         require(reviewerState[target][voter] != Reviewer.NOT_SELECTED, "error_reviewersOnly"); //@audit-issue redundant
157:         require(reviewerState[target][voter] == Reviewer.IS_SELECTED, "error_alreadyVoted");
```

**Client:** We want to give an informative error message for the case where a non-reviewer tries to vote. So: prefer to keep it.

**Cyfrin:** Acknowledged.

### Variables need not be initialized to zero
The default value for variables is zero, so initializing them to zero is redundant.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\DefaultLeavePolicy.sol
10:           uint public penaltyPeriodSeconds = 0;

File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
100:         uint sameSponsorshipPeerCount = 0;
226:         uint rewardsWei = 0;
```

**Client:** Fixed in commit [c847fab](https://github.com/streamr-dev/network-contracts/commit/c847fab3e57f1f2dc3aa26e8cb79d44c8ed86f5e).

**Cyfrin:** Verified.

### Event is not properly `indexed`
Index event fields make the field more quickly accessible to off-chain tools that parse events. This is especially useful when it comes to filtering based on an address. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Where applicable, each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three applicable fields, all of the applicable fields should be indexed.


**Client:** Fixed in commit [92d4145](https://github.com/streamr-dev/network-contracts/commit/92d4145aaf6f56c3e9c72b4f9ef53a89e67aa36e).

**Cyfrin:** Verified.


# Appendix

The Streamr team discovered some vulnerabilities after the audit had started. Also, there were some new risks during the mitigation process.

## Reported Issues

### No validation in `Operator.updateOperatorsCutFraction()`

**Description:** `Operator.updateOperatorsCutFraction()` doesn't check the argument (expected to be less than "1 ether" or 1e18).

```solidity
File: contracts\OperatorTokenomics\Operator.sol
417:     function updateOperatorsCutFraction(uint newOperatorsCutFraction) external onlyOperator {
418:         if (totalStakedIntoSponsorshipsWei > 0) {
419:             revert StakedInSponsorships();
420:         }
421:
422:         operatorsCutFraction = newOperatorsCutFraction;
423:         emit MetadataUpdated(metadata, _msgSender(), newOperatorsCutFraction);
424:     }
```

**Client:** Fixed in commit [d6ae2a6](https://github.com/streamr-dev/network-contracts/commit/d6ae2a6d7283b06be0827906da4294887a4dc1bb).

**Cyfrin:** Verified.

### No minimum delegation amount in `_delegate()`

**Description:** We don't check for minimum delegation amount in `_delegate`. We do check it in `_transfer`, also when processing the undelegation queue.

```solidity
File: contracts\OperatorTokenomics\Operator.sol
274:     function _delegate(address delegator, uint amountDataWei) internal {
275:         uint amountOperatorToken = moduleCall(address(exchangeRatePolicy), abi.encodeWithSelector(exchangeRatePolicy.dataToOperatorToken.selector, amountDataWei, amountDataWei));
276:         _mint(delegator, amountOperatorToken);
277:
278:         // check if the delegation policy allows this delegation
279:         if (address(delegationPolicy) != address(0)) {
280:             moduleCall(address(delegationPolicy), abi.encodeWithSelector(delegationPolicy.onDelegate.selector, delegator));
281:         }
282:
283:         emit Delegated(delegator, amountDataWei);
284:         emit BalanceUpdate(delegator, balanceOf(delegator), totalSupply());
285:         emit OperatorValueUpdate(totalStakedIntoSponsorshipsWei - totalSlashedInSponsorshipsWei, token.balanceOf(address(this)));
286:     }
```

**Client:** Fixed in commit [b1b9d19](https://github.com/streamr-dev/network-contracts/commit/b1b9d191f75f757a6ed8a4483c085ef997b9a31f).

**Cyfrin:** Verified.

### Possible division by zero

**Description:** It's possible to trigger division by zero in `operatorTokenToData` like this:
* Create a new Operator
* Send in tokens using ERC-20 (NOT transferAndCall)
* Go to the undelegation queue (we don't check if you're a delegator; which is fine, all the action happens in the front of the queue during payout only)
* Activate the queue payment

Fix by adding back the guard that we removed during branch coverage work, because we thought the condition was impossible to trigger.

**Client:** Fixed in commit [5740ee5](https://github.com/streamr-dev/network-contracts/commit/5740ee5babbc1160d9176bd7b9e5c4584b998f59).

**Cyfrin:** Verified.

### Lack of validation

**Description:** `SponsorshipFactory` only checks that a given (sponsored) stream exists in one of the two code paths when deploying sponsorships. Going to fix that by putting the check on the common code path.

**Client:** Fixed in commit [db76b0c](https://github.com/streamr-dev/network-contracts/commit/db76b0c8544b0f4e455f8f4863b799a5e81ac007).

**Cyfrin:** Verified.

### Redundant `payable` modifier

**Description:** There was a random payable left in one of the functions (`Operator.delegate()`). It will be removed, there's no reason for any of our functions to be "payable" since we aren't dealing with native tokens.

**Client:** Fixed in commit [abc8a8b](https://github.com/streamr-dev/network-contracts/commit/abc8a8bda3737ddd6db0dc64ea481866fd54762d).

**Cyfrin:** Verified.

### First depositor issue in `Operator`

**Description:** The operator owner can steal DATA by evading minimum delegation limits and exploiting rounding errors

* create new Operator
* send in tokens using ERC-20 (e.g. 1000 DATA)
* create a Sponsorship that pays 1 wei / second
* stake to that sponsorship
* withdraw very soon, receive ~10 wei
* initial exchange rate is 1:1, so totalSupply becomes 10 wei
** valueWithoutEarnings() is still ~1000 DATA
** exchange rate is now ~10 : 1000e18
* new delegators send DATA to the contract
** here could also be some rounding errors problem: if anyone delegates < ~100 DATA, they won't get ANY operator tokens
* owner can undelegate < ~100 DATA
** corresponds to 0 operator tokens, so 0 get burned, but 100 DATA gets paid out
** repeat

Will fix by burning AT LEAST so many operator tokens that the amount of DATA is covered (i.e. rounding UP instead of down). Also staking will be disabled when there's no self-delegation. This prevents operating the contract before there are operator tokens in existence, blocking this path to rounding error exploits via absurd exchange rates.

**Client:** Fixed in commit [8dc015f](https://github.com/streamr-dev/network-contracts/commit/8dc015f6a2f41340242c0060b137b81bb1055750).

**Cyfrin:** Verified.


## Mitigation Review

### In `_handleNoKick()`, it slashes the flagger although he has forceUnstaked already.

**Description:** In `_handleNoKick()`, it shouldn't slash the flagger if he has forceUnstaked already. It's because the flagger has paid the penalty while forceUnstaking.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
330:         // Unlock the flagger's stake. Slash just enough to cover the rewards, the rest will be unlocked = released
331:         uint flagStake = flagStakeWei[target];
332:         if (lockedStakeWei[flagger] >= flagStake) {
333:             lockedStakeWei[flagger] -= flagStake;
334:         } else {
335:             //...unless flagger has forceUnstaked or been kicked, in which case the locked flag-stake was moved into forfeited stake
336:             forfeitedStakeWei -= flagStake - lockedStakeWei[flagger];
337:             lockedStakeWei[flagger] = 0;
338:             leftoverWei += flagStake - rewardsWei;
339:         }
340:         if (stakedWei[flagger] > 0) {
341:             _slash(flagger, rewardsWei); //@audit shouldn't slash if forceUnstaked
342:             emit StakeLockUpdate(flagger, lockedStakeWei[flagger], getMinimumStakeOf(flagger));
343:         }
```

We should slash inside the `if (lockedStakeWei[flagger] >= flagStake)` block only.

**Client:** Fixed in commit [8fb3ac0](https://github.com/streamr-dev/network-contracts/commit/8fb3ac03556bccf2e204b1caee999a1e025aa7ce).

**Cyfrin:** Verified.


### `OperatorFactory.voters` might contain stale voters.

**Description:** `OperatorFactory.updateStake()` is called only when the operator changes the staked amount.
So if operators don't change the staked amount for a while, voters won't be updated promptly.

- An operator staked enough amounts but couldn't be a voter due to `minEligibleVoterAge`. He won't be a voter until he changes the staked amount.
- With an increased `totalStakedWei`, a voter is not eligible anymore due to the `minEligibleVoterFractionOfAllStake`, but he will be a voter anyway.

```solidity
File: contracts\OperatorTokenomics\OperatorFactory.sol
236:     function updateStake(uint newStakeWei) external { //@audit might be stale
237:         address operator = msg.sender;
238:         if (deploymentTimestamp[operator] == 0) { revert OnlyOperators(); }
239:
240:         totalStakedWei = totalStakedWei + newStakeWei - stakedWei[operator];
241:         stakedWei[operator] = newStakeWei;
242:
243:         uint voterThreshold = totalStakedWei * streamrConfig.minEligibleVoterFractionOfAllStake() / 1 ether;
244:         bool isEligible = newStakeWei >= voterThreshold && deploymentTimestamp[operator] + streamrConfig.minEligibleVoterAge() < block.timestamp; // solhint-disable-line not-rely-on-time
245:
246:         if (isEligible && votersIndex[operator] == 0) {
247:             voters.push(operator);
248:             votersIndex[operator] = voters.length; // real index + 1
249:             emit VoterUpdate(operator, true);
250:         }
251:
252:         if (!isEligible && votersIndex[operator] > 0) {
253:             uint index = votersIndex[operator] - 1; // real index = votersIndex - 1
254:             address lastOperator = voters[voters.length - 1];
255:             voters[index] = lastOperator;
256:             voters.pop();
257:             votersIndex[lastOperator] = index + 1; // real index + 1
258:             delete votersIndex[operator];
259:             emit VoterUpdate(operator, false);
260:         }
261:     }
```

**Client:** Fixed in commit [68dea4d](https://github.com/streamr-dev/network-contracts/commit/68dea4d0ffc496e24b628d5d68eede34d415db09).

**Cyfrin:** Verified.


### In `_handleKick()`, a flagger wouldn't receive a flagger reward if he has forceUnstaked/staked

**Description:** `_handleKick()` checks flaggers with `lockedStakeWei` mapping and the below scenario would be possible.

- Alice got flagged(e.g. Flag1) by other one and `lockedStakeWei[Alice] = targetStakeAtRiskWei[Alice] = 100`.
- For some reason, Alice has forceUnstaked and staked again.
- After that, Alice noticed a bad operator and flagged(e.g. Flag2) him. So `lockedStakeWei[Alice] = 100` again.
- In `_handleKick()` for Flag1, Alice will be kicked at `L260` and `lockedStakeWei[Alice] = 0`.
- In `_handleKick()` for Flag2, it will go to `L277`(else block) and she will be considered as forceUnstaked(so no rewards) although she's eligible to receive a reward.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
256:         // Take the slashing from target's locked stake...
257:         uint slashingWei = targetStakeAtRiskWei[target];
258:         if (lockedStakeWei[target] >= slashingWei) {
259:             lockedStakeWei[target] -= slashingWei;
260:             _kick(target, slashingWei); // ignore return value, there should be enough (now unlocked) stake to slash
261:         } else {
262:             //...unless target has forceUnstaked, in which case the locked stake was moved into forfeited stake, and they already paid for the KICK
263:             forfeitedStakeWei -= slashingWei - lockedStakeWei[target];
264:             lockedStakeWei[target] = 0;
265:             if (stakedWei[target] > 0) {
266:                 emit StakeLockUpdate(target, lockedStakeWei[target], getMinimumStakeOf(target));
267:             }
268:         }
269:
270:         // Unlock the flagger's stake and pay them from the slashed stake
271:         uint flagStake = flagStakeWei[target];
272:         if (lockedStakeWei[flagger] >= flagStake) {
273:             lockedStakeWei[flagger] -= flagStake;
274:             slashingWei -= safeSendRewards(flagger, flaggerRewardWei[target]);
275:         } else {
276:             //...unless flagger has forceUnstaked or been kicked; so unlock the remaining part from forfeitedStake
277:             forfeitedStakeWei -= flagStake - lockedStakeWei[flagger];
278:             lockedStakeWei[flagger] = 0;
279:             leftoverWei += flagStake;
280:         }
281:         if (stakedWei[flagger] > 0) {
282:             emit StakeLockUpdate(flagger, lockedStakeWei[flagger], getMinimumStakeOf(flagger));
283:         }
```

**Client:** Most of that flagging would happen in fully-automatic operator software written by us, so maybe it's enough that we prevent our operator from doing it.

**Cyfrin:** Acknowledged.
