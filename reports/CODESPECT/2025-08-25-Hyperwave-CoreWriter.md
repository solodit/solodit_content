**Auditors**

[Talfao](https://x.com/talfao1), [JecikPo](https://x.com/jecikpo)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/028_CODESPECT_HYPERWAVE_COREWRITER.pdf)

# Findings

## Low Risk

### [L-01] Blocked whitelist removals for de-listed accounts

**Files:** [`TradeStakeManager.sol`](https://github.com/SwellNetwork/hlp-corewriter/blob/5a19bb4373eaf3eb57d872f81d20177fb5cf9b2b/src/TradeStakeManager.sol#L606), [`VaultManager.sol`](https://github.com/SwellNetwork/hlp-corewriter/blob/5a19bb4373eaf3eb57d872f81d20177fb5cf9b2b/src/VaultManager.sol#L183)

**Description:**

Both `TradeStakeManager` and `VaultManager` maintain separate whitelists to control different operations:

1. `TradeStakeManager` maintains `SpotWhitelist`, `PerpWhitelist`, `ValidatorWhitelist` and `TransferWhitelist`;
2. `VaultManager` maintains `TransferWhitelist` and `VaultWhitelist`;

Those whitelists allow whitelisting of different indexes or addresses per each `HyperCoreAccount` address. It is not possible to update any of those lists without first having the the concrete `HyperCoreAccount` address whitelisted through a separate whitelist - `AccountWhitelist` first. This is also valid for de-listing. Once a `HyperCoreAccount` is de-listed it is not possible to delist anything associated with that account from the above whitelists.

The problem might arise if a `HyperCoreAccount` is de-listed (e.g. due to emergency reasons) and needs to be listed back, yet without certain perps, or indexes, it cannot be done. It must first be whitelisted and only then the perps or indexes can be delisted.

**Impact:** Inability to whitelist an account without previously delisting problematic assets.

**Recommendation:** Allow whitelist removals without checking first if the `HyperCoreAccount` is whitelisted.

**Status:** Fixed

**Client response:** Resolved in [7a406f9e70161ad69884eac8e2f6f4ae84b2cbe1](https://github.com/SwellNetwork/hlp-corewriter/commit/7a406f9e70161ad69884eac8e2f6f4ae84b2cbe1)

### [L-02] Joint withdrawal and spot transfer whitelist could lead to locked funds

**Files:** [`TradeStakeManager.sol`](https://github.com/SwellNetwork/hlp-corewriter/blob/5a19bb4373eaf3eb57d872f81d20177fb5cf9b2b/src/TradeStakeManager.sol#L618)

**Description:**

The `TradeStakeManager` contract uses `transferWhitelist` to validate both:

1. withdrawals from the account’s owned L1 address using `withdraw(...)` and `withdrawNative(...)`;
2. spot transfers using `spotSend(...)` and `spotSendUSDC(...)`;

This leads to a situation where a valid destination account used for `withdraw(...)` or `withdrawNative(...)` could be passed to the `spotSend(...)` or `spotSendUSDC(...)` and still be allowed. This could lead to transfer of the spot balance to an account which the protocol doesn’t have control of, and hence lead to locked funds. While this mistake would have to be made through an off-chain mechanism or the `BoringVault`, which is outside of the scope, it is worth raising the issue due to tight design control of the flow transfer within the protocol.

**Impact:** Potential loss of transferred funds.

**Recommendation:** Create separate whitelists for spot transfers and withdrawals

**Status:** Fixed

**Client response:** Resolved in [4b75b10a6aa5e1b59ed000bc7379af10eacefd67](https://github.com/SwellNetwork/hlp-corewriter/commit/4b75b10a6aa5e1b59ed000bc7379af10eacefd67)

### [L-03] Receive of Native token is enabled while withdrawal is not possible

**Files:** [`TradeStakeManager.sol`](https://github.com/SwellNetwork/hlp-corewriter/blob/5a19bb4373eaf3eb57d872f81d20177fb5cf9b2b/src/TradeStakeManager.sol#L62)

**Description:**

The `TradeStakeManager` contract contains a `receive()` function which allows receiving of the Native HYPE token by the contract.

There is however no option to withdraw the excess Native asset from the `TradeStakeManager`. The existing `withdrawNative(...)` function allows withdrawal of HYPE from `HyperCoreAccount` only.

**Impact:** Native HYPE asset transferred to the contract cannot be withdrawn.

**Recommendation:** Add a function allowing withdrawal of excess HYPE balance from the `TradeStakeManager` contract.

**Status:** Fixed

**Client response:** Resolved in [0096241fb91c1158c3b3526a06049bad08c7b0eb](https://github.com/SwellNetwork/hlp-corewriter/pull/58/commits/0096241fb91c1158c3b3526a06049bad08c7b0eb)

## Informational

### [I-01] Introduce validation of the order size

**Files:** [`TradeStakeManager.sol`](https://github.com/SwellNetwork/hlp-corewriter/blob/5a19bb4373eaf3eb57d872f81d20177fb5cf9b2b/src/TradeStakeManager.sol#L127)

**Description:**

The Hyperliquid core enforces decimal limits on order sizes for both perps and spots.

According to the documentation: *Prices can have up to 5 significant figures, but no more than `MAX_DECIMALS - szDecimals` decimal places, where `MAX_DECIMALS` is 6 for perps and 8 for spot.* [[ref](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/tick-and-lot-size)]

Implementing similar validation in `TradeStakeManager` would help prevent potential silent failures when an incorrect order size is submitted. While a similar limitation already exists for prices — and `TradeStakeManager` enforces price boundaries — the same should be applied to order sizes.

**Impact:** Proper validation can prevent unnecessary silent failures on the Hyperliquid Core side.

**Recommendation:** Validate order sizes in accordance with Hyperliquid documentation to ensure compliance before submission.

**Status:** Acknowledged

**Client response:** We acknowledge and choose not to implement in the contract. If invalid, hypercore will reject the request. There are many other validations we could do and this fits into the same category

### [I-02] Replace storage writes with memory ones

**Files:** [`TradeStakeManager.sol`](https://github.com/SwellNetwork/hlp-corewriter/blob/5a19bb4373eaf3eb57d872f81d20177fb5cf9b2b/src/TradeStakeManager.sol#L103)

**Description:**

Whenever the bounds for native or other tokens are queried, they are assigned to a variable declared as `storage`, e.g.:

```solidity
PriceBounds storage bounds; // @audit Is storage necessary here?
if (requestedNative) {
    spotWhitelisted = _getNativeSpotWhitelist(account);
    bounds = _getNativePriceBounds();
} else {
    spotWhitelisted = _getTokenSpotWhitelist(account, tokenAddress);
    bounds = _getTokenPriceBounds(tokenAddress);
}
```

The `_getTokenPriceBounds(...)` function returns a storage variable. However, in functions such as `placeSellSpotOrder(...)` or `placeBuySpotOrder(...)`, the bounds are only read, not modified. Using `storage` in these cases is unnecessary and increases gas usage. A getter returning a memory copy of the bounds would be more efficient.

**Impact:** Minor gas optimisation; limited effect due to the use of Hyperliquid.

**Recommendation:** Introduce getter functions that return the bounds as `memory` when only reading data, avoiding unnecessary storage references.

**Status:** Acknowledged

**Client response:** We acknowledge and choose not to change, since the benefits are eroded by copying unused fields from storage.

### [I-03] Possible overflow during price tolerance control

**Files:** [`TradeStakeManager.sol`](https://github.com/SwellNetwork/hlp-corewriter/blob/a48131cc4317304b4a7f32d90f5d8404abd6466d/src/TradeStakeManager.sol#L234)

**Description:**

The `6dabdeeec70afc123d694fac0fcbb2d90ddb4c87` commit introduced control on perps price using tolerance parameter which can be set by the `GOVERNOR` role for each `perpIndex`. The price privided by the `OPERATOR` during sell and buy order placing is compared against `markPrice` which is obtained from the HyperCore and is scaled to 8 decimals by the `_getMarkPxForPerp(...)` function.

Below `minPrice` and `maxPrice` calculation can revert during the multiplication part, especially for the last one. In case the `markPrice * (10000 + toleranceBps)` crosses the `type(uint64).max` value, overflow happens.

```solidity
uint64 markPrice = _getMarkPxForPerp(perpIndex);
uint64 minPrice = (markPrice * (10000 - toleranceBps)) / 10000;
uint64 maxPrice = (markPrice * (10000 + toleranceBps)) / 10000;
```

As the `markPrice` is checked inside `_getMarkPxForPerp(...)` to be lower than `type(uint64).max`, such situation can theoretically happen, If the returned `markPrice` is above 922383322851620. That is assuming the maximum possible `toleranceBps` value, which is `19_999`.

**Impact:** Order placement will revert in case the price is sufficiently high. The above number, when interpreted as an 8 decimal USDC price, is $9.223.372 hence it is not likely to happen in the near future; however, if the HyperCore introduces other quote currencies in the future, it might become a problem.

**Recommendation:** Introduce `uint256` casting to the operation, before casting it down to `uint64`.

**Status:** Fixed

**Client response:** Resolved in [4108aed13b26ad0cb70fc42d2dba99eec145af7c](https://github.com/SwellNetwork/hlp-corewriter/pull/58/commits/4108aed13b26ad0cb70fc42d2dba99eec145af7c)
