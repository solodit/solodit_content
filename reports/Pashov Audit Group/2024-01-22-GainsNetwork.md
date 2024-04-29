**Auditor**

[Pashov Audit Group](https://twitter.com/PashovAuditGrp)

# Findings

## Medium Risk

### [M-01] User can't unstake in case he gets blacklisted by reward token USDC

**Severity**

Impact: High. User can't unstake his Gns

Likelihood: Low. USDC blacklisting is not usual operation.

**Description**

USDC is supposed to be used as reward token, note it has blacklist functionality. Thus USDC tokens can't be transferred to/from blacklisted address.

Currently harvested token rewards are transferred in a force manner to staker on any interaction: claim vested tokens, stake, unstake, revoke vesting. Additionally, in `unstakeGns()` all the rewards are processed. Therefore if any of token transfers reverts, user will be unable to harvest and therefore unstake.

While unstake it firstly proccesses rewards for all tokens:

```solidity
    function unstakeGns(uint128 _amountGns) external {
        require(_amountGns > 0, "AMOUNT_ZERO");

        harvestDai();
@>      harvestTokens();

        ... // Main unstake logic
    }
```

And then transfers harvested amount of every reward token, including USDC. As discussed above, USDC transfer can revert:

```solidity
    function _harvestToken(address _token, uint128 _stakedGns) private {
        ... // Main harvest logic

@>      IERC20(_token).transfer(msg.sender, uint256(pendingTokens));

        emit RewardHarvested(msg.sender, _token, pendingTokens);
    }
```

Thus user who got blacklisted by USDC can't harvest all token rewards and therefore unstake. His stake will exist forever, receiving portion of rewards which will never be claimed.

The same scenario is when user tries to claim vested amount.

**Recommendations**

Use claim pattern: instead of transferring tokens, only increase internal balances. Something like:

```solidity
mapping (address token => uint256 pendingBalance) pendingBalances;
```

And also introduce claim method.

## Low Risk

### [L-01] Reward tokens cannot be removed

The new version of `GNSStaking` allows for multiple reward tokens. The contract owner adds new reward tokens through `addRewardToken`. There is however no way to remove reward tokens.

All major interactions by a user, `stakeGns`, `unstakeGns`, and `createUnlockSchedule` loops through all reward tokens to claim rewards and update latest debts. If the amount of reward tokens grows too much it is possible this could result in out of gas for the user.

Consider adding a max number of rewards tokens or a method of removing/disabling reward tokens.

### [L-02] Use of `transfer`/`transferFrom`

Not all ERC20 implementations are well behaved and revert on failure. Some simply return false and some might not return anything at all (USDT on mainnet).

Consider using OpenZeppelin `SafeERC20`s `safeTransfer` and `safeTransferFrom` to be as safe as possible for any atypical ERC20 tokens.
