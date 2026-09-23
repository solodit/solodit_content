**Auditors**

[shaflow01](https://github.com/shaflow01), [0xSynthrax](https://x.com/0xSynthrax), [0xbountyhunt3r](https://github.com/0xbountyhunt3r)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/049_CODESPECT_DUTCH.pdf)

# Findings

## High Risk

### [H-01] Contribution oversubscription will cause calimReward(...) call revert due to insufficient funds for late callers

**Files:** [`AuctionAssist.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/AuctionAssist.sol)

**Description:**

The `recordPurchase(...)` function allows total contribution shares to exceed 100% when multiple users contribute to a collection. The function caps each individual user's share at 10,000 BPS (100%), but does not normalize the total when the sum of all shares exceeds 10,000 BPS.

```solidity
function recordPurchase(...) external returns (uint256 purchaseId_) {
    //...
    // 7. Calculate and store contributor shares based on actual cost.
    for (uint256 i; i < contributors_.length; ++i) {
        address contributor_ = contributors_[i];
        uint256 contributionAmount_ =
            _contributions[contributor_][collection_].amount;

        if (contributionAmount_ > 0) {
            // Calculate share as (contribution * 10000) / cost.
            // @audit The total of `contributionAmount_` may be greater than `costETH_`.
            uint256 shareBPS_ =
                (contributionAmount_ * 10000) / costETH_;

            // Cap at 10000 BPS (100%) to prevent over-distribution.
            if (shareBPS_ > 10000) {
                shareBPS_ = 10000;
            }

            _purchaseShares[purchaseId_][contributor_] = shareBPS_;
            purchase_.contributors.push(contributor_);
        }
    }
    //...
}
```

If user A contributed 50% of the NFT price and user B contributed 60%, the late `claimReward(...)` caller's transaction will either revert, or pull the tokens that belong to other users from different contributed settlements, causing their claim call to revert.

**Impact:** Contribution oversubscription will cause 100% fund loss for late `claimReward(...)` callers.

**Recommendation:** Remove contribution oversubscription option for a single purchase or normalize shares proportionally when total exceeds 10,000 BPS.

**Status:** Fixed

**Client response:** Fixed in [74ee906fd0f0588124acef2486a951415685e31d](https://github.com/dutch-protocol/Protocol-Contracts/commit/74ee906fd0f0588124acef2486a951415685e31d)

### [H-02] Contributions not reset after purchase allows infinite reward claims

**Files:** [`AuctionAssist.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/AuctionAssist.sol#L309)

**Description:**

The `recordPurchase(...)` function calculates contributor shares based on their contribution amounts stored in `_contributions[contributor_][collection_].amount`. However, after recording a purchase and snapshotting the shares, these contribution amounts are never reset.

```solidity
function recordPurchase(...) external returns (uint256 purchaseId_) {
    // ...
    for (uint256 i; i < contributors_.length; ++i) {
        address contributor_ = contributors_[i];
        // @audit contribution amount is read but never reset
        uint256 contributionAmount_ =
            _contributions[contributor_][collection_].amount;

        if (contributionAmount_ > 0) {
            uint256 shareBPS_ = (contributionAmount_ * 10000) / costETH_;
            // ...
            _purchaseShares[purchaseId_][contributor_] = shareBPS_;
            purchase_.contributors.push(contributor_);
        }
    }
    // @audit no reset of _contributions[contributor_][collection_].amount
}
```

**Impact:** This allows a contributor who made a single contribution to receive reward shares for every subsequent NFT purchase from that collection indefinitely:

1. User contributes 1 ETH to Collection A;
2. NFT 1 is purchased for 2 ETH — User receives 50% share (1 ETH / 2 ETH);
3. NFT 2 is purchased for 2 ETH — User still has 1 ETH recorded, receives another 50% share;
4. This repeats for every future purchase, draining rewards meant for new contributors.

The vulnerability effectively allows infinite DUTCH token extraction from a single contribution, severely diluting rewards for legitimate contributors and draining protocol funds.

**Recommendation:** Reset contribution amounts after recording a purchase.

**Status:** Fixed

**Client response:** Fixed in commit [b5bacb594faa34b4d77a87588f0b8fbd0b781b6d](https://github.com/dutch-protocol/Protocol-Contracts/commit/b5bacb594faa34b4d77a87588f0b8fbd0b781b6d)

### [H-03] DOS in _createVestingStreams(...) due to unbounded loop

**Files:** [`Presale.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/Presale.sol#L585)

**Description:**

The `_createVestingStreams(...)` function iterates over all contributors in a single transaction to create Sablier vesting streams. Each stream creation costs approximately 175,000–250,000 gas.

```solidity
function _createVestingStreams(uint256 totalTokens) internal {
    uint256 contributorCount = contributors.length;

    // @audit unbounded loop over all contributors
    for (uint256 i = 0; i < contributorCount; i++) {
        address contributor = contributors[i];
        // ...
        // @audit each call costs ~175,000-250,000 gas
        try sablierV2.createWithDurationsLL(params, unlockAmounts, durations)
            returns (uint256 streamId) {
            // ...
        } catch {
            // ...
        }
    }
}
```

With a block gas limit of 30 million, the function can only handle approximately 120–170 contributors before reverting. An attacker can exploit this by creating 200+ small contributions from different addresses during the presale. When the presale ends and `_createVestingStreams(...)` is called, the transaction reverts due to exceeding the block gas limit. This permanently locks all presale funds (both ETH and DUTCH tokens) with no recovery mechanism.

**Impact:** Lock of funds of all contributors.

**Recommendation:** Implement a batched claiming pattern where each contributor claims their own vesting stream.

**Status:** Fixed

**Client response:** Fixed in [5f12a82ae4dc706ae4a0891b272b22f69b0cea7f](https://github.com/dutch-protocol/Protocol-Contracts/commit/5f12a82ae4dc706ae4a0891b272b22f69b0cea7f)

### [H-04] Incorrect SpecifiedDelta returned

**Files:** [`DUTCHBondingHook.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DUTCHBondingHook.sol)

**Description:**

In `DUTCHBondingHook`, the `_beforeSwap(...)` function actually executes the user's swap using a custom curve. Therefore, it needs to return a `BeforeSwapDelta` to offset the user's `params_.amountSpecified` input and transfer the `AccountDelta` generated by the hook to the sender.

```solidity
function _executeBuy(...) internal returns (...) {
    //...
    if (params_.amountSpecified > 0) {
        // Exact input: specify input, calculated output.
        delta_ = toBeforeSwapDelta(
            int128(-params_.amountSpecified),
            int128(int256(outputAmount_))
        );
    } else {
        // Exact output: calculated input, specify output.
        delta_ = toBeforeSwapDelta(
            -int128(int256(inputAmount_)),
            int128(-params_.amountSpecified)
        );
    }
}
function _executeSell(...) internal returns (...) {
    //...
    if (params_.amountSpecified > 0) {
        // Exact input: specify input, calculated output.
        delta_ = toBeforeSwapDelta(
            int128(-params_.amountSpecified),
            int128(int256(outputAmount_))
        );
    } else {
        // Exact output: calculated input, specify output.
        delta_ = toBeforeSwapDelta(
            -int128(int256(inputAmount_)),
            int128(-params_.amountSpecified)
        );
    }
}
```

However, the `_executeBuy(...)` and `_executeSell(...)` functions construct the `BeforeSwapDelta` incorrectly. The high 128 bits of `BeforeSwapDelta` represent `SpecifiedDelta`, which must cancel out the entire `params_.amountSpecified` inside `beforeSwap(...)` so that the AMM swap does not actually occur in Uniswap V4. Therefore, it should always be the exact opposite of `params_.amountSpecified`. In the branches of `_executeBuy(...)` and `_executeSell(...)` where `params_.amountSpecified < 0`, the high 128 bits of `BeforeSwapDelta` are not the opposite of `params_.amountSpecified`. This causes `params_.amountSpecified` to fail to be adjusted to 0, resulting in the AMM swap actually executing. Additionally, the lower 128 bits of `BeforeSwapDelta` represent the `UnspecifiedDelta`, which is used in `afterSwap(...)`. It should always have the opposite sign of the hook's target token `AccountDelta`, so that the hook's debt is correctly transferred to the sender.

**Impact:** `BeforeSwapDelta` does not return the correct value, causing the swap to fail because it cannot follow the intended execution flow.

**Recommendation:** It is recommended that when constructing `BeforeSwapDelta`, the high 128 bits should always be the opposite of `params_.amountSpecified`, and the low 128 bits should be the opposite of the hook's target token `AccountDelta`.

**Status:** Fixed

**Client response:** Fixed in [287dbcb65f7aa549afe3595493d0c09ca5bac068](https://github.com/dutch-protocol/Protocol-Contracts/commit/287dbcb65f7aa549afe3595493d0c09ca5bac068)

**CODESPECT fix review:** Fixed due to the removal of the hook design.

### [H-05] Inverted sign convention for amountSpecified breaks swap logic

**Files:** [`DUTCHBondingHook.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DUTCHBondingHook.sol#L905)

**Description:**

The `_executeBuy(...)` and `_executeSell(...)` functions interpret the sign of `params.amountSpecified` to determine swap type. In Uniswap V4, the sign convention is:

- Negative: Exact Input → User specifies exact amount to pay;
- Positive: Exact Output → User specifies exact amount to receive.

The hook inverts this convention:

```solidity
function _executeBuy(...) internal returns (...) {
    // ...
    // @audit inverted - positive should be exact output, not exact input
    if (params_.amountSpecified > 0) {
        // Hook treats this as "Exact Input"
        inputAmount_ = uint256(params_.amountSpecified);
        // ...
    } else {
        // Hook treats this as "Exact Output"
        outputAmount_ = uint256(-params_.amountSpecified);
        // ...
    }
}
```

The V4 source code confirms the correct convention in `lib/v4-core/src/libraries/Hooks.sol`.

**Impact:** This causes the specified and unspecified amounts to be swapped together in the `afterSwap` hook in V4, eventually causing the transaction to revert.

**Recommendation:** Invert the condition to match V4's sign convention:

```solidity
if (params_.amountSpecified < 0) {
    // Exact Input - user specifies input amount (negative in V4)
    inputAmount_ = uint256(-params_.amountSpecified);
    // ... exact input logic
} else {
    // Exact Output - user specifies output amount (positive in V4)
    outputAmount_ = uint256(params_.amountSpecified);
    // ... exact output logic
}
```

Same fix can be applied ot `_executeSell(...)`.

**Status:** Fixed

**Client response:** Fixed in commit [c79da4114ffd2b47c5fd313bfd38ec1ca8540044](https://github.com/dutch-protocol/Protocol-Contracts/commit/c79da4114ffd2b47c5fd313bfd38ec1ca8540044)

**CODESPECT fix review:** Fixed due to the removal of the hook design.

### [H-06] Tax fee revenue may be lost due to Uniswap V4 trading

**Files:** [`DUTCHBondingHook.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DUTCHBondingHook.sol#L985)

**Description:**

By design, mainstream DEXs are expected to be blacklisted for DUTCH tokens to restrict off-market trading, ensuring that all trades go through the `DUTCHBondingHook` and incur a tax fee. The `DUTCHBondingHook` overrides `_beforeSwap(...)` so that when a user trades via Uniswap V4 swap, the calculation does not go through the AMM but is performed using a custom curve.

```solidity
function _executeBuy(...) internal returns (...) {
    //...
    // Take WETH from pool (user already deposited it).
    poolManager.take(_wethCurrency, address(this), inputAmount_);

    // Distribute tax.
    (uint256 vaultAmount_, uint256 opsAmount_) =
        _calculateBuyTax(taxAmount_);
    IERC20(Currency.unwrap(_wethCurrency)).safeTransfer(
        _dutchVault,
        vaultAmount_
    );
    IERC20(Currency.unwrap(_wethCurrency)).safeTransfer(
        _opsWallet,
        opsAmount_
    );

    // Mint DUTCH to hook.
    dutchToken.mint(address(this), outputAmount_);
    //...
}
```

During the purchase process, `DUTCHBondingHook` calls `take(...)` to withdraw WETH from the pool, mints DUTCH to send to the pool, and calls `settle(...)` to update the account. The Hook `accountDelta` are transferred to the sender in `afterSwap`. Afterwards, the sender must repay the WETH to the Hook and withdraw DUTCH from the pool. The above process prevents the Uniswap V4 `PoolManager` from being blacklisted for DUTCH tokens, because this contract must handle sending tokens to the buyer during the purchase process.

**Impact:** This allows off-market trading to occur on Uniswap V4. Users can create DUTCH-related pools and trade without using the `DUTCHBondingHook`, causing the protocol to lose tax fee revenue.

**Recommendation:** It is recommended to send DUTCH directly to the user during the purchase process, instead of involving the `PoolManager` in the DUTCH token transfer. This way, the `PoolManager` can be blacklisted to prevent off-market trading on Uniswap V4.

**Status:** Fixed

**Client response:** Fixed in [9da861422e64236acfe61feb3efcc0ccbbe5f49a](https://github.com/dutch-protocol/Protocol-Contracts/commit/9da861422e64236acfe61feb3efcc0ccbbe5f49a)

**CODESPECT fix review:** Fixed. Since the Hook design was removed, Uniswap V4 can be added to the blacklist.

### [H-07] settleWithETH(...) function call will revert because of ETH refunds

**Files:** [`DUTCHBondingHook.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DUTCHBondingHook.sol), [`DutchAuctionMarketplace.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchAuctionMarketplace.sol)

**Description:**

`settleWithETH(...)` forwards all `msg.value` to `swapETHForDUTCH(...)`, which refunds unused ETH to `DutchAuctionMarketplace` after the swap. This will cause NFT purchases with ETH to revert because the `DutchAuctionMarketplace` contract doesn't implement `receive()` or `fallback()` functions.

**Impact:** `settleWithETH(...)` function DoS.

**Recommendation:** Implement a `receive()` function in the `DutchAuctionMarketplace` contract.

**Status:** Fixed

**Client response:** Fixed in [ac0dda10392af280b44d249e70230e45660fc793](https://github.com/dutch-protocol/Protocol-Contracts/commit/ac0dda10392af280b44d249e70230e45660fc793)

## Medium Risk

### [M-01] Bonding-curve taxes and burn revenue cannot be distributed within the collection

**Files:** [`DUTCHBondingHook.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DUTCHBondingHook.sol)

**Description:**

Bonding-curve taxes and burn revenue are sent to the `DutchVault`. These revenues are distributed according to the basis-point weights configured by the owner, increasing the available funds for NFT purchases in each collection. This process is carried out in the `receive()` function.

```solidity
function withdrawExcessReserve(uint256 amount_) external onlyOwner {
    //...
    IERC20(Currency.unwrap(_wethCurrency)).safeTransfer(
        _dutchVault,
        amount_
    );
    emit ExcessReserveWithdrawn(amount_, _dutchVault);
}

function _executeBuy(...) internal returns (...) {
    //...
    // Distribute tax.
    (uint256 vaultAmount_, uint256 opsAmount_) =
        _calculateBuyTax(taxAmount_);
    IERC20(Currency.unwrap(_wethCurrency)).safeTransfer(
        _dutchVault,
        vaultAmount_
    );
    //...
}

function _executeSell(...) internal returns (...) {
    //...
    // Distribute tax.
    (uint256 vaultAmount_, uint256 opsAmount_) =
        _calculateBuyTax(taxAmount_);
    IERC20(Currency.unwrap(_wethCurrency)).safeTransfer(
        _dutchVault,
        vaultAmount_
    );
    //...
}
```

However, when sending bonding-curve taxes and burn revenue, WETH is transferred directly instead of being unwrapped into ETH.

**Impact:** Bonding-curve taxes and burn revenue are sent as WETH to the `DutchVault`, and there is no function in `DutchVault` to unwrap WETH. This causes the revenue distribution to not actually occur, leaving the WETH stuck.

**Recommendation:** It is recommended to unwrap the revenue into ETH before sending it to the `DutchVault`.

**Status:** Fixed

**Client response:** Fixed in commit [734d928fa3344927a8ac14a402778486cdc8e6cf](https://github.com/dutch-protocol/Protocol-Contracts/commit/734d928fa3344927a8ac14a402778486cdc8e6cf)

### [M-02] Inventory costETH records max price instead of actual cost causing cascading errors

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol), [`AuctionAssist.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/AuctionAssist.sol)

**Description:**

When `_buyNFTInternal(...)` purchases an NFT, it records `costETH` as the maximum specified `value_` parameter and passes this same inflated value to `AuctionAssist`. If the marketplace refunds excess ETH, the recorded cost does not reflect the actual price paid.

```solidity
function _buyNFTInternal(...) internal {
    // ...
    // @audit Records max price before purchase executes
    _inventory[collection_][tokenId_] = InventoryRecord({
        // ...
        costETH: value_, // Max price, not actual
        // ...
    });

    // Marketplace call - may refund excess
    (bool success,) = marketplace_.call{value: value_}(marketplaceCalldata_);
    // Refund goes to receive() but costETH is never updated

    // @audit Passes same inflated value to AuctionAssist
    if (_auctionAssist != address(0)) {
        uint256 purchaseId_ = IAuctionAssist(_auctionAssist).recordPurchase(
            collection_, tokenId_, value_ // Inflated cost
        );
    }
}
```

**Impact:** The inflated `costETH` propagates through multiple calculations:

1. AuctionAssist Contributor Shares (`recordPurchase`):

   ```solidity
   uint256 shareBPS_ = (contributionAmount_ * 10000) / costETH_;
   // @audit Inflated denominator reduces contributor shares
   ```

2. Listing Prices (`_listNFTOnMarketplace`):

   ```solidity
   uint256 maxPrice_ = (costETH_ * _upperAuctionMultiplierBps) / _BPS_DENOMINATOR;
   uint256 minPrice_ = (costETH_ * _lowerAuctionMultiplierBps) / _BPS_DENOMINATOR;
   // @audit NFTs listed higher than necessary, reducing sales
   ```

3. Profit/Loss Tracking (`_processSettlement`):

   ```solidity
   bool isProfit_ = salePriceETH_ >= costETH_;
   profitOrLoss_ = isProfit_ ? salePriceETH_ - costETH_ : costETH_ - salePriceETH_;
   // @audit Profits understated, losses overstated
   ```

4. Contributor Share in Settlement (`_splitProceeds`):

   ```solidity
   uint256 contributorShareBPS_ = (totalContributions_ * 10000) / costETH_;
   // @audit Contributors receive diluted reward shares
   ```

**Recommendation:** Track actual ETH spent by measuring the balance before/after the marketplace call.

**Status:** Fixed

**Client response:** Fixed in [ffa02072c6b0c681673c8bdc44de8e810bcbeb20](https://github.com/dutch-protocol/Protocol-Contracts/commit/ffa02072c6b0c681673c8bdc44de8e810bcbeb20)

### [M-03] Marketplace refunds misattributed across collections causing contributor reward loss

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol)

**Description:**

When `_buyNFTInternal(...)` purchases an NFT, it deducts the full specified `value_` from the target collection's balance before executing the marketplace call. If the actual purchase price is less than `value_`, the marketplace refunds the excess ETH to the vault.

```solidity
function _buyNFTInternal(...) internal {
    // ...
    // Full value deducted from specific collection
    _collectionBalances[collection_] -= value_;

    // Marketplace call - may refund excess
    (bool success,) = marketplace_.call{value: value_}(marketplaceCalldata_);
    // ...
}
```

The refund arrives at the vault's `receive()` function, which distributes incoming ETH proportionally across ALL collections based on their allocation percentages:

```solidity
receive() external payable {
    // ...
    for (uint256 i; i < _collectionList.length; ++i) {
        address collection_ = _collectionList[i];
        uint16 allocation_ = _allocations[collection_];
        uint256 share_ = (msg.value * allocation_) / _BPS_DENOMINATOR;
        _collectionBalances[collection_] += share_; // All collections receive share
    }
    // ...
}
```

**Impact:** Contributors who funded Collection A through `AuctionAssist` receive reduced rewards because their collection's balance is understated. Meanwhile, Collection B contributors receive unearned gains. Over multiple purchases with refunds, this drift accumulates, causing systematic unfairness to active collection contributors.

**Recommendation:** Track the actual ETH spent and credit any refund back to the originating collection.

**Status:** Acknowledged

**Client response:** Issue fixed: [a970cba006ae1bbc0145015bd7b8495d47d359e5](https://github.com/dutch-protocol/Protocol-Contracts/commit/a970cba006ae1bbc0145015bd7b8495d47d359e5)

**CODESPECT fix review:** This fix is incorrect and may lead to double accounting. For example:

1. The vault sends 1 ETH to `marketplace_` (`value = 1 ETH`);
2. Purchasing the NFT costs 0.95 ETH, with a 0.05 ETH refund;
3. The refund triggers the `receive` function, which distributes it across collections;
4. After the call completes, `collectionBalances[collection_]` records an additional 0.05 ETH refund again.

**Client response:** I think for simplicity, we are fine with redistributing the refund across all collections. The refund will be small and in many cases, there will be no refund at all. Fixed in: [PR-126](https://github.com/dutch-protocol/Protocol-Contracts/pull/126)

### [M-04] Stuck NFTs block rebalancing and write-off workaround causes contributor reward loss

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol)

**Description:**

The `_rebalanceCollections(...)` function prevents removing any collection that still holds NFTs. If a collection has even one NFT in inventory, it cannot be removed from the allocation list.

```solidity
function _rebalanceCollections(...) internal {
    // Mark new collections temporarily using max value.
    for (uint256 i; i < collections_.length; ++i) {
        _allocations[collections_[i]] = type(uint16).max;
    }

    for (uint256 i; i < _collectionList.length; ++i) {
        address oldCollection_ = _collectionList[i];

        // @audit If ANY collection being removed has NFTs, entire rebalance reverts
        if (
            _allocations[oldCollection_] != type(uint16).max
                && _itemsHeld[oldCollection_] > 0
        ) {
            revert CollectionHasNFTs();
        }
        // ...
    }
}
```

If an NFT becomes unsellable (marketplace delists it, collection rugpulls, floor price crashes, legal issues), that collection can never be removed from allocations. This blocks ALL rebalancing operations for the entire vault.

**Workaround causes contributor loss:** The owner can call `recordSettlement(collection, tokenId, 0, 0)` to manually "settle" a stuck NFT at zero price. However, this causes real fund loss:

1. Contributor reward loss: If `AuctionAssist` contributors funded the NFT purchase with ETH, settling at 0 means `salePriceDUTCH_ = 0`, so contributors receive 0 DUTCH rewards despite funding the purchase;
2. Vault retains asset: The vault still physically holds the NFT (which may have value), but contributors get nothing;
3. Inflated losses: Full `costETH` is recorded as realized loss, overstating the protocol's trading losses;
4. No recovery path: If the NFT is later sold through other means, there's no mechanism to credit contributors retroactively.

**Impact:** This can prevent rebalancing from being possible. And the workaround causes the vault to incur total loss of the asset.

**Recommendation:** Add a dedicated `writeOffNFT(...)` function that cleanly handles stuck inventory.

**Status:** Fixed

**Client response:** Fixed in commit [f88609069273aeb2cbb8df62dcd950018da95d21](https://github.com/dutch-protocol/Protocol-Contracts/commit/f88609069273aeb2cbb8df62dcd950018da95d21)

### [M-05] The contribute(...) function may be DoS

**Files:** [`AuctionAssist.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/AuctionAssist.sol#L272)

**Description:**

In `AuctionAssist`, each collection can have a maximum of 100 contributors to prevent excessive loops from causing gas exhaustion.

```solidity
function contribute(address collection_) external payable nonReentrant whenNotPaused{
    //...
    if (!_hasContributed[collection_][msg.sender]) {
        // Check max contributors limit.
        if (_collectionContributors[collection_].length >= MAX_CONTRIBUTORS)
        {
            revert MaxContributorsExceeded();
        }
        //...
    }
}
```

However, a malicious actor can switch accounts and contribute 1 wei 100 times to reach the contributor limit, preventing funds from being raised.

**Impact:** The fund-raising functionality of `AuctionAssist` may be vulnerable to a DoS attack.

**Recommendation:** It is recommended to refactor the code logic and use alternative methods to avoid excessive gas consumption.

**Status:** Fixed

**Client response:** Fixed by commit [8e583e00553d9178e8204bacda50fdfcaffae9f4](https://github.com/dutch-protocol/Protocol-Contracts/commit/8e583e00553d9178e8204bacda50fdfcaffae9f4)

**CODESPECT fix review:** We believe this issue has only been partially fixed. Although setting a minimum contribution amount mitigates dust attacks to some extent, `AuctionAssist` may still become unusable in the long run. The reasons are:

- Each collection allows a maximum of 100 contributors.
- The contributors array has no cleanup mechanism and can only continue to grow. In this situation, even if some contributors' balances are reduced to zero, they still occupy slots in the array. Over time, this may prevent new valid contributors from being added, ultimately blocking the contract's functionality. Additionally, an attacker can still increase the array length by repeatedly calling `contribute` and then immediately `withdrawContribution`.

**CODESPECT fix review recommendations:**

- Add a cleanup mechanism to remove contributors from the array when their balances are zeroed.
- Additionally, provide an admin-controlled queue cleanup mechanism to manually remove malicious contributor records when necessary.

**Client response:** Fixed in [PR-124](https://github.com/dutch-protocol/Protocol-Contracts/pull/124)

**CODESPECT fix review:** The above fix initially establishes a clearing mechanism, and it is also recommended to add an admin clearing mechanism for withdrawing small `_contributions`. Additionally, when processing refunds, if sending ETH fails in admin clear, the funds that were not successfully sent are stored in a mapping to allow later claims, preventing DoS.

**Client response:** Good call, pushed into the same PR [e04b86410fcf5c6d066d4908c67b968071864f08](https://github.com/dutch-protocol/Protocol-Contracts/pull/124/changes/e04b86410fcf5c6d066d4908c67b968071864f08)

### [M-06] The swapETHForDUTCH function does not charge royalty fees

**Files:** [`DUTCHBondingHook.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DUTCHBondingHook.sol#L734)

**Description:**

Purchasing DUTCH through Uniswap V4 requires paying a 10% tax fee. The `swapExactETHForDUTCH(...)` function, used in `DutchAuctionMarketplace` to swap ETH for DUTCH when buying NFTs, does not incur the tax fee.

```solidity
function swapETHForDUTCH(...)
    external
    payable
    whenNotPaused
    nonReentrant
    returns (uint256 ethUsed_)
{
    //...
    // 1. Calculate ETH needed (no tax for marketplace).
    ethUsed_ = _calculateMintCost(exactDutchAmount_);
    //...
}
```

However, in `DutchAuctionMarketplace`, listing and settlement lack access control. A user can list their own NFT, set `minPrice` to 0 to avoid paying the listing fee, and then call `settleWithETH(...)` to buy their own listed NFT. The ETH they input will be swapped for DUTCH with 0 tax fee, and they only need to pay a 2.5% marketplace fee. This allows them to acquire DUTCH while bypassing most of the tax fee.

**Impact:** `swapExactETHForDUTCH(...)` could be exploited to acquire DUTCH at a lower fee.

**Recommendation:** It is recommended to apply the tax fee in `swapETHForDUTCH(...)` for listings that are not from `DutchVault`.

**Status:** Fixed

**Client response:** Fixed in commit [2dfad0d5e39282955bd2a1098d5d9d249be003ff](https://github.com/dutch-protocol/Protocol-Contracts/commit/2dfad0d5e39282955bd2a1098d5d9d249be003ff)

**Client response:** The preview fix is reverted and a new is implemented in [PR-99](https://github.com/dutch-protocol/Protocol-Contracts/pull/99). Previously, we added tax to all `settleWithETH` purchases. That is undesired behaviour. In the new PR, we keep no tax, but only protocol listings can be bought with ETH. This prevents this attack vector. Also, in this PR we add a view method on the hook to get the current no tax ETH needed for the purchase.

**Client response:** Fixed in [ee8512b51c68567c69237d215e56754d2cd2ca96](https://github.com/dutch-protocol/Protocol-Contracts/commit/ee8512b51c68567c69237d215e56754d2cd2ca96)

### [M-07] User’s contribution can be rebalanced to other collections

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol)

**Description:**

The `setCollectionAllocations(...)` function has an option to rebalance the funds through collections but it doesn't check if there are any user contributions to the collection, nor it modifies user's contribution data in `AuctionAssist`. Users' contributions are bound to the collection address they have contributed to, and if admin decides to rebalance that allocation to other collections user contribution mapping will still point to the old inactive collection. This will cause users not being able to claim the rewards for their contribution, and because there is no option to withdraw funds from inactive collections, users will lose their contributions.

**Impact:** Users can lose 100% of their contributions.

**Recommendation:** Consider excluding user contributions from rebalancing calculations and adding an option for users to withdraw their funds from inactive collections.

**Status:** Fixed

**Client response:** Fixed in [ceac1ccf5ba85cbac5e71f160a58959c3aed6207](https://github.com/dutch-protocol/Protocol-Contracts/commit/ceac1ccf5ba85cbac5e71f160a58959c3aed6207)

**Client response:** Regression bug fixed here: [Protocol-Contracts issue #114](https://github.com/dutch-protocol/Protocol-Contracts/issues/114)

## Low Risk

### [L-01] Bad ownership check will prevent DutchVault from buying NFTs that were purchased in the past

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol)

**Description:**

The `_buyNFTInternal(...)` function performs following NFT ownership check:

```solidity
InventoryRecord storage existing_ = _inventory[collection_][tokenId_];
if (existing_.costETH > 0) {
    revert NFTAlreadyOwned();
}
```

This will cause `DutchVault` being unable to buy back and sell NFTs that were purchased in the past because of the previously created `_inventory[collection_][tokenId_]` record. The check also creates attack surface by transfering NFT directly to the contract or making the `DutchVault` to purchase the NFT for free.

**Impact:** Vault will be unable to buy the same NFT again.

**Recommendation:** The check should look like this:

```solidity
if (IERC721(collection_).ownerOf(tokenId_) == address(this)) {
    revert NFTAlreadyOwned();
}
```

**Status:** Fixed

**Client response:** It is now possible to buy the same NFT multiple times [2346c1e0379cba853a153f9914a0011ea07b84af](https://github.com/dutch-protocol/Protocol-Contracts/commit/2346c1e0379cba853a153f9914a0011ea07b84af)

### [L-02] Failed vesting stream creation causes permanent loss of contributor funds

**Files:** [`Presale.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/Presale.sol)

**Description:**

The `_createVestingStreams(...)` function iterates through all contributors to create Sablier vesting streams for their token allocations. If stream creation fails for any contributor, the function catches the error, emits an event, and continues to the next contributor. However, there is no fallback mechanism to recover the funds of the failed contributor.

```solidity
function _createVestingStreams() internal {
    // ...
    for (uint256 i = 0; i < contributorCount; i++) {
        address contributor = contributors[i];
        uint256 userTokenAllocation = (totalTokens * userContribution) / totalFundedAmount;

        // ...

        try sablierV2.createWithDurationsLL(params, unlockAmounts, durations)
            returns (uint256 streamId) {
            vestingStreamIds[contributor] = streamId;
            streamsCreated++;
            emit TokensClaimed(contributor, userTokenAllocation);
        } catch {
            // @audit Tokens stuck - no recovery mechanism
            emit StreamCreationFailed(contributor, userTokenAllocation);
            continue;

            // ! no fallback method to return the funds to the user
        }
    }
}
```

**Impact:** Contributors who experience failed stream creation permanently lose both their ETH contribution and their token allocation. The tokens remain stuck in the `Presale` contract with no recovery path.

**Recommendation:** Implement a fallback claiming mechanism for failed streams.

**Status:** Fixed

**Client response:** Commit [b6ec15a6aa038829722c1d34079aba58cdcaca6b](https://github.com/dutch-protocol/Protocol-Contracts/commit/b6ec15a6aa038829722c1d34079aba58cdcaca6b)

### [L-03] Fake NFT contract attack due to missing ownership verification

**Files:** [`DutchAuctionMarketplace.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchAuctionMarketplace.sol#L1013)

**Description:**

The `_settleInternal(...)` function transfers an NFT from the seller to the buyer when an auction is settled. After calling `transferFrom(...)` on the NFT contract, the function burns the buyer's DUTCH tokens as payment.

```solidity
function _settleInternal(...) internal {
    // ...
    // @audit no ownership verification after transfer
    IERC721(listing_.nftContract).transferFrom(
        listing_.seller, buyer_, listing_.tokenId
    );

    // Buyer's DUTCH tokens are burned regardless of transfer success
    _DUTCH.burn(buyer_, listing_.currentPrice);
    // ...
}
```

However, there is no verification that the NFT transfer actually succeeded.

**Impact:** A malicious seller can deploy a fake ERC721 contract with a `transferFrom(...)` function that always returns success but never actually transfers any tokens. When a victim bids on this fake NFT listing:

1. The attacker creates a listing with their fake NFT contract;
2. The victim places a bid on what appears to be a legitimate NFT;
3. On settlement, the fake `transferFrom(...)` succeeds but transfers nothing;
4. The victim's DUTCH tokens are burned as payment;
5. The attacker receives the payment while the victim receives nothing.

This results in a complete loss of the buyer's DUTCH tokens with no recourse.

**Recommendation:** Add ownership verification after the transfer:

```solidity
IERC721(listing_.nftContract).transferFrom(
    listing_.seller, buyer_, listing_.tokenId
);

require(
    IERC721(listing_.nftContract).ownerOf(listing_.tokenId) == buyer_,
    "NFT transfer failed"
);
```

**Status:** Fixed

**Client response:** Fixed in [7b9105de3aaeb5808a76caeb75ef0016bc9777b1](https://github.com/dutch-protocol/Protocol-Contracts/commit/7b9105de3aaeb5808a76caeb75ef0016bc9777b1)

### [L-04] Inappropriate slippage control is present in settleWithETH(...)

**Files:** [`DutchAuctionMarketplace.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchAuctionMarketplace.sol#L790)

**Description:**

`settleWithETH(...)` allows an NFT buyer to send ETH to purchase an NFT. The ETH is converted into DUTCH via the `DUTCHBondingHook` to settle for users in `AuctionAssist`.

```solidity
function settleWithETH(uint256 listingId_, uint256 maxPriceETH_) external payable whenNotPaused nonReentrant
{
    //...
    // 5. Get current NFT price, convert to DUTCH, and check slippage.
    CurrentPrice memory currentPrice_ = _getCurrentPrice(listingId_);
    if (currentPrice_.priceETH > maxPriceETH_) revert SlippageExceeded();

    // 6. Mark as settled (before external calls).
    listing_.settled = true;

    // 7. Swap ETH for exact DUTCH amount via bonding hook.
    // Hook will refund excess ETH back to this contract.
    uint256 ethUsed_ = _bondingHook.swapETHForDUTCH{value: msg.value}(
        currentPrice_.priceDUTCH,
        address(this)
    );
    //...
}
```

However, during the slippage check, the NFT's ETH price is used directly for comparison. The slippage from the DUTCH price in the transaction is not accounted for and cannot be controlled via parameters.

**Impact:** The actual ETH paid by the user may exceed the slippage parameter `maxPriceETH_`.

**Recommendation:** It is recommended to perform the slippage check by comparing `ethUsed_` with `maxPriceETH_`.

**Status:** Fixed

**Client response:** Fixed in [1b78ffeecea23ba20aaec534a580b35366dc6bda](https://github.com/dutch-protocol/Protocol-Contracts/commit/1b78ffeecea23ba20aaec534a580b35366dc6bda)

### [L-05] It’s possible to create duplicate listings in DutchAuctionMarketplace

**Files:** [`DutchAuctionMarketplace.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchAuctionMarketplace.sol)

**Description:**

The `_createListing(...)` function doesn't check if the `tokenId_` of the `nftContract_` is currently listed or not and lets users to create duplicate listings, which will remain in the marketplace even after the NFT will be sold.

**Impact:** This will cause gas griefing on users that will try to purchase the NFT using duplicate listings.

**Recommendation:** Check if the NFT is already listed in the `DutchAuctionMarketplace`.

**Status:** Fixed

**Client response:** Fixed in [28b610ddfda26c7d47f0915e79de67ed57b2bb53](https://github.com/dutch-protocol/Protocol-Contracts/commit/28b610ddfda26c7d47f0915e79de67ed57b2bb53)

### [L-06] The presale(...) function may be DoS

**Files:** [`Presale.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/Presale.sol#L366)

**Description:**

The presale is expected to be the first DUTCH purchase transaction to prevent users from frontrunning and buying DUTCH at the lowest price for profit. Therefore, when presale calculating the expected DUTCH output, a supply of 0 is used as the starting point by default. To prevent DUTCH purchases before the presale, the project team needs to pause the system. However, during the process of unpausing the system and calling `performUpkeep(...)` to settle the presale, if a malicious actor frontruns by calling `settleWithETH` or making a purchase, causing the DUTCH supply to be nonzero. Then `performUpkeep(...)` could fail due to slippage control.

**Impact:** `performUpkeep(...)` may fail due to frontrunning, causing the presale funds to be locked.

**Recommendation:** It is recommended to add an `isPresaleEnd` field in `DUTCHBondingHook` and prohibit calls to `_beforeSwap` and `swapETHForDUTCH` while this field is false. The field would be set to true after `swapExactETHForDUTCH` is called.

**Status:** Fixed

**Client response:** Fixed in commit [03c6753d1d7677f2d16d4aaf0b33e7238256d4d1](https://github.com/dutch-protocol/Protocol-Contracts/commit/03c6753d1d7677f2d16d4aaf0b33e7238256d4d1)

### [L-07] Unused vaultFee

**Files:** [`DutchAuctionMarketplace.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchAuctionMarketplace.sol#L991)

**Description:**

In `DutchAuctionMarketplace`, selling an NFT incurs a `_sellerFeeOnSettled`. In the `_settleInternal(...)` function, a portion of `_dutchToken` is sent to the `dutchVault` as `vaultFee_`.

```solidity
function _settleInternal(...) internal {
    //...
    if (vaultFee_ != 0) {
        IERC20(address(_dutchToken)).safeTransfer( //NOTE
            _dutchVaultAddress, vaultFee_
        );
    }
    //...
}
```

However, in `dutchVault`, this portion of `_dutchToken` fees is not utilized.

**Impact:** The `_dutchToken` fees that are sent will be stuck.

**Recommendation:** It is recommended to merge `vaultFee_` into `burnFee` and burn the corresponding `_dutchToken`.

**Status:** Acknowledged

**Client response:** We acknowledge this issue. We are aware, that dutch vault cannot currently handle incoming DUTCH token, but we keep it in the fee split for potential future replacement of `DutchVault` with a new implementation, that would need the DUTCH token to be send to it.

### [L-08] _buyNFTInternal(...) function uses unchecked calldata for NFT purchases

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol)

**Description:**

The `_buyNFTInternal(...)` function receives arbitrary calldata and uses it to purchase NFTs from the marketplaces:

```solidity
(bool success,) =
    marketplace_.call{value: value_}(marketplaceCalldata_);
if (!success) revert MarketplaceCallFailed();
```

The function doesn't validate the arbitrary calldata, which means that any function with any parameters can be called on approved marketplaces. Attack scenario:

1. Attacker waits for approved collection's `getMaxPriceForCollection(...)` to be high enough or an NFT listed below the floor price to pass the `value_ > this.getMaxPriceForCollection(collection_)` check;
2. Calls marketplace function that batch buys NFTs;
3. Besides purchasing the NFT for the vault from approved collection, attacker also buys unvalued NFT that he listed on the marketplace.

**Impact:** Attacker can profit by making the vault buy unvalued NFTs.

**Recommendation:** Validate `marketplaceCalldata_` parameter.

**Status:** Fixed

**Client response:** Fixed in commit [5dd04c73343a03b823f6b1037dc52d60696bb620](https://github.com/dutch-protocol/Protocol-Contracts/commit/5dd04c73343a03b823f6b1037dc52d60696bb620)

**CODESPECT fix review:** Since the unverified calldata may still potentially expand the attack surface, we still recommend using a `(market address => (function signature => bool))` mapping to further restrict callable functions and reduce attack risks.

**Client response:** Fixed in [PR-125](https://github.com/dutch-protocol/Protocol-Contracts/pull/125). We changed the marketplace address allowlist to `[address, selector]` allowlist.

### [L-09] record.listed status is not updated after the NFT is sold

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol)

**Description:**

After the vault's listed NFT is purchased contract doesn't update the `_inventory[collection_][tokenId_].listed` status to false, although it's not listed anymore. This will cause `DutchVault` being unable to list previously sold NFTs again because of following check in `listNFTOnMarketplace(...)`:

```solidity
if (record_.listed) {
    revert NFTAlreadyListed();
}
```

**Impact:** `DutchVault` can't list previously sold NFTs again.

**Recommendation:** Set `_inventory[collection_][tokenId_].listed` to false after NFT got purchased.

**Status:** Fixed

**Client response:** Fixed in [26db416a8d185792721a1da1f9622eb8252f7838](https://github.com/dutch-protocol/Protocol-Contracts/commit/26db416a8d185792721a1da1f9622eb8252f7838)

### [L-10] withdrawCollectionBalance(...) can withdraw user contributions

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol)

**Description:**

The `withdrawCollectionBalance(...)` function doesn't check if the collection has any user contributions before withdrawing the balance. This can cause owner mistakenly withdrawing user contributions.

**Impact:** Users can lose 100% of their contributions.

**Recommendation:** Check if there are any user contributions for the collection and let the owner to only withdraw protocol funded assets.

**Status:** Fixed

**Client response:** Fixed here: [PR-89](https://github.com/dutch-protocol/Protocol-Contracts/pull/89)

## Informational

### [I-01] Improper collection configuration checks

**Files:** [`AuctionAssist.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/AuctionAssist.sol#L259)

**Description:**

In the `contribute(...)` function, the collection configuration is checked by verifying whether `basePrice_` is zero.

```solidity
function contribute(...) external payable nonReentrant whenNotPaused{
    //...
    // 2. Get base price (target price) from DutchVault.
    uint256 basePrice_ = IDutchVault(_dutchVault).getBasePrice(collection_);

    // 3. Validate collection is configured.
    if (basePrice_ == 0) {
        revert CollectionNotConfigured();
    }
    //...
}
```

However, when a collection is removed, `basePrice_` may not be cleared.

**Impact:** This may cause users to mistakenly deposit into an inactive collection.

**Recommendation:** It is recommended to check that `_allocations[collection_]` is not equal to zero.

**Status:** Fixed

**Client response:** Fixed in [e38f1cd269761bb4b5901c321cedb83b2eab515e](https://github.com/dutch-protocol/Protocol-Contracts/commit/e38f1cd269761bb4b5901c321cedb83b2eab515e)

### [I-02] Open TODOs

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol)

**Description:**

There are several TODO sections in the code and it's recommended to complete these TODOs before deployment. In particular, the TODO in `_splitProceeds(...)` function, which states that contribution calculation doesn't exclude new contributions that occurred after the purchase. This will cause `recordAndPullRewards(...)` function being called with inflated `contributorAmount_` value.

**Impact:** Recorded contributors will receive more share from the purchase than they should.

**Recommendation:** Exclude new contributions from the calculation.

**Status:** Fixed

**Client response:** Fixed in [67b3b69cef288936425f47313485286ffb4a87f0](https://github.com/dutch-protocol/Protocol-Contracts/commit/67b3b69cef288936425f47313485286ffb4a87f0)

### [I-03] The price growth has no upper limit

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol#L651)

**Description:**

If `_buyIncrements[collection_]` is configured, the price the protocol can pay for an NFT will increase over time until an NFT in the collection is purchased, at which point it resets.

```solidity
function getMaxPriceForCollection(address collection_)
    external
    view
    returns (uint256 maxPrice_)
{
    //...
    uint256 timeBuffer_ =
        (blocksSinceLastBuy_ + 1) * _buyIncrements[collection_];
    return basePrice_ + timeBuffer_;
}
```

If NFTs are never purchased, the price will continue to rise without any upper limit.

**Impact:** If a collection continuously has no NFT purchases due to no NFTs being listed, the protocol's bid could become far higher than the funds raised.

**Recommendation:** It is recommended to set a limit on price growth.

**Status:** Acknowledged

**Client response:** We acknowledge this finding, but consider the current implementation intentional and safe by design.

The `getMaxPriceForCollection` function represents the protocol's maximum willingness to pay (a market signal), not a guarantee of payment capability. While this value can grow unbounded over time, actual purchases in `_buyNFTInternal` are protected by a critical balance check (`if (_collectionBalances[collection_] < value_) revert InsufficientBalanceForPurchase();`) that ensures the protocol can never spend more than it has, regardless of how high `getMaxPriceForCollection` grows. The unbounded growth in willingness to pay is intentional, and we believe the market should determine pricing without artificial protocol-imposed limits. The real constraint (available treasury funds) is what matters, not an artificial cap on willingness to pay. If `getMaxPriceForCollection` grows beyond available funds, purchases will simply fail until more funds are deposited or the price resets via a purchase, and the protocol owner can always adjust `_buyIncrements[collection_]` or reset `_lastBuyBlock[collection_]` if needed.

### [I-04] _lastBuyBlock is not fully updated

**Files:** [`DutchVault.sol`](https://github.com/dutch-protocol/Protocol-Contracts/tree/cec8b243aa645fdbe80e05e613eca69542b62a2e/src/DutchVault.sol#L499)

**Description:**

`_lastBuyBlock` stores the last purchase block for each collection. The protocol allows the maximum price for purchasing NFTs from that collection to increase over the elapsed blocks. When calling the `setCollectionAllocations(...)` function to update collections, only collections with `_lastBuyBlock` equal to 0 will have their `_lastBuyBlock` updated.

```solidity
function setCollectionAllocations(...) external onlyOwner whenNotPaused {
    //...
    for (uint256 i; i < collections_.length; ++i) {
        _allocations[collections_[i]] = allocationsBPS_[i];
        _collectionList.push(collections_[i]);

        // Initialize lastBuyBlock if not set (prevents huge first-buy cap).
        if (_lastBuyBlock[collections_[i]] == 0) {
            _lastBuyBlock[collections_[i]] = block.number;
        }

        emit AllocationUpdated(collections_[i], allocationsBPS_[i]);
    }
}
```

However, if a collection was previously removed and then added again, its `_lastBuyBlock` will not be 0, so `_lastBuyBlock` will not be updated when it is re-added.

**Impact:** If too much time has passed since `_lastBuyBlock`, the payable price for the collection could become excessively high.

**Recommendation:** It is recommended to update `_lastBuyBlock` for each collection when calling `setCollectionAllocations(...)`.

**Status:** Fixed

**Client response:** Fixed in [f6c7c3e0a42e80ff2cb0d147854a96113a6c276b](https://github.com/dutch-protocol/Protocol-Contracts/commit/f6c7c3e0a42e80ff2cb0d147854a96113a6c276b)
