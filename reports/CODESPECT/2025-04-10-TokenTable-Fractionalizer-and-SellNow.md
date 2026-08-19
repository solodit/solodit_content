**Auditors**

[Talfao](https://x.com/talfao1), [Kalogerone](https://x.com/kalogerone)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/012_CODESPECT_TOKENTABLE_FRACTIONALIZER_AND_SELLNOW.pdf)

# Findings

## Critical Risk

### [C-01] isApprovedForAll mapping allows arbitrary transfer of any segment

**Files:** [ShareFractionalizer.sol](https://github.com/EthSign/tokentable-sellnow-ft-fractionalizer-evm/blob/076d70abad50c2dddf8c5324c98d72f68ec6dc72/src/ShareFractionalizer.sol#L207)

**Description:**

The `setApprovalForAll(...)` function is intended to allow a trusted operator to manage *all* of a user's segments. While an operator cannot claim a user's segment (i.e., receive token distribution), they are allowed to transfer segments to arbitrary recipients. Since the user explicitly chooses the operator, this is assumed to be a trusted delegation.

However, the `transferFrom(...)` function contains a flawed access control check:

```solidity
function transferFrom(address from, address to, uint256[] calldata segmentIDs) external {
    for (uint256 i = 0; i < segmentIDs.length; i++) {
        // has to be approved to transfer the segment
        require(
            operators[from][segmentIDs[i]] == _msgSender() || isApprovedForAll[from][_msgSender()],
            InvalidAccess() // @audit-issue: no relation to specific segmentID
        );
        // cannot transfer segments that are already claimed
        require(segments[segmentIDs[i]].claimableAmount != 0, SegmentAlreadyClaimed());
        // clear approval
        operators[from][segmentIDs[i]] = address(0);
        segments[segmentIDs[i]].recipient = to;
    }
}
```

The first `require` checks whether `msg.sender` is either:

- Approved for a specific segment via `operators[from][segmentID]`, or;
- Approved for all of the user's segments via `isApprovedForAll[from][msg.sender]`;

The issue is with the second condition: `isApprovedForAll[from][msg.sender]` is a global flag and does not validate that the segment in question actually belongs to `from`. This means `msg.sender` can transfer any segment—even if `from` is not the original owner of that segment—as long as the global approval flag is set.

**Impact:** An attacker can exploit this logic to transfer *any* active segment (i.e., those with non-zero claimable tokens) from *any* user who has granted them global approval. This effectively allows the theft of all future token distributions tied to those segments.

**Recommendation:** Modify the access control check to ensure that each `segmentID` being transferred is indeed owned by the `from` address.

**Status:** Fixed

**Update from TokenTable:** [ecdf862821b03307ef07940a8c7097f8803cbdb7](https://github.com/EthSign/tokentable-sellnow-ft-fractionalizer-evm/pull/13/commits/ecdf862821b03307ef07940a8c7097f8803cbdb7)

## High Risk

### [H-01] Aggregator does not approve token spending

**Files:** [BuyerAggregator.sol](https://github.com/EthSign/tokentable-sellnow-swap-evm/blob/101eff63f094aafd9f5ea7bcd1025acba2e272f0/src/BuyerAggregator.sol#L86-L97)

**Description:**

The `BuyerAggregator` contract collects funds from whitelisted buyers (in ETH or an ERC20 token) to facilitate the purchase of future tokens, which are later fractionalized via the `ShareFractionalizer`. The deposit token is defined by the `SellNow` session configuration. Once the required amount is collected, the `confirm()` function is called to finalize the process. This performs two key actions:

1. Locks the aggregator from accepting further deposits;
2. Calls `sellNow.confirm()` to proceed with the buyer-side confirmation;

However, when the `sellNow.confirm()` function is later called from the seller side, it will revert because the aggregator does not approve the `SellNow` contract to transfer the payment tokens on its behalf.

Since ERC20 transfers initiated by external contracts require explicit approval, the lack of an `approve(...)` call prevents the funds from being moved to the seller, halting the process.

**Impact:** Denial of service: the selling process cannot be completed because the aggregator has not authorized the `SellNow` contract to transfer the required tokens. As a result, the seller cannot receive their funds, and the finalisation of the session remains stuck.

**Recommendation(s):** Add an `approve(...)` call for the payment token (including any associated fees) to the `SellNow` before calling `sellNow.confirm()`

**Status:** Fixed

**Update from TokenTable:** [1b00e2c2e3daf853d320440668a812e075889ee7](https://github.com/EthSign/tokentable-sellnow-swap-evm/pull/11/commits/1b00e2c2e3daf853d320440668a812e075889ee7)

### [H-02] Fees not considered in expectedAmount inside BuyerAggregator

**Files:** [BuyerAggregator.sol](https://github.com/EthSign/tokentable-sellnow-swap-evm/blob/101eff63f094aafd9f5ea7bcd1025acba2e272f0/src/BuyerAggregator.sol#L89-L94)

**Description:**

The `BuyerAggregator` contract is used to collect funds from whitelisted buyers (in ETH or an ERC20 token) for the purpose of purchasing future tokens. The future tokens are later fractionalized via the `ShareFractionalizer`. The type of token to be deposited depends on the `SellNow` session configuration. Once the required amount is collected, the `confirm(...)` function is called to finalize the process. This action:

1. Locks the aggregator from further deposits;
2. Calls `sellNow.confirm(...)` to proceed with the buyer-side confirmation;

The issue lies in how the aggregator checks whether the required amount of tokens has been collected. In the `confirm(...)` function, the aggregator compares its current balance against `expectedAmount`, which is loaded from `sessionConfig.paymentToken.amount`. However, this value does not include session fees.

```solidity
function confirm() external onlyBeforeConfirm {
    confirmed = true;
    address paymentToken = sessionConfig.paymentToken.tokenAddress;
    uint256 expectedAmount = sessionConfig.paymentToken.amount;
    require(
        paymentToken == address(0)
            ? address(this).balance == expectedAmount
            : IERC20(paymentToken).balanceOf(address(this)) == expectedAmount,
        IncorrectPaymentAmount()
    );
    sellNowInstance.confirm(sessionId);
}
```

As a result, when `sellNow.confirm(sessionId)` is called, the aggregator might not have enough funds to cover both the token price and the associated fees. This leads to a revert, even though the aggregator believes the full amount has been collected.

**Impact:** Denial of service of the selling process: the aggregator will be unable to confirm the session if fees are not accounted for in the `expectedAmount`.

**Recommendation(s):** Update the aggregator's `confirm(...)` logic to include any applicable session fees in the required balance check.

**Status:** Fixed

**Update from TokenTable:** [887d94a2a456bbece39074bf63ced2c33d203646](https://github.com/EthSign/tokentable-sellnow-swap-evm/pull/11/commits/887d94a2a456bbece39074bf63ced2c33d203646)

### [H-03] Native ETH is not supported by SellNow contract

**Files:** [SellNow.sol](https://github.com/EthSign/tokentable-sellnow-swap-evm/blob/101eff63f094aafd9f5ea7bcd1025acba2e272f0/src/SellNow.sol)

**Description:**

The `BuyerAggregator` contract collects funds from whitelisted buyers (in ETH or an ERC20 token) to facilitate the purchase of future tokens, which are later fractionalized via the `ShareFractionalizer`. The type of token accepted is defined in the `SellNow` session configuration. Once the required amount is collected, the `confirm()` function is called to finalize the process. This function:

1. Locks the aggregator from accepting further deposits;
2. Calls `sellNow.confirm()` to proceed with the buyer-side confirmation;

As previously mentioned, the aggregator supports native ETH deposits. However, the `SellNow` contract is not capable of receiving ETH during the `confirm(...)` call. Moreover, due to the two-step confirmation process, it is not feasible to use `transferFrom` with native ETH—this method only works for ERC20 tokens.

This results in a fundamental incompatibility between ETH-based sessions and the payment mechanism in `SellNow`.

Furthermore, it is currently not possible to establish a session using native ETH as the payment token. In the aggregator, native ETH is represented as `address(0)`, but this configuration is not properly handled during session creation and results in an invalid setup.

**Impact:** Denial of service: ETH collected by the aggregator cannot be used to pay and finalize the session in the `SellNow` contract. As a result, the confirmation process fails, and the sale cannot be completed.

**Recommendation(s):** Introduce special handling for ETH payments. Two possible solutions:

- Wrap ETH into WETH within the aggregator before confirming, allowing compatibility with ERC20 logic;
- Define a separate ETH-specific execution path for `SellNow.confirm()` that accepts native ETH;

**Status:** Fixed

**Update from TokenTable:** [423c967dc3596720bb92bd258ad55007263dc888](https://github.com/EthSign/tokentable-sellnow-swap-evm/pull/11/commits/423c967dc3596720bb92bd258ad55007263dc888)

**Update from CODESPECT:** The issue has been resolved. The TokenTable team has chosen to wrap the ETH to WETH. In case the seller does not confirm their part of the escrow process, an owner-only function `withdrawDepositOwner(...)` was introduced to allow withdrawal of these tokens.

This function also updates the payment token, which introduces a level of centralisation. However, this trade-off was accepted to prevent scenarios where ETH from deposits could remain stuck in the contract due to wrapping complications.

### [H-04] approved segments don’t reset their approval after being transferred

**Files:** [ShareFractionalizer.sol](https://github.com/EthSign/tokentable-sellnow-ft-fractionalizer-evm/blob/076d70abad50c2dddf8c5324c98d72f68ec6dc72/src/ShareFractionalizer.sol)

**Description:**

The `approve(...)` function gives permission to a trusted operator to be able to transfer the selected user's segments. This delegation of authority is considered secure because the user specifically selects who becomes an operator and for which of his segments.

However, if the segments get transferred through `updateShareSegmentsAdmin(...)` by the owner or `transferShareSegments(...)` by the user, the operator of those segments remains the same.

The `transferFrom(...)` function doesn't account for this scenario, allowing an operator to transfer segments that he was previously approved for by the previous owner:

```solidity
function transferFrom(address from, address to, uint256[] calldata segmentIDs) external {
    for (uint256 i = 0; i < segmentIDs.length; i++) {
        // has to be approved to transfer the segment
        require(
            operators[from][segmentIDs[i]] == _msgSender() || isApprovedForAll[from][_msgSender()], InvalidAccess()
        );
        // cannot transfer segments that are already claimed
        require(segments[segmentIDs[i]].claimableAmount != 0, SegmentAlreadyClaimed());
        // clear approval
        operators[from][segmentIDs[i]] = address(0);
        segments[segmentIDs[i]].recipient = to;
    }
}
```

The issue arises from the fact that the operator of a segment doesn't get reset when `updateShareSegmentsAdmin(...)` or `transferShareSegments(...)` is called.

**Impact:** A malicious user can always set himself as the operator of his segments and regain access to them in the case owner calls `updateShareSegmentsAdmin(...)` or possibly sells/has to transfer his segments to another address.

**Recommendation(s):** Set the operator to `address(0)` after changing a segment's recipient in `updateShareSegmentsAdmin(...)` and `transferShareSegments(...)`.

**Status:** Fixed

**Update from TokenTable:** [3f3a670c0f67983a0dd3acbec5e0eeee11e1806d](https://github.com/EthSign/tokentable-sellnow-ft-fractionalizer-evm/pull/13/commits/3f3a670c0f67983a0dd3acbec5e0eeee11e1806d)

## Medium Risk

### [M-01] DOS in BuyerAggregator’s confirm function

**Files:** [BuyerAggregator.sol](https://github.com/EthSign/tokentable-sellnow-swap-evm/blob/101eff63f094aafd9f5ea7bcd1025acba2e272f0/src/BuyerAggregator.sol)

**Description:**

Users are supposed to call the `confirm()` function in `BuyerAggregator` when they have collected the required amount to buy the NFT:

```solidity
function confirm() external onlyBeforeConfirm {
    confirmed = true;
    address paymentToken = sessionConfig.paymentToken.tokenAddress;
    uint256 expectedAmount = sessionConfig.paymentToken.amount;
    require(
        paymentToken == address(0)
            ? address(this).balance == expectedAmount
            : IERC20(paymentToken).balanceOf(address(this)) == expectedAmount,
        IncorrectPaymentAmount()
    );
    sellNowInstance.confirm(sessionId);
}
```

However, there is a check that the contract's token balance must be equal to the `expectedAmount`. That means that malicious users can always send as little as 1 wei to the contract to DOS this function call.

**Impact:** An attacker can DOS the `confirm()` function call by frontrunning the transaction and sending 1 wei of the token. Even if users `withdrawDeposit(...)` to make the balance correct again, the malicious user can repeat the attack.

**Recommendation(s):** Make the check confirm that the contract's token balance is `>=` to the `expectedAmount`.

**Status:** Fixed

**Update from TokenTable:** [030d772345c18f08c79dfbab9dd9fffe89772763](https://github.com/EthSign/tokentable-sellnow-swap-evm/pull/11/commits/030d772345c18f08c79dfbab9dd9fffe89772763)

## Low Risk

### [L-01] Disallow aggregator withdrawals once the buyer side is confirmed

**Files:** [BuyerAggregator.sol](https://github.com/EthSign/tokentable-sellnow-swap-evm/blob/eafe0d070b48a51e65394fc7b6f4eb3928f9af12/src/BuyerAggregator.sol)

**Description:**

The `BuyerAggregator` allows buyers to withdraw their deposits at any time. However, this feature can introduce issues and cause the `SellNow` confirmation process to revert if buyers withdraw their funds after the aggregator has confirmed the buy side, but before the seller has confirmed their side of the process.

Once the aggregator confirms the buyer side, withdrawals should be disallowed to preserve the integrity of the session. However, if the seller cancels or fails to confirm before a defined deadline, buyers should regain the ability to withdraw their deposits.

**Impact:** Buyers may withdraw funds after confirmation, potentially breaking the `SellNow` confirmation flow and leading to denial of the confirmation process.

**Recommendation(s):** Consider restricting buyer withdrawals after buy-side confirmation, while also handling the scenario where the seller fails to confirm within the expected timeframe.

**Status:** Fixed

**Update from TokenTable:** [0aa773198e15addcdd9d42637ccc596020e28d46](https://github.com/EthSign/tokentable-sellnow-swap-evm/pull/11/commits/0aa773198e15addcdd9d42637ccc596020e28d46)

### [L-02] Improper handling of cancelled token allocations in Fractionalizer

**Files:** [ShareFractionalizer.sol](https://github.com/EthSign/tokentable-sellnow-ft-fractionalizer-evm/blob/076d70abad50c2dddf8c5324c98d72f68ec6dc72/src/ShareFractionalizer.sol#L161-L195)

**Description:**

The `actual` in the unlocker represents the token allocation, which is distributed over time. The allocation can be cancelled in two ways: either by wiping out the pending claimable amount to prevent future claims, or by preserving the pending claimable amount for future distribution.

This can potentially cause issues on the `Fractionalizer` claiming side. For example:

- An `actual` is configured to distribute 2000 tokens;
- The fractionalizer splits this into four segments: (500, 500) at time X and (500, 500) at time Y, assigned to User A and User B, respectively;
- Before reaching time X (in the X/2), the actual is cancelled with no wiping, meaning that only 500 tokens could be claimed;
- Due to timing, front-running, or delayed admin actions, the segment shares are not updated by the time X is reached;
- User A calls `claim(...)` and successfully receives their 500-token share;
- User B, however, is unable to claim, as the total distributable amount has been depleted by User A;

This results in an unfair scenario where one user receives their allocation while another does not, even though both were entitled to equal shares under the original distribution.

**Impact:** Unfair distribution of tokens could arise if such an event occurs, leading to inconsistent or biased claims depending on timing and order of execution.

**Recommendation(s):** Consider updating the logic to handle this scenario fairly, or require that segments are updated before any further claims can be processed after cancellation.

**Status:** Acknowledged

**Update from TokenTable:** very unlikely scenario, resolving it will significantly increase gas cost. Decide to not patch.

### [L-03] Use call(...) instead of transfer(...)

**Files:** [BuyerAggregator.sol](https://github.com/EthSign/tokentable-sellnow-swap-evm/blob/eafe0d070b48a51e65394fc7b6f4eb3928f9af12/src/BuyerAggregator.sol#L152)

**Description:**

The `BuyerAggregator` allows users to withdraw their deposited funds via the `withdrawDeposit(...)` function. When native ETH is used as the payment token, the contract sends ETH during withdrawal using the low-level `transfer(...)` call. This method has a fixed gas limit, which may cause withdrawals to revert when interacting with smart wallets or contracts that require more gas to accept ETH.

It is generally recommended to use the low-level `call(...)` to avoid such reverting scenarios. However, `call(...)` introduces a reentrancy risk and must be handled with caution. Furthermore, if you introduce the `call(...)`, after execution it returns a boolean value indicating success of the call rather than reverting on failure.

**Impact:** Withdrawals may revert for users with smart contract wallets.

**Recommendation(s):** - Consider replacing `transfer(...)` with a low-level `call(...)`, and ensure the return value is checked for success.

If switching to `call(...)` is not preferred, consider adding an explicit `recipient` parameter to allow users to withdraw to an EOA, reducing the likelihood of failed transfers.

**Status:** Fixed

**Update from TokenTable:** [520ae0070e21837211885a4760636bfc6df53363](https://github.com/EthSign/tokentable-sellnow-swap-evm/pull/11/commits/520ae0070e21837211885a4760636bfc6df53363)

## Informational

### [I-01] Missing check for transferability

**Files:** [SellNow.sol](https://github.com/EthSign/tokentable-sellnow-swap-evm/blob/eafe0d070b48a51e65394fc7b6f4eb3928f9af12/src/SellNow.sol#L48-L54)

**Description:**

The `SellNow` contract allows establishing a session between a buyer and a seller for an `actual` (i.e., `futureToken`). During session creation, several parameters are validated. However, to ensure consistency and prevent potential issues, the `createSession(...)` function could additionally verify whether the `futureToken` is transferable.

**Impact:** If a session is created with a non-transferable `futureToken`, the `SellNow` functionality will operate incorrectly.

**Recommendation(s):** Consider adding a check to ensure the `futureToken` is transferable before allowing session creation.

**Status:** Acknowledged

**Update from TokenTable:** will be handled off-chain
