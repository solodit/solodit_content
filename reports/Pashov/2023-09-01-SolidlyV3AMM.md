**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Medium Risk

### [M-01] Pools are incompatible with non-standard ERC20 tokens

**Severity**

**Impact:**
Medium, as it limits the possible pools in the app and with some special ones can result in loss of funds

**Likelihood:**
Medium, as some of the tokens are not uncommon, like `USDT`

**Description**

The Solidly V3 AMM code removes the balance checks before and after transfers and also the `TransferHelper` library, calling the `transfer` and `transferFrom` methods of ERC20 directly. This will be problematic with three different types of tokens:

1. The `USDT` and `BNB` tokens do not return a `bool` on `transfer`, meaning transactions that add liquidity for a pool of those tokens would directly revert, resulting in limitation of the possible pools in the application
2. The `ZRX` and `EURS` tokens do not revert on failure, but instead return `false` - the return value of the methods is not checked, so this can result in a loss of funds if either `transfer` or `transferFrom` fails
3. Some tokens have a transfer fee (like `STA`, `PAXG`) and for example `USDT` has a fee mechanism but it's currently switched off - such tokens will not work correctly with the pools and can lead to a loss of funds

**Recommendations**

While the docs should state that fee-on-transfer tokens are not supported, the application is incompatible with other tokens as well - either state this in the docs or consider using the `TransferHelper` library, also add the before and after transfer balance checks back.

**Discussion**

**pashov:** Tokens that do not return a `bool` on `transfer` or do not revert on failure are now correctly handled. Fee on transfer tokens are explicitly not supported.

### [M-02] Protocol owner has control over swap fees earned

**Severity**

**Impact:**
High, as it can result in a loss of yield for LPs

**Likelihood:**
Low, as it requires a compromised or malicious owner

**Description**

In the Solidly V3 AMM the swap fees can be collected by an account that is given the `feeCollector` role. The `SolidlyV3Factory` can give and revoke this role to/from accounts. It is expected that the initial role holder will be set to a `RewardsDistributor` contract (expected source code can be seen [here](https://github.com/SolidlyV3/v3-rewards/blob/dev/contracts/RewardsDistributor.sol)). In that contract, a permissioned `owner` will be able to set the Merkle root that shows who is able to claim fees (and how much).

There are a multiple possible options for the owner to misappropriate the swap fees from LPs:

1. The `SolidlyV3Factory` owner can set his own address or another one he controls to hold the `feeCollector` role, then just claim all fees
2. The `RewardsDistributor` owner can set a Merkle root for claimable rewards that holds only his address or another one he controls, then just claim all fees
3. The `RewardsDistributor` owner might never set a Merkle root, leaving the fees stuck in the application forever

**Recommendations**

While the team has communicated that all permissioned addresses will be behind a multi-sig and a Timelock contract, this is still an attack vector that can be exploited either from a compromised or a malicious owner. The Timelock has a 24h delay, which might be too low for users to claim their unclaimed yield. It's also possible that the Merkle root is set so they can't claim at all - you can think about a different mechanism for claiming rewards here.

**Discussion**

**Solidly Team** As per protocol design specification. For cases of malicious governance, our time-lock contract (OpenZeppelin) would warn users ahead of time. Acknowledged.

## Low Risk

### [L-01] Fee value is not validated in constructor

The pool fee value is checked in the `setFee` method and in `enableFeeAmount` as well with `require(fee <= 100000);`, but this check is omitted in the `SolidlyV3Pool` constructor, which means the initial fee can be set to a larger value. The check should be present in the constructor of `SolidlyV3Pool` as well.

**Discussion**

**Solidly Team** Acknowledged.