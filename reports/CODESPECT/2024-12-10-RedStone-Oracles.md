**Auditors**

[Talfao](https://x.com/talfao1)

[Bloqarl](https://x.com/TheBlockChainer)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/001_CODESPECT_REDSTONE_AUDIT_ORACLES.pdf)

# Findings

## Medium Risk

### [M-01] Potential Scaling to Unexpected Decimal Places

**Files:** [MultiFeedAdapterWithoutRounds.sol](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/ff0f3dcb085f28bd80ddc096825701db6e14d0af/packages/on-chain-relayer/contracts/price-feeds/without-rounds/MultiFeedAdapterWithoutRounds.sol#L231-L232)

**Description:**

The `MultiFeedAdapterWithoutRounds` contract allows updates and price retrieval for multiple feeds. One way to obtain an asset’s price is through the `priceOf(...)` function, which accepts the asset’s address as input:

```solidity
function priceOf(address asset) public view virtual returns (uint256) {
    bytes32 dataFeedId = getDataFeedIdForAsset(asset);
    uint256 latestValue = getValueForDataFeed(dataFeedId);
    return convertDecimals(dataFeedId, latestValue);
}
```

This function retrieves the data feed ID using the asset address, gets the latest price, and then scales it to 10^18 decimals using:

```solidity
function convertDecimals(bytes32 /* dataFeedId */, uint256 valueFromRedstonePayload) public view virtual returns (uint256) {
    // @audit-issue Missing address(dataFeed).decimals() and proper scaling to reflect correct decimals
    return valueFromRedstonePayload * DEFAULT_DECIMAL_SCALER_LAYERBANK;
}
```

The issue is that the price is always multiplied by 10^10, assuming a default of 10^8 decimals from the `PriceFeed`. However, the function does not consider the actual decimals of the specific data feed. This could lead to a mismatch where the smart contract receives a price in a different decimal format than expected, potentially causing incorrect calculations.

**Recommendation:** Incorporate the data feed’s `decimals()` function to adjust the scaling based on the returned value dynamically.

**Status:** Acknowledged

**Client response:** We’ve assumed we’ll override the `convertDecimals` function fo such cases. In practice, this LayerBank interface is in use only by one project and we’ll probably remove it in future.

## Informational

### [I-01] Compiler Version with Assembly Bugs

**Original severity:** Best Practices

**Files:** [on-chain-relayer/*.sol](https://github.com/redstone-finance/redstone-oracles-monorepo/tree/ff0f3dcb085f28bd80ddc096825701db6e14d0af/packages/on-chain-relayer)

**Description:**

The the `on-chain-relayer` package rely on compiler version (`^0.8.14`) known to have bugs affecting assembly code blocks. While your current code does not appear to be impacted, it is strongly recommended to upgrade to the latest Solidity version or at least to `^0.8.15`, which addresses the bugs detailed [here](https://soliditylang.org/blog/2022/06/15/solidity-0.8.15-release-announcement/).

**Recommendation:** Upgrade the Solidity compiler to the latest version to ensure the integrity and security of the contracts.

**Status:** Fixed

**Client response:** Fixed in [198c17ee5123fbcc654b9490c8ea2d0857638705](https://github.com/redstone-finance/redstone-oracles-monorepo/commit/198c17ee5123fbcc654b9490c8ea2d0857638705)

### [I-02] Unused Constant

**Original severity:** Best Practices

**Files:** [RedstoneConstants.sol](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/ff0f3dcb085f28bd80ddc096825701db6e14d0af/packages/evm-connector/contracts/core/RedstoneConstants.sol#L21)

**Description:**

The `RedstoneConstants` contract defines various constants utilized across the `evm-connector` package. However, the constant `FUNCTION_SIGNATURE_BS` is not used in any contract.

Removing unused code is a best practice to maintain code clarity.

**Recommendation:** Consider the removal of the `FUNCTION_SIGNATURE_BS` constant.

**Status:** Fixed

**Client response:** Fixed in [be508fa80152bad5c8a4535a8ab1df18e4bad372](https://github.com/redstone-finance/redstone-oracles-monorepo/commit/be508fa80152bad5c8a4535a8ab1df18e4bad372).

### [I-03] Potential Rounding Down Could Decrease Overall Price

**Files:** [NumericArrayLib.sol](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/ff0f3dcb085f28bd80ddc096825701db6e14d0af/packages/evm-connector/contracts/libs/NumericArrayLib.sol#L17-L26)

**Description:**

The prices for the feeds are calculated as the median of all values. The median requires a sorted array of elements, taking the middle value. However, when the number of elements is even, the median is calculated as the arithmetic average of the two middle elements:

```solidity
uint256 sum = arr[middleIndex - 1] + arr[middleIndex];
return sum / 2;
```

Since Solidity does not support decimal numbers, some precision may be lost during this operation due to rounding down which could lead to different prices than was expected. This behaviour is likely known to the Redstone protocol, but it’s important to note this potential loss in precision to maintain reasonable decimal accuracy for the prices.

By default, Redstone feeds use eight decimals, where the expected precision loss is around 0.0000005%, which is considered negligible.

**Recommendation:** Ensure awareness of this rounding behaviour to maintain appropriate precision in price calculations.

**Status:** Acknowledged

**Client response:** Acknowledged!

### [I-04] MAX_DATA_STALENESS Should Vary Based on Data Feed

**Files:** [MultiFeedAdapterWithoutRounds.sol](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/ff0f3dcb085f28bd80ddc096825701db6e14d0af/packages/on-chain-relayer/contracts/price-feeds/without-rounds/MultiFeedAdapterWithoutRounds.sol#L31)

**Description:**

The `MAX_DATA_STALENESS` constant defines the maximum duration for which price data is considered valid. Once this period expires, the data is marked as stale, and the oracle will revert rather than return this outdated data. The staleness check is conducted in the following function:

```solidity
function _validateLastUpdateDetailsOnRead(
    bytes32 /* dataFeedId */,
    uint256 /* lastDataTimestamp */,
    uint256 lastBlockTimestamp,
    uint256 lastValue
) internal view virtual returns (bool) {
    return lastValue > 0 && lastBlockTimestamp + MAX_DATA_STALENESS > block.timestamp;
}
```

Since different data feeds have varying update frequencies (heartbeats), the staleness period should also be tailored to each data feed type to ensure accurate and reliable data.

**Recommendation:** Implement dynamic staleness periods for different data feeds to accommodate their specific update frequencies.

**Status:** Acknowledged

**Client response:** This function is virtual and allows the add this logic. However, we wanted to have this param quite high, as for the majority of feeds we can’t predict exact update frequency, as usually it depends on the market volatility.
