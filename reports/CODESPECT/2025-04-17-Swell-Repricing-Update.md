**Auditors**

[CODESPECT](https://codespect.net/)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/017_CODESPECT_SWELL_REPRICING_UPDATE.pdf)

# Findings

## Medium Risk

### [M-01] Double Accounting for Slashing

**Files:** [RepricingOracle.sol](https://github.com/SwellNetwork/v3-contracts-lrt/blob/c1c7cde2e8af6fdc68116ee7d236f45f17d9487a/contracts/implementations/RepricingOracle.sol#L306)

**Description:**

The recent update to the `RepricingOracle` contract introduced slashing accounting related to EigenLayer. Swell added two new variables that are part of the snapshot used for updating the price of rswETH:

- `totalSlashedEigenLayer` – represents the total ETH slashed from EigenLayer;
- `totalSlashedBurnedEigenLayer` – represents the portion of slashed ETH that has been permanently removed (burned). Once this variable is updated, the "slashing" is propagated to `balances.executionLayer`, which tracks ETH across the protocol, including EigenPods;

A potential issue arises when slashing is double-accounted — particularly in scenarios where a slashing event has occurred, but the ETH has not yet been burned on the EigenLayer side. The root cause is that the `reserveAsset()` function, which reflects all ETH under Swell’s control, immediately accounts for Beacon Chain slashing (as it tracks the BC state). At the same time, `totalSlashedEigenLayer` may also be incremented, even though `totalSlashedBurnedEigenLayer` has not yet been updated.

In such cases, slashing may be counted twice, especially when calculating `preRewardETHReserves` and `unclampedRewardsPayableForFees`:

```solidity
function _referenceOnChainRate(...) internal view returns (uint256) {
    // ...
    unclampedRewardsPayableForFees =
        // @audit reserve change containing BC slashing already
        reserveAssetsChange -
        int256(ethDepositsChange) +
        int256(ethExitedChange) -
        // @audit Slashed ETH at Eigen
        int256(totalSlashedEigenLayerChange) +
        // @audit Slashed ETH at Eigen which was burned (It does not exist in EigenPod.)
        int256(totalSlashedBurnedEigenLayerChange);
    // ...
}


function handleReprice(...) internal {
    // ...
    uint256 preRewardETHReserves = reserveAssets - liabilities - newETHRewards;
    // ...
}
```

*Note: The liabilities in the snippet above contains the sum of `lockedShares` (`totalSlashedEigenLayerChange - totalSlashedBurnedEigenLayerChange`) and `exitingETH`.*

Additionally, EigenLayer has reported a [DoubleSlashingEdgeCase](https://github.com/Layr-Labs/eigenlayer-contracts/blob/dev/docs/core/accounting/DualSlashingEdgeCase.md), which should be considered. Depending on the order of EigenPod actions during simultaneous BC and AVS slashing, the amount of withdrawable ETH for users can vary.

**Impact:** Incorrect pricing and potential loss of value for rswETH.

**Recommendation:** Consider changing the repricing logic to handle edge cases where slashed shares on EigenLayer are not yet burned at the time of repricing.

**Status:** Fixed

**Client response:** Resolved in commit [b7a2975d2a3e4a946d506397476ec755b30e97b3](https://github.com/SwellNetwork/v3-contracts-lrt/pull/109/commits/b7a2975d2a3e4a946d506397476ec755b30e97b3).

We’ve implemented a solution to properly handle dual slashing scenarios between the Beacon Chain and EigenLayer:

1. Introduced a new accounting variable (`totalDualSlashedEigenLayer`) and modified the liability calculation to reduce the effective liability when ETH is slashed in both systems.
2. Added constraints to ensure accounting accuracy while maintaining flexibility for edge cases outlined in EigenLayer’s dual slashing documentation.

**CODESPECT fix review:** Swell introduced a new variable to address the drafted issue. However, due to the centralisation of the process, Swell is responsible for accurately reporting these values.
