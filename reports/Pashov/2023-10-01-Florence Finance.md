**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Medium Risk

### [M-01] Burned tokens can be re-minted into the `totalSupply`

**Severity**

**Impact:**
Low, as it won't lead to funds loss but breaks a protocol invariant/assumption

**Likelihood:**
High, as it becomes a problem whenever someone burns their tokens

**Description**

The `FlorenceFinanceMediciToken` contract inherits from `ERC20CappedUpgradeable` and has a max supply limit of "1_000_000_000 \* 10 \*\* 18" token units. The issue is that the contract also inherits from the `ERC20BurnableUpgradeable` contract, which means that when a user calls the `burn` method, the `totalSupply` will be subtracted from, meaning if 10 tokens existed and are all burned, but then 10 new tokens are minted, now `totalSupply = 10` which is not the assumption that the protocol has, which is that the total supply of minted tokens can be maximum "1_000_000_000 \* 10 \*\* 18".

**Recommendations**

Remove the inheritance from `ERC20BurnableUpgradeable` in `FlorenceFinanceMediciToken` so that burning tokens with subtracting from `totalSupply` is not possible.

## Low Risk

### [L-01] Missing Arbitrum Sequencer availability check

The `getFundingTokenExchangeRate` method in `LoanVault` makes use of Chainlink price feeds by calling the `latestRoundData` method. While there are sufficient validations for the price feed answer, a check is missing for the L2 sequencer availability which has to be there since the protocol is moving from Ethereum to Arbitrum. In case the L2 Sequencer is unavailable the protocol will be operating with a stale price and also when the sequencer is back up then all of queued transactions will be executed on Arbitrum before new ones can be done. This can result in the `LoanVault::getFundingTokenExchangeRate` using a stale price, which means that a user might receive more or less shares from the `LoanVault` than he should have had. Still, since all of the funding requests are first approved off-chain, the probability of this happening is much lower. For a fix you can follow the [Chainlink docs](https://docs.chain.link/data-feeds/l2-sequencer-feeds) to add a check for sequencer availability.

### [L-02] Upgradeability best practices are not followed

In `LoanVault` the storage layout has been changed in a non-upgradeability safe way. Between commit `616e9d4ba18eef293dc76fb95144bd11fb29549b` and the current HEAD of the `audit` branch it seems like the `fundingFee` storage variable has changed its place and also the whitelisting storage variables have been removed. This is quite dangerous as if a potential upgrade is done on the Ethereum contracts it can mess up the storage layout, potentially bricking the contract. This is highly unlikely to happen though as the protocol is using the OpenZeppelin's upgrades plugin which protects from this. Still, upgradeability best practices should always be applied - do not reorder storage variables and instead of removing old ones just deprecate them.

### [L-03] Enforce initializer methods to be callable just once

The `setLoansOutstanding` method in `LoanVault` says in its NatSpec that "This is used for the initial deployment on Arbitrum." - the method should be callable only once. The same note is valid for the `initializeArbitrumBridging` method in token contracts in the codebase.