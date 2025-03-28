**Auditor**

[Alex The Entreprenerd](https://x.com/gallodasballo?lang=en)

# Findings

## Medium Risk

### [M-01] To Discuss - Liquidations should happen asap even the sequencer is in the grace period 

**Impact**

Liquidations are safer for the system

Liquidations could be extremely unfair to the user

I describe 2 scenarios that are extreme as a means to discuss both sides of the coin

**System Description**

At this time I believe that:
- Scroll has a centralized sequencer
- Scroll processes calls as FIFO (fair ordering from their POV)
- In the case of downtime, all txs will be dropped, and will have to be re-submitted
- Scroll uses EIP1559 to price it's blocks

Assuming this, the cost of a 15 minute DOS may be plausible (TODO)
Anything past 15 minutes will most likely cost millions of dollars, making it unlikely to be worth it unless Quill has a very high TVL


TO FINISH

**Unfair liquidation due to oracle not updating**

The scenario that waiting for the GracePeriod address is an unfair liquidation

Assuming a borrower had to be liquidated or they became liquidatable due to interesting during the Sequencer Downtime

However, per this logic we'd expect that it would be fair to liquidate them, this is due to their negligence and not any chain nor protocol downtime

**Lack of Oracle Update due to TX being dropped due to shutdown**

This is the wort case scenario from the above, if the Oracle update should have prevented the liquidation but didn't because it got dropped, then an argument could be made in favour of the Grace Period, as the Grace Period would grant them sufficient time to recapitalize

**Bad debt due to oracle updating**

This is the scenario we want to avoid

We had a user that was negligent, we couldn't liquidate them and the next oracle update will lock in bad debt to the protocol

This scenario is the tail risk the current code is taking, as you can have a liquidation that should happen, but couldn't and soon that liquidation will happen but the execution will be worse for the protocol

**Conclusion**

I believe either option has tradeoffs

My perspective is this won't happen with high likelihood

But if it will, it would be best to liquidate the user as to protect the protocol rather than risk locking in bad debt

I think the change is consistent in that way, and it can be flagged in the documentation

### [M-02] Must accrue `stabilityPoolYieldSplit` before changing it

**Impact**

Changing the yield can change the results from `calcPendingSPYield`

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/ActivePool.sol#L150-L153

```solidity
    function calcPendingSPYield() external view returns (uint256) {
        return calcPendingAggInterest() * stabilityPoolYieldSplit / DECIMAL_PRECISION;
    }

```

This means that the estimated value could change

Based on integrations this can cause a repricing of a wrapped SP Token (staked Quill)

**Mitigation**

Mitigation is straightforward, just `mintAggInterest` before updating the value

**Mitigation Status**

Fixed: https://github.com/subvisual/quill/pull/381

### [M-03] `CCR == SCR` can cause unintended shutdown when the protocol has only one Trove per Branch

**Impact**

The deployment scripts for Quill looks as follows:

https://github.com/subvisual/quill/blob/23e53123a16b12614d25bfb715e17dd41bcebbdd/contracts/scripts/DeployQuillLocal.s.sol#L38-L42

```solidity
        TroveManagerParams[] memory troveManagerParamsArray = new TroveManagerParams[](4);
        troveManagerParamsArray[0] = TroveManagerParams(150e16, 110e16, 110e16, 5e16, 10e16, 500e18, 72e16, _1pct / 2); // WETH
        troveManagerParamsArray[1] = TroveManagerParams(150e16, 120e16, 110e16, 5e16, 10e16, 500e18, 72e16, _1pct / 2); // wstETH
        troveManagerParamsArray[2] = TroveManagerParams(150e16, 120e16, 110e16, 5e16, 10e16, 500e18, 72e16, _1pct / 2); // weETH
        troveManagerParamsArray[3] = TroveManagerParams(150e16, 120e16, 110e16, 5e16, 10e16, 500e18, 72e16, _1pct / 2); // SCROLL
```

The MCR and SCR are matching, this means that for Branches that have one Trove, that trove could cause a shutdown of the branch as soon as the Trove is liquidatable

**Mitigation**

Either set the SCR to a lower amount

Or perform the following as part of your deployment:
- Open a trove on each branch
- Redeem it down to 0 debt
- Keep it open for each branch

This will make it so that no single borrower could cause a branch to shutdown


### [M-04] `QuillSimplePriceFeed` not having a `fetchRedemptionPrice` can open up to Oracle Drift Arbitrage via Redemptions

**Impact**

The pricing of a redemption in the `QuillSimplePriceFeed` is as follows:

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/Quill/PriceFeeds/QuillSimplePriceFeed.sol#L28-L31

```solidity
    function fetchRedemptionPrice() external returns (uint256, bool) {
        // Use same price for redemption as all other ops in WETH branch
        return fetchPrice(); /// @audit this may be underpriced due to Oracle Drift | You must raise the Redemption Fee to cover against it
    }
```

Meaning the price is the same as the one for borrowing

This is generally fine because the MCR for assets is expected to be above `110%` which makes oracle imprecisions somewhat negligible

However, for redemptions the base fee is typically 50 BPS

This seems to not be an issue currently as all Price Feeds I can see on: https://data.chain.link/feeds have a 50 BPS threshold

It's worth noting that the realized Deviation Threshold (the actual price change before the Price Feed finishes it's round) can still be greater than this, leading to some arbitrage in-spite of the settings looking correct

**Mitigation**

Ensure that no oracle or composite usage of oracle prossibly under-prices the asset by more than the Redemption Base Fee

As otherwise the system will naturally open itself up to Arbitrage

### [M-05] Risk of overborrowing by only using Price * Rate Feed

**Impact**

NOTE: I'm under the impression that the config for Quill is the following:

```solidity

IWETH weth = IWETH(0x5300000000000000000000000000000000000004);
IERC20Metadata wsteth = IERC20Metadata(0xf610A9dfB7C89644979b4A0f27063E9e7d7Cda32);
IERC20Metadata weeth = IERC20Metadata(0x01f0a31698C4d065659b9bdC21B3610292a1c506);
IERC20Metadata scroll = IERC20Metadata(0xd29687c813D741E2F938F4aC377128810E217b1b);

// https://data.chain.link/feeds/scroll/mainnet/eth-usd
address eth_usd_oracle = 0x6bF14CB0A831078629D993FDeBcB182b21A8774C;

// wstETH steth (exchange rate) | Rate arb??
// https://data.chain.link/feeds/scroll/mainnet/wsteth-steth%20exchangerate
address wsteth_steth_oracle = 0xE61Da4C909F7d86797a0D06Db63c34f76c9bCBDC;

// Exchange rate
// https://data.chain.link/feeds/scroll/mainnet/weeth-eeth-exchange-rate
address weeth_eth_oracle = 0x57bd9E614f542fB3d6FeF2B744f3B813f0cc1258;


// https://data.chain.link/feeds/scroll/mainnet/scr-usd
address scroll_usd_oracle = 0x26f6F7C468EE309115d19Aa2055db5A74F8cE7A5;

// NOTE: No page for this??
// https://scrollscan.com/address/0x45c2b8C204568A03Dc7A2E32B71D67Fe97F908A9#readContract
address chainlinkScrollSequencerUptimeFeed = 0x45c2b8C204568A03Dc7A2E32B71D67Fe97F908A9;
```

Meaning that all oracles use the ETH-USD feed instead of their Underlying * Rate Feed

This is in general a good price to protect against redemptions, as you're using ETH-USD * Rate - Meaning you're pricing the LST as it's underlying available ETH.

However, in the case of a Exploit, Bank Run or a temporary depeg, this price will allow overborrowing

This can be a good design decision against very quick flash crashes (that way customers don't get randomly liquidated), however, for sustained depegs, this decision is adding more risk to the protocol as it's not discounting the LST (which should have a discount vs ETH as it needs to include the various risk + exit queue delay)


https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/Quill/PriceFeeds/QuillCompositePriceFeed.sol#L56-L68

```solidity
    // An individual Pricefeed instance implements _fetchPricePrimary according to the data sources it uses. Returns:
    // - The price
    // - A bool indicating whether a new oracle failure or exchange rate failure was detected in the call
    function _fetchPricePrimary(bool /* _isRedemption */ ) internal returns (uint256, bool) {
        assert(priceSource == PriceSource.primary);
        (uint256 ethUsdPrice, bool ethUsdOracleDown) = _getOracleAnswer(ethUsdOracle);
        (uint256 lstEthPrice, bool lstEthOracleDown) = _getOracleAnswer(lstEthOracle);

        // If the ETH-USD feed is down, shut down and switch to the last good price seen by the system
        // since we need both ETH-USD and canonical for primary and fallback price calcs
        if (ethUsdOracleDown || lstEthOracleDown) {
            return (_shutDownAndSwitchToLastGoodPrice(address(ethUsdOracle.aggregator)), true);
        }
```

**Mitigation**

Ideally you should use:
- stETH / USD * Exchange Rate for Borrowing and ETH / USD * Exchange Rate for Redemptions
- weETH * Exchange Rate for Borrowing and ETH / USD * Exchange Rate for Redemptions

This will provide the best of both worlds in terms of pricing

In lack of that, you may be forced to manually change the price feed at times of liquidity crunch

weETH will not have this, so you may chose to raise the min redemption fee by either overestimating the oracle value, or by actually raising the redemption fee floor

### [M-06] Risky Footguns from Bold

**Impact**
These 3 lines can be VERY dangerous

https://github.com/GalloDaSballo/quill-review/blob/8d6e4c8ed0759cea1ff0376db9fd55db864cd7e8/contracts/src/Zappers/Modules/FlashLoans/BalancerFlashLoan.sol#L46-L52

```solidity
        // This will be used by the callback below no
        receiver = IFlashLoanReceiver(msg.sender);

        vault.flashLoan(this, tokens, amounts, userData);

        // Reset receiver
        receiver = IFlashLoanReceiver(address(0));
```

The reason why this code is safe for BOLD is becasue `vault` has a Reentrancy guard

In lack of that guard many projects can get exploited

**Theoretical Proof Of Code**

https://gist.github.com/GalloDaSballo/a4dd8c2b77a64d602983152d621f55c3

**Theoretical Mitigation**

- Validate AAVE initiator
- Consume the receiver

```solidity
    function receiveFlashLoan(
        IERC20[] calldata tokens,
        uint256[] calldata amounts,
        uint256[] calldata feeAmounts,
        bytes calldata userData
    ) external override {
        require(msg.sender == address(vault), "Caller is not Vault");
        require(address(receiver) != address(0), "Flash loan not properly initiated");

        // NOTE: Validate initiator (if available) (e.g. AAVE)
        // NOTE: Why not consume receiver?
        IFlashLoanReceiver public cachedReceiver = receiver;
        receiver = IFlashLoanReceiver(address(0));
```
## Low Risk

### [L-01] Long enough sequencer shutdown can create undefined shutdown risk

**Impact**

48 hours + no update = Shutdown

48 hours + update = No shutdown

Almost 48 hours + Block stuffing could be used to effectively have a 48 hours delay

All of these can either cause a shutdown or a no-op

**Mitigation**

I don't believe the issue can be mitigated, in case of that big a sequencer downtime, only Scroll will be able to decide what to do

If you can ensure that the oracles update will be processed before user operations then this can prevent branch shutdowns

### [L-02] Low Liquidity Branches will likely use the Redistribution due to 20% vs 5% premium

**Impact**

The difference in premium seems way too high, this can lead to scenarios in which, for low liquidity branches, whales remove bold from the stability pool as a means to trigger a redistribution

This is because redistributions are 4 times more profitable to them


https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/TroveManager.sol#L436-L455

```solidity
        if (_boldInStabPool > 0) {
            debtToOffset = LiquityMath._min(_entireTroveDebt, _boldInStabPool);
            collSPPortion = _collToLiquidate * debtToOffset / _entireTroveDebt;
            (collToSendToSP, collSurplus) =
                _getCollPenaltyAndSurplus(collSPPortion, debtToOffset, liquidationPenaltySP, _price);
        }

        // Redistribution
        debtToRedistribute = _entireTroveDebt - debtToOffset;
        if (debtToRedistribute > 0) {
            uint256 collRedistributionPortion = _collToLiquidate - collSPPortion;
            if (collRedistributionPortion > 0) {
                (collToRedistribute, collSurplus) = _getCollPenaltyAndSurplus(
                    collRedistributionPortion + collSurplus, // Coll surplus from offset can be eaten up by red. penalty
                    debtToRedistribute,
                    liquidationPenaltyRedistribution, // _penaltyRatio
                    _price
                );
            }
        }
```

**Mitigation**

I believe for some highly volatile assets it may be best to raise the premium up to 10% for Liquidations in the Stability Pool

This statement is unbacked as I don't have the time to model the SP + the Liquidity of the whole system

I believe this is not as big of an issue for liquity because they will have very high correlation assets

Whereas for you it may not be the case

### [L-03] Sequencer Sentinel Config can be updated to follow first principles

**Impact**

The sequencer sentinel has 2 types of checks:
- `_requireSequencerUpAndOverGracePeriod` - Safer check, ensures that prices are updated
- `_requireSequencerUp` - Less safe check, prices may not be updated

A check that is less safe could be used for operations that reduce risk to the system, such as repaying, closing and adding collateral

A check that is safer should be used for everything else


**Increase Risk - Should wait for Grace Period**

The following should be changed to use `_requireSequencerUpAndOverGracePeriod`

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/BorrowerOperations.sol#L520-L528

```solidity
    function adjustTroveInterestRate(
        uint256 _troveId,
        uint256 _newAnnualInterestRate,
        uint256 _upperHint,
        uint256 _lowerHint,
        uint256 _maxUpfrontFee
    ) external {
        _requireSequencerUp();
        _requireIsNotShutDown();
```

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/BorrowerOperations.sol#L929-L939

```solidity

    function setBatchManagerAnnualInterestRate(
        uint128 _newAnnualInterestRate,
        uint256 _upperHint,
        uint256 _lowerHint,
        uint256 _maxUpfrontFee
    ) external {
        _requireSequencerUp();
        _requireIsNotShutDown();
        _requireValidInterestBatchManager(msg.sender);
        _requireInterestRateInBatchManagerRange(msg.sender, _newAnnualInterestRate);
```

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/BorrowerOperations.sol#L1066-L1074

```solidity
    function removeFromBatch(
        uint256 _troveId,
        uint256 _newAnnualInterestRate,
        uint256 _upperHint,
        uint256 _lowerHint,
        uint256 _maxUpfrontFee
    ) public override {
        _requireSequencerUp();
        _requireIsNotShutDown();
```

**NOTE: MORE**

https://github.com/subvisual/quill/blob/23e53123a16b12614d25bfb715e17dd41bcebbdd/contracts/src/BorrowerOperations.sol#L996-L1005

```solidity
    function setInterestBatchManager(
        uint256 _troveId,
        address _newBatchManager,
        uint256 _upperHint,
        uint256 _lowerHint,
        uint256 _maxUpfrontFee
    ) public override {
        _requireSequencerUp();
        _requireIsNotShutDown();
        LocalVariables_setInterestBatchManager memory vars;
```

### [L-04] Uncapped Caller Premium raises the likelihood of a unprofitable liquidation to SP stakers by a marginal amount

**Impact**

This change makes liquidations require an additional 50BPS to be able to pay the caller incentive
Overall this is not a massive change
And the change to a capped 5% premium for SP liquidation, paired with a lowest MCR of 110 makes this pretty safe overall as a choice

**Mitigation**

No mitigation is required at this time, you should monitor collaterals and raise the MCR if the current one causes bad debt redistributions or losses to the SP too frequently

### [L-05] Governance Raising the CCR could be used to prevent people from borrowing by griefer

**Impact**

This is partially mitigated by
`_requireNoBorrowingUnlessNewTCRisAboveCCR(_troveChange.debtIncrease, newTCR);`

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/TroveManager.sol#L261-L280

```solidity
    function setNewBranchConfiguration(
        uint256 _scr,
        uint256 _mcr,
        uint256 _ccr,
        uint256 _newLiquidationPenaltySP,
        uint256 _newLiquidationPenaltyRedistribution
    ) external {
        _requireCallerIsCollateralRegistry();
        _requireValidConfig(_ccr, _mcr, _scr, _newLiquidationPenaltySP, _newLiquidationPenaltyRedistribution);

        SCR = _scr;
        MCR = _mcr;
        CCR = _ccr;
        liquidationPenaltySP = _newLiquidationPenaltySP;
        liquidationPenaltyRedistribution = _newLiquidationPenaltyRedistribution;

        emit BranchConfigurationUpdated(
            _scr, _mcr, _ccr, _newLiquidationPenaltySP, _newLiquidationPenaltyRedistribution
        );
    }
```

**Mitigation**

Perform multiple checks in your governance process to prevent griefing if possible

### [L-06] Zombie Trove Spam Risk when Oracle is Highly Predictable could be used to borrow at lower rate and never get redeemed

**Impact**

The logic for Redemptions looks as follows:

The trove data is loaded from the storage pointer

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/TroveManager.sol#L821-L829

```solidity
        SingleRedemptionValues memory singleRedemption;
        // Let’s check if there’s a pending zombie trove from previous redemption
        if (lastZombieTroveId != 0) {
            singleRedemption.troveId = lastZombieTroveId;
            singleRedemption.isZombieTrove = true;
        } else {
            singleRedemption.troveId = sortedTrovesCached.getLast();
        }
        address lastBatchUpdatedInterest = address(0);
```

Then we loop, and a specific edge case could happen:

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/TroveManager.sol#L843-L849

```solidity
            // Skip if ICR < 100%, to make sure that redemptions don’t decrease the CR of hit Troves
            if (getCurrentICR(singleRedemption.troveId, _price) < _100pct) {
                singleRedemption.troveId = nextUserToCheck;
                singleRedemption.isZombieTrove = false;
                continue;
            }

```

When the current trove is liquidatable, even if it's a zombie trove, it will be skipped

This is a necessity because technically the Trove cannot repay 100% of it's debt since it's underwater
And a liquidation is possible

However, this means that if we redeem another trove and make it a zombie, we are going to make the system forget about this one

The key pre-requisite is that each "forgotten" zombie trove must be underwater when this happens

Meaning that to pull this off reliably we'd need to find a highly volatile collateral feed and be able to perform these operations without getting the "Hidden Zombie" liquidated

This also requires the price to go down and then up "forever" hence it's low likelihood

However, introducing an oracle like Pyth, where deviations could be pushed each second could make this more likely

**Proof Of Concept**

**If you could use Pyth or smth, and have the Trove not liquidated you could create a ton of small troves and not pay the borrow rate**


Since the invariant is:
- Check zombie trove
- Skip if underwater

You could spam this to create a ton of zombie troves

The likelihood to pull this off on Mainnet is very low

The likelihood to pull this off with Pull or Manipulatable Oracle is a lot higher as you may be able to send different prices to the oracle


**Mitigation**

Ensure you monitor this behaviour and have liquidators available

Ultimately the R/R to pull this off is not great, so I doubt it will be attempted unless you introduce a Price Feed with Critical Vulnerabilities

### [L-07] BATCH DEBT FIXES - Rebase is economicall unfeasible

```python
SHARES * 1e16 // DENOM
9999999.0
SHARES * 1e14 // DENOM
99999.0
SHARES * 1e13 // DENOM
9999.0
SHARES * 1e9 // DENOM
1.0
SHARES * 1e8 // DENOM
0.0
SHARES * 1e10 // DENOM
10.0
SHARES * 2e8 // DENOM
0.0
SHARES * 9e8 // DENOM
0.0
SHARES * 1.9e8 // DENOM
0.0
SHARES * 1.9e9 // DENOM
1.0
0.06 * 1e9 / 1e18
6e-11
0.06 * 1e9 / 1e18 * 3463
2.0778e-07
0.06 * 1e9 / 1e18 * 3463 * 300_000
0.062334
0.06 * 1e9 / 1e18 * 3463 * 300_00
0.0062334
0.06 * 1e9 / 1e18 * 3463 * 3000
0.00062334
1e9 / 1e18 * 3463
3.463e-06
```

### [L-08] Incorrect Comment about blocks and adjustments 

**Impact**

Quill will reduce the min debt by a factor of 10

Bold had a potential risk when batch debt shares were rebased

The exploit was fixed and as far I was able to tell the system is safe from this exploit

Out of caution, as to maintain a similar risk profile as Liquity, it's best to 10x the `MIN_INTEREST_RATE_CHANGE_PERIOD` as to make it slower and more costly to rebase batches by spamming fees

Also the comment is incorrect since Scroll has faster (2/3 seconds) block times

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/Dependencies/Constants.sol#L33-L34

```solidity
uint128 constant MIN_INTEREST_RATE_CHANGE_PERIOD = 120 seconds; // prevents more than one adjustment per ~10 blocks /// @audit this is MAINNET

```

### [L-09] Scroll Bridge Analysis

**Deposit take about 20 Minutes**




**Withdrawals take about 2 hours**




So the risk from scroll is arguably not as high as for other chains such as OP 

The bridge risk may be perceived as higher due to lack of lindyness


### [L-10] `MAX_ANNUAL_INTEREST_RATE` should be customizable per branch and should be set higher due to exotic collaterals

**Q - Why is this not customizable?**

`MAX_ANNUAL_INTEREST_RATE` is left unchanged from Bold

`uint256 constant MAX_ANNUAL_INTEREST_RATE = _100pct;`

But Quill has collaterals that are more exotic, and also has the ability to update them

See this finding from my original review of Bold

**Lack of premium on redeeming higher interest troves can lead to all troves having the higher interest rate and still be redeemed - Cold Start Problem**

**Impact**

The following is a reasoned discussion around a possibly unsolved issue around CDP Design

In the context of Liquity V2, redemptions have the following aspect:
- Premium is paid to the owner that get's their troved redeemed
- Premium is dynamic like in V1, with the key difference being that Troves are now sorted by interest rate they pay

This creates a scenario, in the most extreme case, in which all Troves are paying the maximum borrow rate, but are still being redeemed against

**Intuition**

Any time levering up costs more than the base redemption fee (brings the price below it), the Trove will get redeemed against

The logic for redeeming is the fee paid

If the fee paid is not influenced by the rate paid by borrowers, then fundamentally there are still scenarios in which redemptions will close Troves in the most extreme scenarios

**Mitigation**

As discussed with the team, it may be necessary to charge a higher max borrow rate

Alternatively, redemptions should pay an additional premium to the Trove, based on the rate that is being paid by the borrower, the fundamental challenge with this is fairly pricing the rate of borrowing LUSD against the "defi risk free rate"

### [L-11] Inconsitent usage of `MIN_POSSIBLE_ANNUAL_INTEREST_RATE`

**Impact**

The variable `MIN_POSSIBLE_ANNUAL_INTEREST_RATE` is being used instead of `MIN_ANNUAL_INTEREST_RATE`


However some parts of the codebase still refer to `MIN_ANNUAL_INTEREST_RATE`


**Mitigation**

Clean up the codebase to consistently use `MIN_POSSIBLE_ANNUAL_INTEREST_RATE`


### [L-12] `ETH_GAS_COMPENSATION` changes may cause unprofitable liquidations during gas fees spikes - Suggested changes

**Impact**

The variable `ETH_GAS_COMPENSATION` in Quill was changed to:

```solidity
// Amount of ETH to be locked in gas pool on opening troves
uint256 constant ETH_GAS_COMPENSATION = 0.001 ether;
```

This can be insufficient for liquidations that happen at a high GWEI

**Data**

From these pages we can see that sometimes the gas can spike up to 3 GWEI on Rollups

https://scrollscan.com/chart/gasprice
https://blastscan.io/chart/gasprice
https://optimistic.etherscan.io/chart/gasprice


**Math**

This is taken off of the Tests provided in the repo

```
| batchLiquidateTroves                                                        | 30566           | 714168 | 648149 | 9548229 | 221     |
| liquidate                                                                   | 76475           | 468522 | 389065 | 727942  | 4796    |
```

```python
>>> ASSUMED_WORST_CASE_LIQUIDATION_COST = 350_000
>>> ASSUMED_WORST_CASE_LIQUIDATION_COST * 2e9 / 1e18
0.0007
>>> ASSUMED_WORST_CASE_LIQUIDATION_COST * 3e9 / 1e18
0.00105
```

```python
>>> ASSUMED_WORST_CASE_LIQUIDATION_COST = 800_000
>>> ASSUMED_WORST_CASE_LIQUIDATION_COST * 2e9 / 1e18
0.0016
>>> ASSUMED_WORST_CASE_LIQUIDATION_COST * 3e9 / 1e18
0.0024
>>> ASSUMED_WORST_CASE_LIQUIDATION_COST * 100e9 / 1e18
0.08
```


**Mitigation**

Perform more in depth benchmarks
And determine if the dynamic collateral premium is sufficient


Am thinking

ASSUMED_WORST_CASE_LIQUIDATION_COST = 350_000
ASSUMED_WORST_CASE_LIQUIDATION_COST * 10e9 / 1e18
0.0035

Which is $12 

Is probably a good point to shot at

Past that MEV actors should generally be able to optimize their gas cost
And 1/20 for min size also seems like it's not too burdensome to users

### [L-13] Missing DisableInitializers

**Impact**

The following contracts are now upgradeable

They do not `disableInitializers` on the logic

This is a code smell, I have yet to weaponize

And due to how the system works I doubt it will cause many issues


https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/HintHelpers.sol#L11-L20

```solidity
contract HintHelpers is QuillUUPSUpgradeable, IHintHelpers {
    string public constant NAME = "HintHelpers";

    ICollateralRegistry public collateralRegistry;

    function initialize(address _authority, ICollateralRegistry _collateralRegistry) public initializer {
        __QuillUUPSUpgradeable_init(_authority);
        collateralRegistry = _collateralRegistry;
    }

```

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/MultiTroveGetter.sol#L11-L17

```solidity
contract MultiTroveGetter is QuillUUPSUpgradeable, IMultiTroveGetter {
    ICollateralRegistry public collateralRegistry;

    function initialize(address _authority, ICollateralRegistry _collateralRegistry) public initializer {
        __QuillUUPSUpgradeable_init(_authority);
        collateralRegistry = _collateralRegistry;
    }
```

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/CollateralRegistry.sol#L42-L55

```solidity

    function initialize(address _authority, IBoldToken _boldToken, ISequencerSentinel _sequencerSentinel)
        public
        initializer
    {
        __QuillUUPSUpgradeable_init(_authority);
        lastFeeOperationTime = block.timestamp;
        boldToken = _boldToken;
        sequencerSentinel = _sequencerSentinel;

        // Initialize the baseRate state variable
        baseRate = INITIAL_BASE_RATE;
        emit BaseRateUpdated(INITIAL_BASE_RATE);
    }
```

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/BoldToken.sol#L43-L49

```solidity

    function initialize(address _authority) public initializer {
        __ERC20_init(_NAME, _SYMBOL);
        __ERC20Permit_init(_NAME);
        __QuillUUPSUpgradeable_init(_authority);
    }

```

### [L-14] Revert Case for Oracle being provided insufficient gas is inaccurate

**Impact**

The following finding has no impact unless Chainlinks proxy is changed to revert for OOG on purpose

Meaning that the CL team would need to:
- Replace their proxy
- Purposefully put a malicious one that reverts


In that scenario, the 1/64 gas check would result as incorrect

And due to it, any call to the price feed would always result in a revert, permanently DOSSing the system

**Proof Of Concept**

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";


contract MockCLFeed {
    enum Behaviour {
        NORMAL,
        REVERT,
        GASGRIEF
    }

    Behaviour behaviour = Behaviour.NORMAL;

    function setBehaviour(Behaviour b) external {
        behaviour = b;
    }

    function latestRoundData() external view returns (
            uint80 roundId, int256 answer, uint256, /* startedAt */ uint256 updatedAt, uint80 /* answeredInRound */
        ) {
            
            if(behaviour == Behaviour.REVERT) {
                revert("No out of gas");
            }

            if(behaviour == Behaviour.GASGRIEF) {
                // Grief them, burn all gas
                uint256 i;
                while (true) {
                    i++;
                }
            }


            answer = 123;   
        }
    

}

contract PriceLibTester is Test {

    function getCurrentChainlinkResponse(MockCLFeed _aggregator)
        external
        view
        returns (int256 price)
    {
        uint256 gasBefore = gasleft();

        // Try to get latest price data:
        try _aggregator.latestRoundData() returns (
            uint80 roundId, int256 answer, uint256, /* startedAt */ uint256 updatedAt, uint80 /* answeredInRound */
        ) {


            return answer;
        } catch {
            // NOTE: The check is ignoring additional costs that come from processing the error + the call
            // So even thought the check is directionally right
            // You would need to give a few thousands gas of leniency to the check to actually be safe
            
            // Require that enough gas was provided to prevent an OOG revert in the call to Chainlink
            // causing a shutdown. Instead, just revert. Slightly conservative, as it includes gas used
            // in the check itself.
            console.log("gasleft()", gasleft());
            console.log("gasBefore / 64", gasBefore / 64);
            if (gasleft() + 2000 <= gasBefore / 64) revert("InsufficientGasForExternalCall()");

            // If call to Chainlink aggregator reverts, return a zero response with success = false
            return -1;
        }
    }

    // forge test --match-test test_normal_and_revert_case -vv
    function test_normal_and_revert_case() public {
        MockCLFeed feed = new MockCLFeed();

        
        feed.setBehaviour(MockCLFeed.Behaviour.NORMAL);
        this.getCurrentChainlinkResponse(feed);
        console.log("Base case ok");

        
        feed.setBehaviour(MockCLFeed.Behaviour.REVERT);
        this.getCurrentChainlinkResponse(feed);
        console.log("Revert case ok");

        // NOTE: This fails
        feed.setBehaviour(MockCLFeed.Behaviour.GASGRIEF);
        this.getCurrentChainlinkResponse(feed);
        console.log("Gas Grief Case ok");


    }

    
}
```

**Mitigation**

Changing the code to have an additional small buffer:
```solidity
       if (gasleft() + 2000 <= gasBefore / 64) revert("InsufficientGasForExternalCall()");
```

Would ensure that the cost of processing the call and the cost of handling the error are accounted for

However, if OOG Dosses are a real concern, you should cap the gas given to CL to a certain amount (e.g. 1MLN Gas) 




### [L-15] Liquity Oracle Gas Math could theortically fail if the CL Price Feed Gas Griefs

**Impact**

The following finding has no impact unless Chainlinks proxy is changed to revert for OOG on purpose

Meaning that the CL team would need to:
- Replace their proxy
- Purposefully put a malicious one that reverts


In that scenario, the 1/64 gas check would result as incorrect

And due to it, any call to the price feed would always result in a revert, permanently DOSSing the system

**Proof Of Concept**

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";


contract MockCLFeed {
    enum Behaviour {
        NORMAL,
        REVERT,
        GASGRIEF
    }

    Behaviour behaviour = Behaviour.NORMAL;

    function setBehaviour(Behaviour b) external {
        behaviour = b;
    }

    function latestRoundData() external view returns (
            uint80 roundId, int256 answer, uint256, /* startedAt */ uint256 updatedAt, uint80 /* answeredInRound */
        ) {
            
            if(behaviour == Behaviour.REVERT) {
                revert("No out of gas");
            }

            if(behaviour == Behaviour.GASGRIEF) {
                // Grief them, burn all gas
                uint256 i;
                while (true) {
                    i++;
                }
            }


            answer = 123;   
        }
    

}

contract PriceLibTester is Test {

    function getCurrentChainlinkResponse(MockCLFeed _aggregator)
        external
        view
        returns (int256 price)
    {
        uint256 gasBefore = gasleft();

        // Try to get latest price data:
        try _aggregator.latestRoundData() returns (
            uint80 roundId, int256 answer, uint256, /* startedAt */ uint256 updatedAt, uint80 /* answeredInRound */
        ) {


            return answer;
        } catch {
            // NOTE: The check is ignoring additional costs that come from processing the error + the call
            // So even thought the check is directionally right
            // You would need to give a few thousands gas of leniency to the check to actually be safe
            
            // Require that enough gas was provided to prevent an OOG revert in the call to Chainlink
            // causing a shutdown. Instead, just revert. Slightly conservative, as it includes gas used
            // in the check itself.
            console.log("gasleft()", gasleft());
            console.log("gasBefore / 64", gasBefore / 64);
            if (gasleft() + 2000 <= gasBefore / 64) revert("InsufficientGasForExternalCall()");

            // If call to Chainlink aggregator reverts, return a zero response with success = false
            return -1;
        }
    }

    // forge test --match-test test_normal_and_revert_case -vv
    function test_normal_and_revert_case() public {
        MockCLFeed feed = new MockCLFeed();

        
        feed.setBehaviour(MockCLFeed.Behaviour.NORMAL);
        this.getCurrentChainlinkResponse(feed);
        console.log("Base case ok");

        
        feed.setBehaviour(MockCLFeed.Behaviour.REVERT);
        this.getCurrentChainlinkResponse(feed);
        console.log("Revert case ok");

        // NOTE: This fails
        feed.setBehaviour(MockCLFeed.Behaviour.GASGRIEF);
        this.getCurrentChainlinkResponse(feed);
        console.log("Gas Grief Case ok");


    }

    
}
```

**Mitigation**

Changing the code to have an additional small buffer:
```solidity
       if (gasleft() + 2000 <= gasBefore / 64) revert("InsufficientGasForExternalCall()");
```

Would ensure that the cost of processing the call and the cost of handling the error are accounted for

However, if OOG Dosses are a real concern, you should cap the gas given to CL to a certain amount (e.g. 1MLN Gas) 




### [L-16] Addresses

All addresses sourced from:
https://github.com/subvisual/quill/blob/a7c93057ce2de13a27a774f22bddd63878c9fe18/contracts/src/scripts/DeployQuillSCrollMainnet.s.sol#L64-L68

```solidity
    struct TroveManagerParams {
        uint256 CCR;
        uint256 MCR;
        uint256 SCR;
        uint256 LIQUIDATION_PENALTY_SP;
        uint256 LIQUIDATION_PENALTY_REDISTRIBUTION;
        uint256 MIN_DEBT;
        uint256 SP_YIELD_SPLIT;
        uint256 minAnnualInterestRate;
    }
```

```solidity
        TroveManagerParams[] memory troveManagerParamsArray = new TroveManagerParams[](4);
        troveManagerParamsArray[0] = TroveManagerParams(140e16, 110e16, 110e16, 5e16, 10e16, 500e18, 75e16, 6 * _1pct); // WETH
        troveManagerParamsArray[1] = TroveManagerParams(160e16, 120e16, 120e16, 5e16, 10e16, 500e18, 75e16, 6 * _1pct); // wstETH
        troveManagerParamsArray[2] = TroveManagerParams(160e16, 120e16, 120e16, 5e16, 10e16, 500e18, 75e16, 6 * _1pct); // weETH
        troveManagerParamsArray[3] = TroveManagerParams(160e16, 120e16, 120e16, 5e16, 30e16, 500e18, 75e16, 7 * _1pct); // SCROLL
```
--------

20% MCR for wstETH | weETH and SCROLL

120 MCR for wstETH looks crazy - What a loss of efficiency for close to zero advantage

weETH -> Need to research further

--------


120% on Scroll looks somewhat scary on the worst days
25% price change in about 1 day
So arguably there can be a few days where the protocol get's rekt by this

---------

```solidity

    IWETH weth = IWETH(0x5300000000000000000000000000000000000004);
    IERC20Metadata wsteth = IERC20Metadata(0xf610A9dfB7C89644979b4A0f27063E9e7d7Cda32);
    IERC20Metadata weeth = IERC20Metadata(0x01f0a31698C4d065659b9bdC21B3610292a1c506);
    IERC20Metadata scroll = IERC20Metadata(0xd29687c813D741E2F938F4aC377128810E217b1b);

    // https://data.chain.link/feeds/scroll/mainnet/eth-usd
    address eth_usd_oracle = 0x6bF14CB0A831078629D993FDeBcB182b21A8774C;

   // wstETH steth (exchange rate) | Rate arb??
// https://data.chain.link/feeds/scroll/mainnet/wsteth-steth%20exchangerate
    address wsteth_steth_oracle = 0xE61Da4C909F7d86797a0D06Db63c34f76c9bCBDC;

// Exchange rate
// https://data.chain.link/feeds/scroll/mainnet/weeth-eeth-exchange-rate
    address weeth_eth_oracle = 0x57bd9E614f542fB3d6FeF2B744f3B813f0cc1258;


// https://data.chain.link/feeds/scroll/mainnet/scr-usd
    address scroll_usd_oracle = 0x26f6F7C468EE309115d19Aa2055db5A74F8cE7A5;

// NOTE: No page for this??
// https://scrollscan.com/address/0x45c2b8C204568A03Dc7A2E32B71D67Fe97F908A9#readContract
    address chainlinkScrollSequencerUptimeFeed = 0x45c2b8C204568A03Dc7A2E32B71D67Fe97F908A9;


// TODO: Check uptime?
    uint256 eth_usd_stalenessThreshold = _48_HOURS;
    uint256 wsteth_steth_stalenessThreshold = _48_HOURS;
    uint256 weeth_eth_stalenessThreshold = _48_HOURS;
    uint256 scroll_usd_stalenessThreshold = _48_HOURS;

```

**Sequencer Feed**

4/9 Multi can DOS it

https://scrollscan.com/address/0xDce20610907bf67D97d0ECcF31C50eaec73bC034#readProxyContract

**Tokens on Scroll**

wstETH
https://scrollscan.com/address/0xf610a9dfb7c89644979b4a0f27063e9e7d7cda32


WETH
https://scrollscan.com/token/0x5300000000000000000000000000000000000004

SCR
https://scrollscan.com/token/0xd29687c813d741e2f938f4ac377128810e217b1b

weETH
https://scrollscan.com/token/0x01f0a31698c4d065659b9bdc21b3610292a1c506

---
## Gas Optimizations

### [G-01] Gas Optimizations

**5k+ fetch price only if not shutdown**

https://github.com/subvisual/quill/blob/23e53123a16b12614d25bfb715e17dd41bcebbdd/contracts/src/BorrowerOperations.sol#L736-L739

```solidity
        (uint256 price,) = priceFeed.fetchPrice();
        if (!hasBeenShutDown) {
            _requireNewTCRisAboveCCR(_getNewTCRFromTroveChange(troveChange, price));
        }
```

You could fetch the price only if `!hasBeenShutDown`

**2.1k gas Can be made immutable**

I cannot find a setter, so you can save gas by making this immutable

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/TroveManager.sol#L225-L226

```solidity
        sequencerSentinel = _addressesRegistry.sequencerSentinel();

```

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/BorrowerOperations.sol#L31-L32

```solidity
    ISequencerSentinel internal sequencerSentinel;

```

**200+ Unnecessary Double Call**

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/CollateralRegistry.sol#L88-L114

```solidity

        // Gather and accumulate unbacked portions
        for (uint256 index = 0; index < totals.numCollaterals; index++) {
            ITroveManager troveManager = getTroveManager(index);
            (uint256 unbackedPortion, uint256 price, bool redeemable) =
                troveManager.getUnbackedPortionPriceAndRedeemability();
            prices[index] = price;
            if (redeemable) {
                totals.unbacked += unbackedPortion;
                unbackedPortions[index] = unbackedPortion;
            }
        }

        // There’s an unlikely scenario where all the normally redeemable branches (i.e. having TCR > SCR) have 0 unbacked
        // In that case, we redeem proportinally to branch size
        if (totals.unbacked == 0) {
            unbackedPortions = new uint256[](totals.numCollaterals);
            for (uint256 index = 0; index < totals.numCollaterals; index++) {
                ITroveManager troveManager = getTroveManager(index);
                (,, bool redeemable) = troveManager.getUnbackedPortionPriceAndRedeemability();
                if (redeemable) {
                    uint256 unbackedPortion = troveManager.getEntireSystemDebt();
                    totals.unbacked += unbackedPortion;
                    unbackedPortions[index] = unbackedPortion;
                }
            }
        }
```

## Informational

### [I-01] Analysis: Suggested Next Steps

**Executive Summary**

No major smart contract risk was identified by my review

Extensive integration risks can be introduced via zappers

And governance has the ability to cause many issues due to misconfiguration, lack of borrow limits and race conditions around config changes

**Misconfig Risk**

An incorrect relation between MCR, CCR and SCR could cause branch shutdowns 

MCR also may need to be adjusted frequently for tokens that have a high volatility

From my research SCR cannot be considered a "real token" as of now as it tends to trade at 1-1 with ETH, while having a substantially smaller market cap, leading me to believe the price is there due to lack of liquidity and trading more so than because it's properly valued

**Lack of Borrow Limits**

Could be "Mango'd", as discussed for SCR, in lack of borrow limits, low collateral assets could be used to over borrow and lock in massive losses to the protocol

It's crucial that conservative caps are established and constantly monitored and updated

**Race Condition Risks**

Griefing from moving the TCR below the CCR

Self-liquidating if MCR is changed or liquidation premiums are changed

Skipping limits and caps by performing these operations around governance

Shutdown can cause immense MEV opportunities

It's crucial that the execution of these functions is done by Governance, a permissioneless execute massively increases risks to all users

**Private Key Risks**

Shutdown into urgent redemptions can be used steal system value (1% premium + Oracle Drift)

**Suggested Next Steps**

Given the updates, changes, economic considerations and need for borrow limits

As well as your plan to add Borrow Limits and Zaps

I believe the best next step is to go through the mitigation review, set everything up and deploy it

Then go through a Security Contest, with:
- Smart Contracts
- Governance Contracts
- Configuration
- Future Configurations

As part of the scope

This will help you secure not just the Software Architecture, but also the deployed bytecode, which massively reduces operational risks


**Suggested Governance Setup**

Multisig -> Timelock -> Changes

Multisig MUST have `onlyOwner` guard:
https://github.com/safe-global/safe-smart-account/blob/main/contracts/examples/guards/OnlyOwnersGuard.sol


Timelock needs to have cancellor setup
Cancellor should be faster than the Multisig as vetoing is generally a safer operation than executing (can be owners of the multi, or another set of signers (even EOAs)

Shutdown is possibly the one function that would require being fast, you may want to enable a Guardian contract that a set of signers can call, allowing them to perform certain operations instantaneously

It's worth noting that shutdown will lead to `urgentRedemptions` causing a high amount of value to be lost to MEV actors, you should ideally plan around this, by performing the redemptions yourself and passing on the collateral to the original depositors, this is non-trivial and requires planning

**Governance Next steps**

Chart out all functions
Chart out the speed that each function needs
Decide if the Multi / Timelock should only be able to perform it
Decide which operations should be even faster


**Economic Suggested Next Steps**

Due to the introduction of borrow limits, as well as the low liquidity environment currently offered by scroll, no "long lasting" economic decision can be fully done

You'll more likely be forced to monitor the chain for available liquidity (which implies safe liquidations that can be performed) and over time will be able to alter caps

The following methodology could be applied to determine if caps are safe enough:
- Take the average liquidity available on chain
- Compute the amount of assets that could be sold before a PREMIUM change in price (where premium is 5% in your code, but that may change)
- These are the amount of assets that can be sold during liquidations while them being profitable (which ensures they will happen)
- Any amount above this may make liquidation unprofitable (or require more risk, making them less likely)
- Cap the borrow limits to these amounts, and monitor their change over time

### [I-02] Suggested Changes - Change stETH to 110 MCR

**Disclosures**

I do not hold any LIDO, I do hold LQTY, stETH and wstETH

**Executive Summary**

stETH doesn't need to have a 120 MCR, I recommend 110, 112 if you want to maintain a relative risk profile against ETH

As it stands even the MCR for ETH is very conservative, so 110 for stETH seems fine as well

120 MCR implies that stETH is MASSIVELY riskier than ETH

While it's factually true that stETH is riskier, Quill has access to 2 tools

1) Debt Limits, which limit the maximum exposure and risk to bad debt
2) Upgradeability, and shutdown

Due to this, Quill could change the risk parameters at a later time

**stETH Tail Event**

stETH presents 3 key risks that must be addressed in a thorough risk analysis

1) Slashing
2) Upgrade Risk and Perceived Governance risk
3) Exit Queue Liquidity Risk


**Slashing**

Slashing is arguably a real risk, but in analyzing slashing, we have to account for the relative amount lost vs the total amount staked and yielding

Slashing risk is a real risk, but in most cases the impact is negligible, definitely not "volatility generating"

The real risk around slashing is if something massive happen, such as the "GETH BUG" causing a massive amount of stake to be lost

This is a real risk, however this would cause similar issues to ETH itself, meaning this black swan can kill Quill in both cases

And I'm not convinced it can be mitigated

**Upgrade Risk and Perceived Governance Risk**

Upgrade risk is a real risk, which can put 100% of funds at risk

Upgrades from lido are behind a timelock

The governance could have failed many times already

Meaning that a future upgrade could change the risk profile of stETH

But as of today this upgrade risk is minimized

In the event of an upgrade, which would be behind a timelock, the branch could be shutdown to cause a 1% haircut to all borrowers

Which is a lot lower than any real risk

The perceived risks of a governance takeover, upgrade, or general fear are instead worth entertaining

These would cause stETH to depreciate, the question is by how much

As long as these changes don't cause stETH to lose 10% of it's value in a matter of minutes, then the system could handle them

**Exit Queue Risk - 1 / 1 stETH Pricing**

Illiquidity can be a huge issue, and liquidation cascades are very common during the Bull Market

Many risk advisors have suggested hardcoding the price of stETH to 1, this can be fine for short burst

From working with Liquity we've discussed this as a tail risk mostly around redemptions, more so than around liquidations

That's because Liquity has a fairly high MCR (110%) for ETH, meaning Collateral has to massively depreciate before it would cause bad debt to be locked in by the system

On the long term, 1/1 par pricing is ignoring the necessary discount that comes from stETH having a redemption queue

In most times, the withdrawal queue takes 7 days, this is a real cause for discount, but fairly limited

If we naively take 15% (currently the degen rate of lending), we can infer the following:

15/365*7 = 0.287671232877

stETH should naturally trade below parity by about 30 BPS

However it doesn't because of liquity and incentives that make it so that there's a fairly deep buffer to conver from stETH back to ETH

As long as that buffer is there, and that buffer is mostly available to Quill, then stETH should be priced at parity with ETH

**Validator Queue Exit Math**

Assuming 100% of stake wanted to exit, with 1 MLN Validators it would take around 277 days (linear interpolation, which is inexact)

If we apply the same idea around delay = discount, this should cause a 15% to 20% discount on stETH

This could lead to wanting a 120 MCR, however this is the absolute worst case which could happen as a black swan, but shouldn't happen for a prolonged amount of time

In am more realistic scenario we'd expect the discount to vary as more people want to exit, which means that stETH shouldn't reprice by more than 10% within an hour

**stETH day to day pricing**

On the day to day, stETH should be priced based on it's liquidity and based on the amount that may need to be liquidated on a % swing

These are the same principles for ETH

**Borrow Caps**

By capping the total amount of borrows against stETH to the amount that is liquidatable you are limiting the maximum exposure of the protocol by a lot

**Methodology**

- Track all borrows on your chain (including other protocols)
- Track the % of stETH that would need to be liquidated if X% price change would happen (generally 5% to 20%)
- Talk to liquidators and investors to have an additional liquidity pool
- Cap the borrow caps at the $ equivalent of this
- Monitor and change this amount based on what happens


**Price Analysis**

Below I provided all data and analyses of it

Even if you check all prices from back in 2021, the maximum swing that happened in an hour is contained at about 2%

The maximum swing in a day is less than 5%

Meaning that stETH has had volatility, but this is fairly limited against ETH

If we look at the last year, which has had incredible volatility for ETH

The relative price changes for stETH have been very contained, between 1.4 and 2% in an hour

These can lead to a best case of setting the MCR to 110 (same as ETH as roughly the same risk parameters, given a borrow cap of available liquidity)

And a more conservative MCR of 112 (the extra 2%), which should hold under most circumnstances, barring extreme volatility due to a perceived governance risk. For those scenarios you may want to shutdown the branch temporarily if necessary

**All Data Info**

Data was scraped from CL, with a Recon Pro tool

All data is public 

NOTE: Not all prices scraped where the exact prices from the aggregator, this is due to the methodology

**All Prices**

```ts
pricesWithinTimePeriod 1265
mean 994826037804453500
STANDARD DEVIATION 9789739730572964
as percent of mean 0.9840654907041415
getHighestAndLowestPrice 1011099584285630300 935019330000000000


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1652457558, price: 956120080000000000 },
  { date: 1651872078, price: 998995978974146300 }
]
deviation 4.291899054305995
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1652895059, price: 984201283138867600 },
  { date: 1652457558, price: 956120080000000000 }
]
deviation 2.936995438780826


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1655197624, price: 941991810710835000 },
  { date: 1655154105, price: 962272040061608300 }
]
deviation 2.1075359676329115
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1655154105, price: 962272040061608300 },
  { date: 1655109764, price: 940974924073828400 }
]
deviation 2.2633032446366244


interval 14400

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1733800235, price: 987343063643981400 },
  { date: 1733791463, price: 1005061173410059600 }
]
deviation 1.762888691238828
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1652462990, price: 976665000054923600 },
  { date: 1652457558, price: 956120080000000000 }
]
deviation 2.1487803137576242


interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1725412283, price: 982523475032176500 },
  { date: 1725412223, price: 996572661455605200 }
]
deviation 1.4097503340005673
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1725414119, price: 1000884598543477800 },
  { date: 1725412283, price: 982523475032176500 }
]
deviation 1.8687719914987213
```

**Last Year**

Price changes
```ts
pricesWithinTimePeriod 422
mean 998928145333159300
STANDARD DEVIATION 2765111607703231.5
as percent of mean 0.2768078585653446
getHighestAndLowestPrice 1011099584285630300 982523475032176500


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1733800235, price: 987343063643981400 },
  { date: 1733791463, price: 1005061173410059600 }
]
deviation 1.762888691238828
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1723157987, price: 1011099584285630300 },
  { date: 1722824399, price: 984642681457643900 }
]
deviation 2.6869547020671702


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1733800235, price: 987343063643981400 },
  { date: 1733791463, price: 1005061173410059600 }
]
deviation 1.762888691238828
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1725414119, price: 1000884598543477800 },
  { date: 1725412283, price: 982523475032176500 }
]
deviation 1.8687719914987213


interval 14400

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1733800235, price: 987343063643981400 },
  { date: 1733791463, price: 1005061173410059600 }
]
deviation 1.762888691238828
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1725414119, price: 1000884598543477800 },
  { date: 1725412283, price: 982523475032176500 }
]
deviation 1.8687719914987213


interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1725412283, price: 982523475032176500 },
  { date: 1725412223, price: 996572661455605200 }
]
deviation 1.4097503340005673
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1725414119, price: 1000884598543477800 },
  { date: 1725412283, price: 982523475032176500 }
]
deviation 1.8687719914987213
```

### [I-03] Quill necessitates having borrow caps

**Impact**

As discussed here #20 

Many assets on Scroll are extremely illiquid

This means that Quill cannot be fully trustless as some branches pose counter-party risk to all users

Because of this, it's necessary you add borrow caps as to limit the damage that borrowers can cause by borrowing with illiquid assets

**Mitigation**

Introduce borrow caps

You can set these up in the `BoldToken` 

### [I-04] Economic Analysis off of CL Price Feeds

**Methodology**

I fetch all prices from CL

I run a script to find the biggest deltas given a period of 1 week or 1 day

I use these periods to identify high volatility periods

I determine the % change as follows:

```js
- pointsOfBiggestNegativeSwing

(Highest - Lowest) / Highest * 100
= -%

- pointsOfBiggestPositiveSwing

(Highest - Lowest) / Lowest * 100
= +%
```

Because of the uniqueness of the chain I then looked at the liquidity sources for various assets

Overall the key issue for scroll is the lack of liquidity

This makes most assets extremely risk to Quill

Here are some rational limitations given the current state of scroll:

- 25k SCR 

SCR is illiquid and even this amount could cause bad debt to the protocol, more so than liquidation risk and premium caps are necessary
Generally speaking a 150% CR seems fine, but the reality is that SCR is subject to "exit scam" risk, a single seller will always be able to cause bad debt to your project, so I believe the issue cannot be solved via CR, but rather by limiting it or ideally removing it until it's better distributed.

- 300 wstETH

Value above this will incurr too much slippage, until you've gone through some liquidations, I believe that this should be the max cap for wstETH, this can be raised if you find someone that is willing to take the delay risk tied with bridging out wstETH back onto mainnet


- 65 weETH 

After this value slippage becomes too big of a factor

- 400 WETH (Limited by Liquidity)

Similarly, however based on the amount of QUILL and based on growth, I'd expect WETH to be the token that could most likely grow


**Initial Takeaways**

It seems clear that Scroll is behaving as ETH Beta

When reviewing the prices and looking at them something felt off

I'm not realizing that Scroll has only 14% Circulating Supply

I believe the economic analysis then needs to be conducted with the main modelling assuming a certain amount of actors dumping
Because I believe SCR is not trading at it's fair market price, but rather as a pegged ETH Beta, due to supply games

**Scroll Illiquidity**

As said above, I'm very surprised by the behaviour of SCR, this leads me to believe that the price for the token is "forced", it's there because of supply and perhaps incentive games, but I don't see how this price is real in any way

A quick tour around Exchanges will illustrate my point thoroughly


**Concentrated Ownership**

The ownership of Scroll is alarming

The SCR token is distributed over multiple gnosis safes

However, each safe belongs to the same group of signers and has the same threshold

There is factually no advantage, nor protection being setup on these safes afaict

https://scrollscan.com/address/0x212499e4e77484e565e1965ea220d30b1c469233#readProxyContract

```js
[ getOwners method Response ]
[[0x558A9596940AD909C9e6695Ecd1864b27dE0138f]
[0xba5D4c4475992cc50ae9Cb561a216840Ece68A76]
[0x108493124adf60F401E051e6A05043d8967bff6f]
[0x33fCB6845F6Cf2Da11fA2D68cf0a9F04C8A69be6]
[0x1Da431d2D5ECA4Df735F69fB5ea10c8E630b8f50]]
```

getThreshold
3

https://scrollscan.com/address/0xee198f4a91e5b05022dc90535729b2545d3b03df#readProxyContract
https://scrollscan.com/address/0x206367ebd1fb54f4f33818821feab16f606eebb7#readProxyContract
https://scrollscan.com/address/0x4cb06982dd097633426cf32038d9f1182a9ada0c#readProxyContract
https://scrollscan.com/address/0xff120e015777e9aa9f1417a4009a65d2eda78c13#readProxyContract
https://scrollscan.com/address/0x86e3730739cf5326eeba4cb8a2bf57dd91a2e455#readProxyContract


**Sword of Damocles**

https://scrollscan.com/token/0xd29687c813d741e2f938f4ac377128810e217b1b?a=0x687b50a70d33d71f9a82dd330b8c091e4d772508

This is just an EOA with 72 MLN tokens

Crunching some basic numbers:
>>> 1e9 - 242e6 - 200e6 - 180e6 - 105e6 - 96e6 - 83e6 
94000000.0
>>> 94000000 / 1e6
94.0
>>> 76/94 * 100
80.85106382978722
>>> 


$94MLN is the circulating market cap
$76MLN are in one EOA





**Off of CL I'm looking for the largest swings**

**0x6bF14CB0A831078629D993FDeBcB182b21A8774C - ETH / USD**

```js
pricesWithinTimePeriod 12520
mean 296623020987.87286
STANDARD DEVIATION 55803492279.62583
as percent of mean 18.812933700755245
getHighestAndLowestPrice 409921000000 174995423003


interval 1 Week

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1722820388, price: 217303390000 },
  { date: 1722247086, price: 339447816000 }
]
deviation 35.98327054783584
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1731401978, price: 342694000000 },
  { date: 1730836693, price: 240278000000 }
]
deviation 42.623960578996


interval One Day

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1722820388, price: 217303390000 },
  { date: 1722738242, price: 291707960000 }
]
deviation 25.506527144476966
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1716292174, price: 381035000000 },
  { date: 1716205947, price: 309105437400 }
]
deviation 23.27023529738788


interval 4 Hours

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1722820388, price: 217303390000 },
  { date: 1722809531, price: 275053760000 }
]
deviation 20.99603001246011
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1716243802, price: 366926000000 },
  { date: 1716232916, price: 317582000000 }
]
deviation 15.537404512850225


interval One Hour

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1722820388, price: 217303390000 },
  { date: 1722817067, price: 267641000000 }
]
deviation 18.80788444221924
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1716235256, price: 346141010000 },
  { date: 1716232916, price: 317582000000 }
]
deviation 8.992641270600979
```

**23% on a day**
The real risk is liquidity crunch making liquidations impossible

**18% in an hour**

If there was going to be a day in which Quill was going to lock-in bad debt, it was going to be that day

It's worth noting that Liquity had no bad debt even during that day

This is a key advantage of the Stability Pool, even though it could technically result in risky behaviour for the "economic system", the system is a lot more resilient because of it

Ultimately all SP stakers are taking the liquidation at face value which is a huge advantage to the system

However, this swing shows how even ETH may need a borrow cap as otherwise it could cause damage to the system



**0x26f6F7C468EE309115d19Aa2055db5A74F8cE7A5 - SCR USD**

```js
pricesWithinTimePeriod 8280
mean 87985535.4692029
STANDARD DEVIATION 20027569.770544685
as percent of mean 22.7623434508219
getHighestAndLowestPrice 144111474 53601588


interval 1 Week

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1734574388, price: 95669702 },
  { date: 1734071585, price: 144111474 }
]
deviation 33.6140979308837
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1734071585, price: 144111474 },
  { date: 1733778351, price: 86669799 }
]
deviation 66.27646038500677


interval One Day

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1734129342, price: 118921555 },
  { date: 1734071585, price: 144111474 }
]
deviation 17.479468012380472
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1733808664, price: 116083620 },
  { date: 1733778351, price: 86669799 }
]
deviation 33.937797640444515


interval 4 Hours

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1734078367, price: 122305169 },
  { date: 1734071585, price: 144111474 }
]
deviation 15.131553647144017
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1733808664, price: 116083620 },
  { date: 1733801040, price: 90725025 }
]
deviation 27.95104768502406


interval One Hour

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1734074465, price: 129019596 },
  { date: 1734071585, price: 144111474 }
]
deviation 10.47236391461793
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1733808664, price: 116083620 },
  { date: 1733805125, price: 99448009 }
]
deviation 16.727947766153868
```



**0x45c2b8C204568A03Dc7A2E32B71D67Fe97F908A9**

SEQUENCER

JUST GET BIGGEST UPTIME AND DOWNTIME

The transition from round 18446744073709551721 (value 1) to round 18446744073709551720 (value 0), with times 1734366945 and 1734377886 respectively. We calculate the difference between the two timestamps.

Seems like biggest downtime was 3 hours

Would a 1 day downtime kill the project?
In what way?
How is Scroll Priced? Can it be DOSSed relatively easily?


### [I-04] Exchange Rate Feeds

Question is if these are creating realistic arbitrages and delays between the prices on SCROLL and the Prices on Mainnet
I'd need to normalize them and find the relative deltas to see if that's the case

- The main question here is: Arbitrage
- Value Leak due to not using Market Rate (in case of exploit, in case of big liquidity crunch)
----


**0x57bd9E614f542fB3d6FeF2B744f3B813f0cc1258 - weETH / eETH Exchange Rate**

```js
pricesWithinTimePeriod 200
mean 1047992809278279200
STANDARD DEVIATION 4586508011714059
as percent of mean 0.43764689710730437
getHighestAndLowestPrice 1055895708780116400 1040176765876107400


interval 1 Week

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1723638354, price: 1045578046819054700 },
  { date: 1723119744, price: 1044895476259292500 }
]
deviation 0.06532429082818567


interval One Day

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1730897218, price: 1052073665612945800 },
  { date: 1730810818, price: 1051990739275359400 }
]
deviation 0.0078828011018


interval 4 Hours

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing []


interval One Hour

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing []
```

0.007882801101796958 - Below dev

So you have to expect each operation to be due to time and not price




**0xE61Da4C909F7d86797a0D06Db63c34f76c9bCBDC - wstETH-ETH Exchange Rate**

```js
pricesWithinTimePeriod 400
mean 1168659431678961200
STANDARD DEVIATION 11860049778819950
as percent of mean 1.014842259201309
getHighestAndLowestPrice 1188596651060688600 1147270030912712700


interval 1 Week

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1723243491, price: 1175451889673963300 },
  { date: 1722724889, price: 1174628840044321500 }
]
deviation 0.07006891041519739


interval One Day

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1733865222, price: 1187362676214505200 },
  { date: 1733778822, price: 1187251706061207600 }
]
deviation 0.009346809335470692


interval 4 Hours

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing []


interval One Hour

eth_usd_swings.pointsOfBiggestNegativeSwing []
```

0.0093468093354653 - Below dev

### [I-05] Key Security Invariants

---

**Whenever I open a trove, I get the debt assigned to me, with at most 1e9 in error**

**I can never open a trove and be liable for zero debt**

**Once a Batch is rebased, no new troves can be opened in it, nor added to it, nor any debt can be increased**
The only succesful function must be `removeTroveFromBatch`

---

**Shutdown only ever reverts becasue of this error**
No other reason

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/BorrowerOperations.sol#L1222-L1223

```solidity
        if (TCR >= SCR()) revert TCRNotBelowSCR();

```

**Shutdown only ever reverts becasue of this error**
No other reason

https://github.com/subvisual/quill/blob/d4a5dcc168dfc315eef6a4c9c465a36c86ca0ddc/contracts/src/BorrowerOperations.sol#L1222-L1223

```solidity
        if (TCR >= SCR()) revert TCRNotBelowSCR();

```


**_activePool.getCollBalance() - _collRemainder**
Never revert


**Never Revert**
https://github.com/GalloDaSballo/quill-review/blob/8d6e4c8ed0759cea1ff0376db9fd55db864cd7e8/contracts/src/TroveManager.sol#L1096-L1103

```solidity
    function _updateSystemSnapshots_excludeCollRemainder(IActivePool _activePool, uint256 _collRemainder) internal {
        totalStakesSnapshot = totalStakes;

        uint256 activeColl = _activePool.getCollBalance();
        uint256 liquidatedColl = defaultPool.getCollBalance();
        totalCollateralSnapshot = activeColl - _collRemainder + liquidatedColl;
    }

```



**Stake math**

My stake accurately represents my % deposits of all collateral (ignoring precision loss due to redistribution and division)

https://github.com/GalloDaSballo/quill-review/blob/8d6e4c8ed0759cea1ff0376db9fd55db864cd7e8/contracts/src/TroveManager.sol#L1037-L1053

```solidity

    function _computeNewStake(uint256 _coll) internal view returns (uint256) {
        uint256 stake;
        if (totalCollateralSnapshot == 0) {
            stake = _coll;
        } else {
            /*
            * The following assert() holds true because:
            * - The system always contains >= 1 trove
            * - When we close or liquidate a trove, we redistribute the redistribution gains, so if all troves were closed/liquidated,
            * rewards would’ve been emptied and totalCollateralSnapshot would be zero too.
            */
            // assert(totalStakesSnapshot > 0);
            stake = _coll * totalStakesSnapshot / totalCollateralSnapshot;
        }
        return stake;
    }
```

**Never reverts with Overflow**

https://github.com/subvisual/quill/blob/23e53123a16b12614d25bfb715e17dd41bcebbdd/contracts/src/TroveManager.sol#L299-L307

```solidity

    function _liquidate(
        IDefaultPool _defaultPool,
        uint256 _troveId,
        uint256 _boldInStabPool,
        uint256 _price,
        LatestTroveData memory trove,
        LiquidationValues memory singleLiquidation
    ) internal {
```



