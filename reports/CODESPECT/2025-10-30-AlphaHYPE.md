**Auditors**

[Talfao](https://x.com/talfao1)

[suspiciousbandicoot](https://github.com/suspiciousbandicoot)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/041_CODESPECT_AlphaHYPE.pdf)

# Findings

## Medium Risk

### [M-01] Stale precompile data leads to incorrect pricing and loss for withdrawers

**Files:** [`AlphaHYPEManager03.sol`](https://github.com/Hypurr-Fun/hfun-ahype/blob/5b8381ff277085280ed19ab069668fc8f2a47c8e/src/AlphaHYPEManager03.sol#L134-L144)

**Description:**

The function `getUnderlyingSupply(...)` returns the total HYPE amount held by the protocol. It first checks the native balance and subtracts pending deposits, fees, and claimable withdrawals, as these should not affect the price. Then, the contract interacts with the `DelegatorSummary` precompile, which returns the current state of delegated, undelegated, and pending withdrawal funds (all still considered part of the system). Finally, the spot balance is fetched via another precompile. The issue arises from the behaviour of precompiles. According to the documentation: “The values are guaranteed to match the latest HyperCore state at the time the EVM block is constructed.” [Reference](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore)

This means the precompile values are only updated at the **beginning** of a new block. Therefore, if a transaction interacts with precompiles within the **same block** after HyperCore-related state changes (e.g., assets transferred from HyperEVM to HyperCore), the returned data will be **stale**.

This becomes problematic when the protocol’s `processQueues(...)` function is executed to resolve pending deposits and withdrawals. Consider a scenario where only pending deposits exist, the following branch of code is executed:

```solidity
if (evmHype8 > 0) {
    uint256 toSendWei = Math.mulDiv(evmHype8, SCALE_18_TO_8, 1);
    (bool success,) = payable(HYPE_SYSTEM_ADDRESS).call{value: toSendWei}("");
    require(success, "Failed to send HYPE to spot");
    emit EVMSend(evmHype8, HYPE_SYSTEM_ADDRESS);
}
if (sb.total > 0) { // @audit likely related to acknowledged L-04 issue
    L1Write.stakingDeposit(sb.total);
    emit StakingDeposit(sb.total);
}
if (ds.undelegated > 0) {
    L1Write.tokenDelegate(validator, ds.undelegated, false);
    emit TokenDelegate(validator, ds.undelegated, false);
}
```

As shown, assets are transferred to HyperCore, affecting the Hyperliquid spot balance and reducing the native contract balance. However, since the precompile data is not refreshed mid-block, this new state is not yet reflected.

If a user submits a withdrawal request in the same block as the deposit queue processing, the withdrawal price calculation will use outdated (stale) data. This typically results in an inflated price and the user receiving fewer funds. The issue occurs because the spot balance used for pricing does not include the newly transferred assets, even though the corresponding aHYPE tokens have already been minted.

Although the correct price will be reflected in the next block, the user will have already incurred a loss, since the withdrawal process only considers the current (lower) price during execution. As a result, the discrepancy never self-corrects.

**Impact:** Loss of funds for withdrawers when deposits are transferred to HyperCore within the same block.

**Recommendation(s):** Keep track of the expected precompile state changes if we exist within the same block.

**Status:** Fixed

**Client response:** Resolved in [cac8c9e457602a9778506d07c09a45894866c00e](https://github.com/Hypurr-Fun/hfun-ahype/commit/cac8c9e457602a9778506d07c09a45894866c00e).

## Informational

### [I-01] HyperCore enforces 5-request withdrawal cap

**Files:** [`AlphaHYPEManager03.sol`](https://github.com/Hypurr-Fun/hfun-ahype/blob/5b8381ff277085280ed19ab069668fc8f2a47c8e/src/AlphaHYPEManager03.sol#L259)

**Description:**

Unstaking on HyperLiquid requires two steps:

a. Undelegation — which is immediate if the validator is not locked;

b. Withdrawal from the staking balance — which is subject to a 7-day unstaking period;

Once this period passes, the assets automatically appear in the spot balance on the HyperCore side. However, there is a limitation on unstaking requests in HyperLiquid: a maximum of 5 active withdrawal requests per account. Any additional requests beyond this limit are rejected. The aHYPE manager initiates withdrawal requests when there are insufficient funds in the manager contract to cover user withdrawals and there are assets available in the undelegation balance. In this case, the contract triggers a staking balance withdrawal through the CoreWriter interface on HyperEVM:

```solidity
if (ds.totalPendingWithdrawal < _virtualWithdrawalAmount) {
    // Pending withdrawal doesn't cover withdrawal amount
    _virtualWithdrawalAmount -= ds.totalPendingWithdrawal;
    uint256 toWithdrawFromStaking = Math.min(ds.undelegated, _virtualWithdrawalAmount);
    if (toWithdrawFromStaking > 0) {
        L1Write.stakingWithdraw(toWithdrawFromStaking.toUint64());
        emit StakingWithdraw(toWithdrawFromStaking);
        _virtualWithdrawalAmount -= toWithdrawFromStaking;
    }
    // Not enough pending withdrawals to cover the withdrawals, we need to undelegate the rest
    uint256 toUndelegate = Math.min(ds.delegated, _virtualWithdrawalAmount);
    if (toUndelegate > 0) {
        L1Write.tokenDelegate(validator, toUndelegate, true);
        emit TokenDelegate(validator, toUndelegate, true);
    }
}
```

The issue arises when there are already 5 pending withdrawal requests. In this situation, the contract still attempts to initiate additional withdrawals, but these calls silently fail, resulting in no assets being withdrawn and no error reported.

**Impact:** Depending on how the front-end handles the event emission, it might cause display issues showing a larger number of pending withdrawals than the actual state.

**Recommendation(s):** Implement a check to prevent withdrawal attempts from the staking balance when 5 withdrawal requests are already pending. This can be done by verifying the variable that tracks pending requests within the `DelegatorSummary`.

**Status:** Acknowledged

### [I-02] Undelegation may silently fail due to validator lock

**Files:** [`AlphaHYPEManager03.sol`](https://github.com/Hypurr-Fun/hfun-ahype/blob/5b8381ff277085280ed19ab069668fc8f2a47c8e/src/AlphaHYPEManager03.sol)

**Description:**

When unstaking from HyperCore, a user (in this case, `AlphaHYPEManager03`) must first undelegate their stake. However, if the manager has delegated to the same validator within the past day, there is a 1-day lockup period for undelegation. During the `processQueues(...)` call, if there are insufficient assets in the manager contract or pending withdrawals, the manager attempts to undelegate funds from the validator using the following code:

```solidity
// Not enough pending withdrawals to cover the withdrawals, we need to undelegate the rest
uint256 toUndelegate = Math.min(ds.delegated, _virtualWithdrawalAmount);
if (toUndelegate > 0) {
    L1Write.tokenDelegate(validator, toUndelegate, true);
    emit TokenDelegate(validator, toUndelegate, true);
}
```

If the validator is locked due to a recent delegation, the call to CoreWriter will silently fail, and no undelegation will occur.

**Impact:** Depending on how the front-end handles event emissions, this may lead to display inconsistencies, showing more undelegated funds than are actually available.

**Recommendation(s):** Consider skipping or disallowing this code path when the validator is currently locked to prevent silent failures.

**Status:** Acknowledged

**Client response:** Our keeper bot that processes the queue takes into account the unstaking 1d lock.

### [I-03] Duplicated logic for calculating underlying supply

**Original severity:** Best Practices

**Files:** [`AlphaHYPEManager03.sol`](https://github.com/Hypurr-Fun/hfun-ahype/blob/5b8381ff277085280ed19ab069668fc8f2a47c8e/src/AlphaHYPEManager03.sol)

**Description:**

The `AlphaHYPEManager03` contract provides a view function `getUnderlyingSupply(...)` that calculates the total underlying assets managed by the contract by summing balances across the EVM, delegator, and spot accounts. The `processQueues(...)` function, which is responsible for processing deposit and withdrawal requests, requires this same `underlyingSupply` value to determine the current price per share. However, instead of calling the existing `getUnderlyingSupply(...)` function, it duplicates the entire calculation logic inside its own function body.

```solidity
function processQueues() external nonReentrant {
    // ...
    uint256 evmHype8 = address(this).balance / SCALE_18_TO_8;

    // ...
    require(
        evmHype8 >= owedUnderlyingAmount + pendingDepositAmount + feeAmount,
        "AlphaHYPEManager: BANKRUPT"
    );

    // @audit The logic below for calculating underlyingSupply is duplicated
    // from the getUnderlyingSupply() function.
    uint256 underlyingSupply =
        evmHype8 - pendingDepositAmount - owedUnderlyingAmount - feeAmount; // EVM
        // balance in 8 decimals

    L1Read.DelegatorSummary memory ds =
        L1Read.delegatorSummary(address(this));

    underlyingSupply +=
        (ds.delegated + ds.undelegated + ds.totalPendingWithdrawal);

    L1Read.SpotBalance memory sb =
        L1Read.spotBalance(address(this), hypeTokenIndex); // Assuming token
        // index 1 for HYPE

    underlyingSupply += sb.total; // Assuming HYPE has 8 decimals
    // ...
}
```

This code duplication is considered bad practice as it increases the contract’s size and, more importantly, makes the code harder to maintain. If the logic for calculating the underlying supply ever needs to be changed, it must be updated in both places. Forgetting to update one of them would lead to inconsistencies and potentially critical bugs.

**Recommendation(s):** Consider refactoring the code to re-use the existing functionality. This will reduce code duplication, improve readability, and make the contract easier to maintain.

**Status:** Acknowledged
