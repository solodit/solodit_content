**Auditor**

[@IAm0x52](https://twitter.com/IAm0x52)

# Findings

## High Risk

### [H-01] Precision error in \_fetchTWAP causes vesting to be free after initial lockdrop period

**Details**

[BlueberryStaking.sol#L830-L843](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L830-L843)

    // Adjust for decimals
    if (BLB_DECIMALS > _decimalsStable) {
        _priceX96 /= 10 ** (BLB_DECIMALS - _decimalsStable);
    } else if (_decimalsStable > BLB_DECIMALS) {
        _priceX96 *= 10 ** (_decimalsStable - BLB_DECIMALS);
    }

    // Now priceX96 is the price of blb in terms of stableAsset, multiplied by 2^96.
    // To convert this to a human-readable format, you can divide by 2^96:

    uint256 _price = _priceX96 / 2 ** 96;

    // Now 'price' is the price of blb in terms of stableAsset, in the correct decimal places.
    return _price;

TWAP prices from Uniswap V3 oracles return in a 64x96 format. Due to its unusual format these values are notoriously hard to manipulate. Directly above the price is converted from base 2 to base 10. The issue is that this conversion gives a final price with 0 dp. \_fetchTWAP is expected to return the price of blb with 18 dp. This large disparity between the expected and actual precision causes it to be essentially free to vest all tokens immediately.

**Lines of Code**

[BlueberryStaking.sol#L798-L844](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L798-L844)

**Recommendation**

Convert to 18 dp with the following lines:

    if (blbIsToken0) {
        return FullMath.mulDiv(_priceX96, 10 ** (18 + BLB_DECIMALS stableDecimals), 2 ** 96);
    } else {
        uint256 inversePrice = FullMath.mulDiv(_priceX96, 10 ** (18 - BLB_DECIMALS + stableDecimals), 2 ** 96);
        return 10**36 / inversePrice;
    }

**Remediation**

Fixed in [PR#30](https://github.com/Blueberryfi/blueberry-staking/pull/30/). Math has been rebuilt, ensuring proper precision is maintained during conversion.

## Medium Risk

### [M-01] \_fetchTWAP incorrectly assumes that blb is always token0 leading to bad pricing if it isn't

**Details**

[BlueberryStaking.sol#L812-L819](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L812-L819)

    (int56[] memory tickCumulatives, ) = _pool.observe(_secondsArray);

    int56 _tickDifference = tickCumulatives[1] - tickCumulatives[0];
    int56 _timeDifference = int32(_observationPeriod);

    int24 _twapTick = int24(_tickDifference / _timeDifference);

    uint160 _sqrtPriceX96 = TickMath.getSqrtRatioAtTick(_twapTick);

We see above that the Uniswap V3 oracle price is always used directly as returned. Uniswap V3 always returns the price as a ratio of token0/token1. This means token0 is the base token and token1 is the quote token. Additionally Uniswap V3 sorts tokens by address meaning that blb can be either token0 or token1 depending on the stable asset.

In the event that blb is not token0, the price returned will be the inverse of the actual price since expected quote and base tokens are reversed. This will lead to largely incorrect prices that will either damage the user by grossly overcharging them or damage the treasury as vesting will be much cheaper than intended.

**Lines of Code**

[BlueberryStaking.sol#L798-L844](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L798-L844)

**Recommendation**

When adding the pool, check the pair order and store it. When querying the pool, reference back to this variable to determine if the price should be inverted.

**Remediation**

Fixed in [PR#30](https://github.com/Blueberryfi/blueberry-staking/pull/30/). Token order is now cached when setting Uniswap V3 pool. Price calculations have been adjusted to work regardless of token order.

### [M-02] It is impossible to complete any vest unless all vests are mature

**Details**

[BlueberryStaking.sol#L328-L331](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L328-L331)

    uint256 vestIndexLength = vests.length;
    if (vesting[msg.sender].length < vestIndexLength) {
        revert InvalidLength();
    }

The above lines cache the incorrect array length. It should cache the length of \_vestIndexes but instead caches the length of vests.

[BlueberryStaking.sol#L334-L348](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L334-L348)

    for (uint256 i; i < vestIndexLength; ++i) {
        Vest storage v = vests[_vestIndexes[i]];

        if (!isVestingComplete(msg.sender, _vestIndexes[i])) {
            revert VestingIncomplete();
        }

        totalbdblb += v.amount + v.extra;

        // Ensure accurate redistribution accounting for accelerations.
        totalVestAmount -= v.amount;
        redistributedBLB -= v.extra;

        delete vests[_vestIndexes[i]];
    }

This error causes and OOB error when accessing \_vestIndexes unless is it as least the same length as vests. The result is that the user must vest all at the same time or their call will revert. If even a single vest isn't mature then none will be able to be vested.

**Lines of Code**

[BlueberryStaking.sol#L324-L355](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L324-L355)

**Recommendation**

Use `_vestIndexes.length` instead of `vests.length`

**Remediation**

Fixed in [PR#24](https://github.com/Blueberryfi/blueberry-staking/pull/24) as recommended.

### [M-03] Rewards will be permanently lost for ibToken with no deposits

**Details**

[BlueberryStaking.sol#L482-L485](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L482-L485)

    function rewardPerToken(address _ibToken) public view returns (uint256) {
        if (totalSupply[_ibToken] == 0) {
            return rewardPerTokenStored[_ibToken];
        }

[BlueberryStaking.sol#L782-L790](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L782-L790)

    rewardPerTokenStored[_ibToken] = rewardPerToken(_ibToken);
    lastUpdateTime[_ibToken] = lastTimeRewardApplicable(_ibToken);

    if (_user != address(0)) {
        rewards[_user][_ibToken] = _earned(_user, _ibToken);
        userRewardPerTokenPaid[_user][_ibToken] = rewardPerTokenStored[
            _ibToken
        ];
    }

When distributing rewards, the reward rate and elapsed time are used to determine the number of tokens to distribute. If no ibTokens are deposited, rewards cannot be distributed, however the timestamp in this scenario is still updated. This leads all rewards during this period to be lost.

Additionally rewards are added at the same time that ibTokens are registered and allowed to be deposited. This causes a guaranteed loss of some rewards for every ibToken. The longer it goes before the first deposit the more tokens that are lost.

**Lines of Code**

[BlueberryStaking.sol#L777-L791](https://github.com/Blueberryfi/blueberry-staking/blob/efaf7fc690e38914ba475d5ac61a4d0bd3f45c0d/src/BlueberryStaking.sol#L777-L791)

**Recommendation**

The timestamp should not be updated for an ibToken if there is none of that token deposited.

**Remediation**

Fixed in [PR#29](https://github.com/Blueberryfi/blueberry-staking/pull/29) as recommended.
