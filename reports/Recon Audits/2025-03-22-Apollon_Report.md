

**Auditors**

[Alex The Entreprenerd](https://x.com/gallodasballo?lang=en)



# Findings

## High Risk



### [H-01] `updateSystemSnapshots_excludeCollRemainder` lacks access control

**Impact**

The function `updateSystemSnapshots_excludeCollRemainder` doesn't have access control meaning that stakes can be manipulated

I found this by running invariant tests, with the simple check that no call should ever succeed


```solidity
 => [call] CryticTester.troveManager_updateSystemSnapshots_excludeCollRemainder((address,uint256)[])([]) (addr=0xA647ff3c36cFab592509E13860ab8c4F28781a66, value=0, sender=0x0000000000000000000000000000000000030000)
         => [call] TroveManager.updateSystemSnapshots_excludeCollRemainder((address,uint256)[])([]) (addr=0x48E4F3f3daE11341ff2eDF60Af6857Ae08C871C5, value=0, sender=0xA647ff3c36cFab592509E13860ab8c4F28781a66)
                 => [event] SystemSnapshotsUpdated([], [])
                 => [return ()]
         => [event] Log("should never be possible")
         => [panic: assertion failed]
```

**Mitigation**

Add a check to ensure that the caller is LiquidationOperations

## [H-02] `claimUnassignedAssets` is increasing debt but not checking for it in `_finaliseTrove`, opening up for self-liquidations

**Impact** 

`claimUnassignedAssets` will increase the amount of debt and collateral to a trove by a certain amount

Resulting in a Coll Increase and a Debt Increase

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L614-L665

```solidity
function claimUnassignedAssets(
    uint _percentage,
    address _upperHint,
    address _lowerHint,
    bytes[] memory _priceUpdateData
  ) external payable override {
    if (_percentage == 0) revert ZeroDebtChange();
    /// @audit Self Liquidation Risk?
    address borrower = msg.sender;
    (ContractsCache memory contractsCache, LocalVariables_adjustTrove memory vars) = _prepareTroveAdjustment(
      borrower,
      _priceUpdateData,
      false
    );

    // handle debts
    DebtTokenAmount[] memory debtsToAdd = new DebtTokenAmount[](vars.priceCache.debtPrices.length);
    for (uint i = 0; i < vars.priceCache.debtPrices.length; i++) {
      address debtToken = vars.priceCache.debtPrices[i].tokenAddress;

      uint unassignedDebt = contractsCache.storagePool.getValue(debtToken, false, PoolType.Unassigned);
      if (unassignedDebt == 0) continue;

      uint toClaim = (unassignedDebt * _percentage) / DECIMAL_PRECISION;
      if (unassignedDebt == 0) continue;

      contractsCache.storagePool.transferBetweenTypes(debtToken, false, PoolType.Unassigned, PoolType.Active, toClaim);
      debtsToAdd[i] = DebtTokenAmount(IDebtToken(debtToken), toClaim, 0);
    }
    vars.newCompositeDebtInUSD += _getCompositeDebt(contractsCache.priceFeed, vars.priceCache, debtsToAdd);
    contractsCache.troveManager.increaseTroveDebt(borrower, debtsToAdd);

    // handle colls
    TokenAmount[] memory collsToAdd = new TokenAmount[](vars.priceCache.collPrices.length);
    for (uint i = 0; i < vars.priceCache.collPrices.length; i++) {
      address collToken = vars.priceCache.collPrices[i].tokenAddress;

      uint unassignedColl = contractsCache.storagePool.getValue(collToken, true, PoolType.Unassigned);
      if (unassignedColl == 0) continue;

      uint toClaim = (unassignedColl * _percentage) / DECIMAL_PRECISION;
      if (unassignedColl == 0) continue;

      contractsCache.storagePool.transferBetweenTypes(collToken, true, PoolType.Unassigned, PoolType.Active, toClaim);
      collsToAdd[i] = TokenAmount(collToken, toClaim);
    }
    vars.newCompositeCollInUSD += _getCompositeColl(contractsCache.priceFeed, vars.priceCache, collsToAdd);
    vars.newIMCR = _calculateIMCR(vars, collsToAdd, true);
    contractsCache.troveManager.increaseTroveColl(borrower, collsToAdd);

    _finaliseTrove(false, false, contractsCache, vars, borrower, _upperHint, _lowerHint);
  }
```

but it's calling ```_finaliseTrove(false, false,``` which means that these checks applied when `_isDebtIncrease == true` are going to be skipped

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L978-L986

```solidity
    if (_vars.isInRecoveryMode) {
      // BorrowerOps: Collateral withdrawal not permitted Recovery Mode
      if (_isCollWithdrawal) revert CollWithdrawPermittedInRM();
      if (_isDebtIncrease) _requireICRisAboveCCR(_vars.newICR);
    } else {
      // if Normal Mode

      // check if the individual minimum collateral ratio is met (based on the used coll types)
      if (_isCollWithdrawal || _isDebtIncrease) _requireICRisAboveIMCR(_vars.newICR, _vars.newIMCR);
```

**Mitigation**

Change the code to

```solidity
_finaliseTrove(false, true, contractsCache, vars, borrower, _upperHint, _lowerHint);
```

### [H-03] `BorrowerOperations` Inconsistent IMCR logic could allow risky collaterals to have a higher CollerateralRatio during Recovery Mode

**Impact**

`_requireValidAdjustmentInCurrentMode` is a key part of the security checks for adjusting and opening Troves

From discussing with the team, I was made aware that some Collaterals may have a IMCR that would be above CCR (e.g. 175% CR)

Given this fact, we can check that the logic in `_requireValidAdjustmentInCurrentMode` will relax it's IMCR requirements during recovery mode:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L978-L986

```solidity
    if (_vars.isInRecoveryMode) {
      // BorrowerOps: Collateral withdrawal not permitted Recovery Mode
      if (_isCollWithdrawal) revert CollWithdrawPermittedInRM();
      if (_isDebtIncrease) _requireICRisAboveCCR(_vars.newICR);
    } else {
      // if Normal Mode

      // check if the individual minimum collateral ratio is met (based on the used coll types)
      if (_isCollWithdrawal || _isDebtIncrease) _requireICRisAboveIMCR(_vars.newICR, _vars.newIMCR);
```

In Normal Mode:
```solidity
 _requireICRisAboveIMCR(_vars.newICR, _vars.newIMCR);
```

e.g.
```
X > 175%
```

In Recovery Mode
```solidity
 _requireICRisAboveCCR(_vars.newICR);
```

```
X > 150%
```

Meaning that due to not additionally checking for IMCR, some borrowers will have more borrowing power in Recovery Mode

**Mitigation**

You can quickly mitigate this by:
- Enforcing that all CollateralRatios are below CCR
- Adding an additional check for IMCR in all modes, at all times

It's worth exploring whether certain collaterals could not be added during Recovery Mode as to further reduce risks, I think this would require simulating various prices for specific collaterals

### [H-04] Lack of min borrow + min fee allows Spam Opening troves to trigger Recovery Mode

**Impact**

This finding chains multiple other observations to borrow for free

Because a Trove can be opened with 0 net debt, such trove won't pay a borrow fee

By opening a myriad of Troves, with a ICR < TCR we can drag the TCR down

By choosing an oracle price that is a valid negative update (for collateral, or positive update for debt denomination), we can hurt the ICR of these position slightly

When the system is in Recovery Mode, no borrowing fee is paid on opening a position, this can help borrow more stablecoin as a means to raise the ownership percentage of attacker in the Stability Pool, making the liquidations directly profitable to them

This allows to trigger Recovery Mode at will, and liquidate any victim with ICR < TCR

**Proof of Concept**

- Setup by opening a myriad of Troves at ICR < TCR
- Update the price to trigger Recovery Mode
- Open the "real" trove a user wanted to open
- Borrow and bypass fees
- Liquidate Victims
- Close all other Troves that were opened for the setup

This can be fully automated with a smart contract that creates new proxies that open a Trove each

This could be used for 3 key reasons:
- Trigger Recovery Mode and Liquidate other people
- Borrow for free
- Raise the total amount of debt in the system to reduce the net fee on redemptions

**Mitigation**

I believe that Oracle price being non-deterministic on each block is a key issue

Additionally the fact that no minimum borrow size is enforced, means that these 0-net-debt are effectively free to open, whereas if some fee was charged that wouldn't be the case

Alternatively, you could always enforce a borrow fee at all times, this would have the downside of making liquidations less profitable and should be further researched

### [H-05] `StakingOperations` Token Transfer is updating the total supply before accruing rewards to users causing loss of rewards

**Impact**

```solidity
 function updatePool(ISwapPair _pid) public {
    PoolInfo storage pool = poolInfo[_pid];

    // check
    if (block.timestamp <= pool.lastRewardTime) return;

    // update
    uint tokenSupply = _pid.balanceOf(address(this));
    if (tokenSupply == 0 || totalAllocPoint == 0) {
      pool.lastRewardTime = block.timestamp;
      return;
    }
    uint multiplier = block.timestamp - pool.lastRewardTime;
    uint reward = (multiplier * rewardsPerSecond * pool.allocPoint) / totalAllocPoint;
    pool.accRewardPerShare += (reward * REWARD_DECIMALS) / tokenSupply;
    pool.lastRewardTime = block.timestamp;
  }
```

-> Transfer
-> New Balance
-> Check points (user uses old balance, but total supply is the new one)
-> User lost rewards
-> New total Supply and correct total debt

In fixing this be careful not to cause a division by zero (as the initial deposit would)

This can only happen on deposits (which dilute rewards), if the same mistake was done on withdrawals, then value could be stolen


**Mitigation**

Because the code is tightly coupled, I think accruing and then transferring the tokens could be sufficient

Alternatively the scalable fix is to track the previous balance in storage as you suggested

I think tracking in storage is the best fix long term (allows to re-use the contract separately without bugs)

### [H-06] `StakingOperations.claim` doesn't update reward debt, allowing multiple claims

**Impact**

```solidity
  function claim(ISwapPair _pid) external override {
    requireValidPool(_pid);
    _claim(_pid, msg.sender);
  }

  function batchClaim(ISwapPair[] memory _pids) external override {
    uint length = _pids.length;
    for (uint n = 0; n < length; n++) {
      requireValidPool(_pids[n]);
      _claim(_pids[n], msg.sender);
    }
```

User rewards are computed as `((user.amount * accRewardPerShare) / REWARD_DECIMALS) - user.rewardDebt;`

`user.rewardDebt = (user.amount * pool.accRewardPerShare) / REWARD_DECIMALS;` is set in deposit and withdraw

but it is not update on `claim` and `batchClaim`

Due to this, an exploiter can simply call `claim` and `batchClaim` repeatedly and steal all rewards

**Mitigation**

Change `_claim` to:
- Calculate user rewards and cache that value
- Set user debt to the new value, so their `pendingReward` are now zero
- Then transfer the tokens

Also consider implementing the following invariants:
- `pendingReward` are set to zero after a call to `claim` and `batchClaim`

### H-07 Pull Based Oracle opens up to triggering Recovery Mode and liquidating other Troves as a risk free arbitrage

**Executive Summary**

Pull Based oracle can be viewed as offering an attacker the ability to chose a combination of price that are not stale

This, combined with the possibility of lowering the TCR down via ones own positions opens up to a risk free arbitrage in which an attacker drags the TCR down to trigger Recovery Mode, and then they liquidate other people troves

**Description**

<img width="629" alt="Screenshot 2024-08-08 at 11 26 02" src="https://github.com/user-attachments/assets/6d3e67ef-fa98-43cc-831f-582671ca8aea">

Pyth Pull Oracles can be viewed as a sequence of prices, `p0` to `pn` where any ordered sequence `p0 -> px -> pn` can be picked as long as all prices are within the staleness threshold

Recovery Mode allows healthy Troves to be liquidated when the TCR falls below a certain threshold

Liquidations can be performed arbitrarily and out of order, as long as a Trove meets the requirements for Liquidations

Due to this, an attacker can push the TCR down very close to Recovery Mode and then update prices to a more negative price to trigger Recovery Mode, and perform liquidations.

This is a risk free opportunity that could be abused any time the threshold for profit is met:
- `Profit = Gain from Liquidation * Ownership % - Opening Fees - Gas Costs`


**Proof Of Concept**

- Wait for a price sequence that will result in a decrease of TCR
- Use the older price (healthier price)
- Flashloan funds, borrow to bring the TCR very close to Recovery Mode
- Deposit into the Stability Pool, ideally a very high amount to maximize profits
- Update Prices, Recovery Mode is triggered
- Liquidate other Troves (ideally out of order, to liquidate a higher amount of collateral)
- Attacker closes their position

**Mitigation**

A few ideas for mitigation:
- Offer a Grace Period in which Recovery Mode liquidations cannot happen, this can be a short delay (e.g. 15 minutes), which should allow victims to react (as long as they have setup monitoring and automations) - This is the safer option with the clear downside of a risk of desynching the state (as Recovery Mode may be engaged, but the timer would require an external call to be triggered)

- You could enforce a different ratio at which opening Troves can be performed, but this would limit opening CRs always above the CCR, a buffer could be gamed by setting up Troves that over time have their CR below the CCR, or by setting up a very heathy Trove and then closing it

- You may explore whether it's possible to always use the latest Pyth Price and enforce that only the latest price is used, this  main risk would be the inability to perform operations anytime a transaction was in the mempool for too long

Also note that because you have a mixture of Collaterals and Debt Tokens, the opportunity to trigger Recovery Mode is higher than if you only used a single Collateral and Debt Pair

## Medium Risk

### [M-01] `RedemptionOperations` Redemptions should be disabled during Recovery Mode

**Impact**

`RedemptionOperations` checks the `TCR < MCR`, but should most likely check for `TCR < CCR`

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/RedemptionOperations.sol#L101-L103

```solidity
    (, uint TCR, , ) = storagePool.checkRecoveryMode(vars.priceCache); /// @audit High? not checking RM -> Mint for free, Inflate total supply (pay no redemption), redeem at small fee
    if (TCR < MCR) revert LessThanMCR(); /// @audit-ok force to liquidate if all system is insolvent

```

This is because during Recovery Mode, minting fees are voided, meaning that the system may open up to additional arbitrages via Redemptions

Redemptions are generally disabled during RM in favour of liquidations


**Mitigation**

Investigate if additional arbitrages could be detrimental to your protocol due to reduced minting fees

From checking liquity they use a check similar to yours:
https://github.com/liquity/dev/blob/e38edf3dd67e5ca7e38b83bcf32d515f896a7d2f/packages/contracts/contracts/TroveManager.sol#L948-L962



### [M-02] `_exchangeRate` value validation looks incorrect

**Impact**

By definition a value for `_exchangeRate` should be within the bounds of `STOCK_SPLIT_PRECISION`

```solidity
    if (_exchangeRate >= -STOCK_SPLIT_PRECISION && _exchangeRate < STOCK_SPLIT_PRECISION) revert InvalidExchangeRate();
```


**Mitigation**

It's unclear if this is incorrect

Either way I highly recommend simplifying the logic for stock splits to allow the owner to set a scalar multiplier and a divisor, this makes the logic a lot simpler with no additional risks

### [M-03] `RedemptionOperations` allows redemption against stale prices

**Impact**

The code for `_calculateTroveRedemption` doesn't validate that collaterals nor debts prices are not stale

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/RedemptionOperations.sol#L231-L259

```solidity
  function _calculateTroveRedemption(
    PriceCache memory _priceCache,
    address _borrower,
    uint _redeemMaxAmount,
    bool _includePendingRewards
  ) internal view returns (SingleRedemptionVariables memory vars) {
    address stableCoinAddress = address(tokenManager.getStableCoin());

    // stable coin debt should always exists because of the gas comp
    TokenAmount[] memory troveDebt = _includePendingRewards
      ? troveManager.getTroveRepayableDebts(_priceCache, _borrower, true) // with pending rewards
      : troveManager.getTroveDebt(_borrower); // without pending rewards
    if (troveDebt.length == 0) revert InvalidRedemptionHint();
    for (uint i = 0; i < troveDebt.length; i++) {
      TokenAmount memory debtEntry = troveDebt[i];

      if (debtEntry.tokenAddress == stableCoinAddress) vars.stableCoinEntry = debtEntry;
      vars.troveDebtInUSD += priceFeed.getUSDValue(_priceCache, debtEntry.tokenAddress, debtEntry.amount);
    }

    vars.collLots = _includePendingRewards
      ? troveManager.getTroveWithdrawableColls(_borrower)
      : troveManager.getTroveColl(_borrower);
    for (uint i = 0; i < vars.collLots.length; i++) {
      uint p = priceFeed.getUSDValue(_priceCache, vars.collLots[i].tokenAddress, vars.collLots[i].amount);
      vars.troveCollInUSD += p;
      if (!tokenManager.isDebtToken(vars.collLots[i].tokenAddress)) vars.redeemableTroveCollInUSD += p;
    } /// @audit can change the ratio of a trove to have mostly `isDebtToken` debt | How does this relate to collateral?

```

This opens up to arbitrage opportunities where the pyth price may be higher for a certain collateral, while the fallback oracle price is lower, allowing the caller to receive more collateral than what would be the fair market price

**Mitigation**

Rethink oracle price staleness checks to prevent arbitrages

### [M-04] `LiquidationOperations.batchLiquidateTroves` redistributes bad debt and collateral after all operations, meaning it will allow skipping bad debt redistribution during liquidations

**Impact**

The code for `batchLiquidateTroves` is as follows:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/LiquidationOperations.sol#L106-L137

```solidity
  function batchLiquidateTroves(address[] memory _troveArray, bytes[] memory _priceUpdateData) public payable override {
    if (!troveManager.enableLiquidationAndRedeeming()) revert LiquidationDisabled(); /// @audit MUST be separate
    if (_troveArray.length == 0) revert EmptyArray();

    LocalVariables_OuterLiquidationFunction memory vars;

    // update prices and build price cache
    priceFeed.updatePythPrices{ value: msg.value }(_priceUpdateData); /// @audit custom update on oracle prices, doesn't guarantee everything will be valid
    vars.priceCache = priceFeed.buildPriceCache();

    (vars.isRecoveryMode, vars.TCR, vars.entireSystemCollInUSD, vars.entireSystemDebtInUSD) = storagePool
      .checkRecoveryMode(vars.priceCache);
    vars.remainingStabilities = stabilityPoolManager.getRemainingStability(vars.priceCache);
    _initializeEmptyTokensToRedistribute(vars); // all set to 0 (nothing to redistribute)

    bool atLeastOneTroveLiquidated = false;
    for (uint i = 0; i < _troveArray.length; i++) {
      address trove = _troveArray[i];
      if (!troveManager.isTroveActive(trove)) continue; // Skip non-active troves | // @audit TODO: Are these ones with 0 debt?
      if (troveManager.getTroveOwnersCount() <= 1) continue; // don't liquidate if last trove /// @audit Break?

      bool liquidated = _executeTroveLiquidation(vars, trove); /// @audit Out of order Sorting based on Risk, to maximize profit
      if (liquidated && !atLeastOneTroveLiquidated) atLeastOneTroveLiquidated = true;
    }
    if (!atLeastOneTroveLiquidated) revert NoLiquidatableTrove();

    // move tokens into the stability pools
    stabilityPoolManager.offset(vars.priceCache, vars.remainingStabilities); /// @audit Redistribute, but pay from reserve?

    // and redistribute the rest (which could not be handled by the stability pool)
    troveManager.redistributeDebtAndColl(vars.priceCache, vars.tokensToRedistribute); /// @audit But computed here ??

```

Where in the loop, liquidations are done on Troves for which their Debt and Coll is computed on pre-liquidation storage values:
https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/LiquidationOperations.sol#L186-L192

```solidity
    (
      vars.troveAmountsIncludingRewards,
      vars.IMCR,
      vars.troveCollInUSD,
      vars.troveDebtInUSD,
      vars.troveDebtInUSDWithoutGasCompensation
    ) = troveManager.getEntireDebtAndColl(outerVars.priceCache, trove);
```

Meaning that redistributions of Debt and Coll are skipped while doing the batch liquidation

This will result in a higher premium to liquidators than intended, and a higher loss to CollStakers when the redistribution happens

**Mitigation**

This is an issue with how the math for liquidation is computed, because it applies only to very specific edge cases, it may be best to let end users know as the fix would require having in-memory accounting of debt redistribution, which will increase complexity

### [M-05] Decay Coefficient could round down and have an effective slower decay

**Impact**

`calcDecayedStableCoinBaseRate` calls `_minutesPassedSinceLastFeeOp` which rounds down by up to 1 minute - 1

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/TroveManager.sol#L987-L997

```solidity
  function calcDecayedStableCoinBaseRate() public view override returns (uint) {
    uint minutesPassed = _minutesPassedSinceLastFeeOp();
    uint decayFactor = LiquityMath._decPow(MINUTE_DECAY_FACTOR, minutesPassed);

    return (stableCoinBaseRate * decayFactor) / DECIMAL_PRECISION;
  }

  function _minutesPassedSinceLastFeeOp() internal view returns (uint) {
    return (block.timestamp - lastFeeOperationTime) / 1 minutes;
  }

```

This, in conjunction with the logic `_updateLastFeeOpTime`

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/TroveManager.sol#L980-L984

```solidity
    uint timePassed = block.timestamp - lastFeeOperationTime;
    if (timePassed >= 1 minutes) { /// @audit Can we abuse this in some way? | See ETHOS and eBTC findings
      lastFeeOperationTime = block.timestamp;
      emit LastFeeOpTimeUpdated(block.timestamp);
    }
```

will make the decay factor decay slower than intended

This finding was found in the ETHOS contest by Chaduke:
https://github.com/code-423n4/2023-02-ethos-findings/issues/33





### [M-06] `TroveManager``_calcBorrowingRate` always returns `borrowingFeeFloor`

**Impact**

`_calcBorrowingRate` is using `min(X + Y, X)` meaning it will always return `X` in this case `borrowingFeeFloor`

**Mitigation**

Change 
```solidity
  function _calcBorrowingRate(uint _stableCoinBaseRate) internal view returns (uint) {
    return LiquityMath._min(borrowingFeeFloor + _stableCoinBaseRate, borrowingFeeFloor);
  }
```

To
```solidity
  function _calcBorrowingRate(uint _stableCoinBaseRate) internal view returns (uint) {
    return LiquityMath._min(borrowingFeeFloor + _stableCoinBaseRate, 1e18);
  }
```


### [M-07] Ticking Interest Rate opens up to multi-block MEV - Directly Triggering Recovery Mode on the next block due to interest ticking

**Impact**

Because Apollon charges an interest on borrowing, an attack can guarantee triggering Recovery Mode on the next block by simply borrowing up to the threshold

Triggering Recovery mode would then allow liquidating Troves that are below the TCR

This puts the attacker at risk as well, however with some setup the attack can be +EV, posing a close to unmitigatable risk to other Trove owners


**Mitigation**

Recovery Mode liquidations being too easily accessible is a big risk for users leveraging up

In order to avoid this you could opt-into:
1) Enforcing a Buffer for Opening Positions
- This unfortunately has the downside of allowing the triggering of Recovery Mode via multiple positions

2) Changing the mechanisms around how Recovery Mode works
- This requires extensive work, possible solutions can be: Increasing the fee as you open a Trove that brings the sytem towards recovery mode (Make the attack economically expensive)

3) Remove or Alter the logic for Recovery Mode by enforcing higher Liquidations
- This requires economic modelling

4) Introduce a Delay for Recovery Mode Liquidations like we did in eBTC
- This doesn't remove the risk but reduces it and makes the attack more expensive

### [M-08] `BorrowerOperations` alters user debt but enforces prices are not stale only for debts that are being actively altered

**Impact**

A common pattern used in Apollon for price validation looks as follows:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L248-L270

```solidity
  function addColl(
    TokenAmount[] memory _colls,
    address _upperHint,
    address _lowerHint,
    bytes[] memory _priceUpdateData
  ) public payable override {
    address borrower = msg.sender;
    (ContractsCache memory contractsCache, LocalVariables_adjustTrove memory vars) = _prepareTroveAdjustment(
      borrower,
      _priceUpdateData,
      false
    );

    // revert debt token deposit if prices are untrusted
    for (uint i = 0; i < _colls.length; i++) {
      if (!contractsCache.tokenManager.isDebtToken(_colls[i].tokenAddress)) continue;

      TokenPrice memory tokenPriceEntry = contractsCache.priceFeed.getTokenPrice(
        vars.priceCache,
        _colls[i].tokenAddress
      );
      if (!tokenPriceEntry.isPriceTrusted) revert UntrustedOraclesDebtTokenDeposit(); /// @audit verifying Colls
    } /// But this is user param not all colls the user has
```

Where `_cols` is a user provided parameter

Similarly to increase debt:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L372-L395

```solidity
  function _increaseDebt(
    address _borrower,
    address _to,
    TokenAmount[] memory _debts,
    MintMeta memory _meta,
    bytes[] memory _priceUpdateData
  ) internal {
    (ContractsCache memory contractsCache, LocalVariables_adjustTrove memory vars) = _prepareTroveAdjustment(
      _borrower,
      _priceUpdateData,
      true //will be called by swap operations, that handled the price update
    );

    _requireValidMaxFeePercentage(contractsCache.troveManager, _meta.maxFeePercentage, vars.isInRecoveryMode);
    for (uint i = 0; i < _debts.length; i++) {
      _requireNonZeroDebtChange(_debts[i].amount);

      // revert minting if the oracle is untrusted
      TokenPrice memory tokenPriceEntry = contractsCache.priceFeed.getTokenPrice(
        vars.priceCache,
        _debts[i].tokenAddress
      );
      if (!tokenPriceEntry.isPriceTrusted) revert UntrustedOraclesMintingIsFrozen();
    }
```

Where `debts` is passed by `SwapOperations`

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L271-L293

```solidity
    if (vars.fromMintA != 0 || vars.fromMintB != 0) {
      TokenAmount[] memory debtsToMint;
      if (vars.fromMintA != 0 && vars.fromMintB != 0) {
        // mint both
        debtsToMint = new TokenAmount[](2);
        debtsToMint[0] = TokenAmount(tokenA, vars.fromMintA);
        debtsToMint[1] = TokenAmount(tokenB, vars.fromMintB);
      } else {
        // mint only 1 token
        debtsToMint = new TokenAmount[](1);
        debtsToMint[0] = (
          vars.fromMintA != 0
            ? TokenAmount(tokenA, vars.fromMintA) // mint A
            : TokenAmount(tokenB, vars.fromMintB) // mint B
        );
      }
      borrowerOperations.increaseDebt{ value: msg.value }(
        msg.sender, // OK 
        vars.pair, // OK
        debtsToMint, // OK - See above
        _priceAndMintMeta.meta, /// @audit this looks griefable | TODO
        _priceAndMintMeta.priceUpdateData // TODO
      );
```

This risks missing:
- Multiple other debts -> When a user takes on more than one stock debt token
- Multiple other collaterals -> When a user has already used other collaterals for it's Collateralization Ratio

Due to this, multiple possible oracle related exploits are possible such as:
- Over borrowing
- Self-liquidation by borrowing and then updating a price
- Bypassing the restrictions on triggering Recovery mode

*Instances*
- `addColl`
- `withdrawColl`
- `increaseDebt`

**Mitigation**

I believe all prices must be validated for staleness, offering the ability of chosing between the Pyth and the Alternative Feed should be viewed as a risky opening for arbitrage, self-liquidations and recovery mode liquidations

### [M-09] Users could opt to never use Pyth and always rely on the fallback feed due to lack of validation on certain functions

**Impact**

The rationale for using Pyth and the Fallback oracle is logical:
Sometimes Pyth is unavailable

However, once Pyth becomes unavailable, people will have the option to constantly chose between Pyth and the fallback oracle

The fallback oracle is a push type oracle, meaning that it won't always be updated

This may create opportunity for arbitrage for:
- Redemptions
- Increasing Debts (as other account debts will not have their prices checked for staleness)

**Mitigation**

Overall you should rethink the FSM around how stale vs trusted prices could be used as the current implementation opens up for a lot of arbitrage and edge cases

You should consider changing fees based on the oracle you're using

An oracle deviation threshold + time to update are inherently +EV to arbitrageurs
You should consider changing fees based on which oracle is being used, where Pyth could have a lower fee and the fallback would most likely have to charge a higher fee

### [M-10] `SwapOperations` `swapFee` is non deterministic and can cause people to lose funds

**Impact**

SwapOperations charges a fee that is computed by `SwapPair` as follows:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L187-L209

```solidity
  function getSwapFee(uint postReserve0, uint postReserve1) public view override returns (uint feePercentage) {
    address nonStableCoin = token1; // find stable coin
    if (tokenManager.isDebtToken(nonStableCoin) && totalSupply > 0) {
      // query prices
      (uint oraclePrice, ) = priceFeed.getPrice(nonStableCoin); /// @audit not validated for staleness
      uint preDexPrice = _getDexPrice(reserve0, reserve1); /// @audit this is spot and can be altered in some way
      uint postDexPrice = _getDexPrice(postReserve0, postReserve1);

      // only apply the dynamic fee if the swap trades against the oracle peg
      if ( /// @audit ??? | No deviation threshold | Spot vs Oracle (Donate to reserves?)
        (postDexPrice > oraclePrice && postDexPrice > preDexPrice) ||
        (postDexPrice < oraclePrice && preDexPrice > postDexPrice)
      ) {
        uint avgDexPrice = (preDexPrice + postDexPrice) / 2;
        uint priceRatio = (oraclePrice * DECIMAL_PRECISION) / avgDexPrice; // 1e18
        return
          _calcFee(priceRatio > DECIMAL_PRECISION ? priceRatio - DECIMAL_PRECISION : DECIMAL_PRECISION - priceRatio) +
          swapOperations.getSwapBaseFee();
      }
    }

    return swapOperations.getSwapBaseFee();
  }
```

The fee uses the previous "spot ratio" and compares it against the new pre-fee "spot ratio"

This means that any transaction that alters the reserves between when a user signs their transaction and when it get's included in a block may alter the fee that a user pays

This creates scenario where, if a minority of users owns the majority of the LP tokens, they can perform donations to front-run swaps, which will cause benign users to pay an additional fee that they didn't intend to pay

**Mitigation**

Add an explicit check for the fees that will be paid on a swap as to ensure that's intended or within an acceptable range

### [M-11] `SwapOperations` is performing some swaps without updating the price, resulting in incorrect fees being charged

**Impact**
Operations such as: `swapExactTokensForTokens`, `swapTokensForExactTokens` receive `_priceUpdateData` but update the price only at the `_swap` operation

`_swap` uses `getAmountsOut` and `getAmountsIn` to determine the price at which to charge fees at

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L469-L484

```solidity

  function swapExactTokensForTokens(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline,
    bytes[] memory _priceUpdateData
  ) public payable virtual override ensure(deadline) returns (SwapAmount[] memory amounts) {
    amounts = getAmountsOut(amountIn, path); /// @audit Used old prices
    if (amounts[amounts.length - 1].amount < amountOutMin) revert InsufficientOutputAmount();

    safeTransferFrom(path[0], msg.sender, getPair[path[0]][path[1]], amounts[0].amount);
    _swap(amounts, path, to, _priceUpdateData, false); /// @audit Updates prices after having used old prices
  }
```

This means that fees on swaps are computed on the stale price rather then the latest

Because oracle prices can be updated via other operations, this also allows swappers to pick the most optimal price at which to pay the lowest fee  


**Mitigation**

You can ensure the price is updated by:
- Forcing the update in all external functions
- Performing the staleness check in `getSwapFee`

### [M-12] Pull Based Oracle may allow for profitable self-liquidations

**Executive Summary**

Pull based oracle allow using more than one price within the last 5 minutes, this opens up to a myriad of combinations that may allow for self-liquidations

**Description**

The likelihood that a price moves by 10% is fairly low

However, when you start adding more collaterals and debt types, the likelihood that the combination of any two Collateral and Debt asset to have high volatility between each other raises

Meaning that when considering risk parameters, you shouldn't simply look at a debt or collateral price against USD, but also the ratio and correlation between those assets, as the correlation may open up to higher swings that may cause the protocol to lock-in bad debt


**Proof Of Concept**

A sample scenario would look as follows:
- Find coll and asset that have 11% difference in price from current price to new price
- Open up a trove
- Borrow as much as possible
- Deposit into the stability pool
- Update the prices
- Get liquidated, with bad debt

The likelihood of the profitability is reduced the more deposits are done in the stability pool and the higher opening fees are

**Mitigation**

Unfortunately this is a statistical risk, and to mitigate it you'd need to:
- Research all historical Pyth Prices
- Check for realized volatility of assets
- Add a buffer to account for errors or additional realized volatility
- Move the MICR to a value that is consistent with this risk

### [M-13] Operative Risks tied to changing Risk Based Parameter

**Executive Summary**

This is a collection of operative risks that come from maintaining and updating Apollon

I highly recommend you go through this list, create your own list, and ensure that at all times these risks are considered


**Updating `setCollTokenSupportedCollateralRatio` can cause multiple economic exploits**

```solidity
  function setCollTokenSupportedCollateralRatio(
    address _collTokenAddress,
    uint _supportedCollateralRatio
  ) external override onlyOwner {
    if (_supportedCollateralRatio < MCR) revert SupportedRatioUnderMCR();
    collTokenSupportedCollateralRatio[_collTokenAddress] = _supportedCollateralRatio;
    emit CollTokenSupportedCollateralRatioSet(_collTokenAddress, _supportedCollateralRatio);
  }
```

Updating this ratio can:

- Cause Recovery Mode
- Be sandwhiched to trigger Recovery Mode
- Cause Liquidations
- Be sandwhiched to cause self-liquidations

The setter itself is not a vulnerability, however, the mechanisms around changing these risk-based values are very commonly a pre-condition to Critical Severity Exploits

The most important consideration is tied to how exactly a change in Collateral Ratio would be enacted

- Can you pause minting and borrowing?
- Can you verify that all users are solvent and will remain solvent after the change?
- Can you have a buffer that will prevent actors from having a step-wise change in their Collateralization Ratio?
- Will the governance proposal be executable by anyone?
- Will you liquidate any unhealthy position as part of the proposal?

Due to the complexity, I'm flagging this as a delicate Operational Security area, however, I will not be able to provide specific advice at this time

**`setAlternativePriceFeed` can cause liquidations, self-liquidations or insolvency and bad debt**

This change could also cause positions to go from healthy to undercollateralized

The change may also be sandwiched

More importantly, if governance changes can be broadcasted by anyone, the sandwiched will not be mitigable and would be a perfect opportunity for an economic exploit

**Gov token must be configured**

Since gov token is used as part of reserve pool, then it must be configured to have some validity as collateral

**Mitigation**

Recognize the risks tied to changing these settings and plan accordingly, do consult Security Researchers at that time

## Low Risk

### [L-01] `LiquidationsOperations` Comment around `_emitLiquidationSummaryEvent` is incorrect

**Impact**

`_emitLiquidationSummaryEvent` has the following comment
https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/LiquidationOperations.sol#L447-L455

```solidity

  function _emitLiquidationSummaryEvent(LocalVariables_OuterLiquidationFunction memory vars) internal {
    TokenAmount[] memory liquidatedColl = new TokenAmount[](vars.priceCache.collPrices.length);
    for (uint i = 0; i < vars.priceCache.collPrices.length; i++) {
      liquidatedColl[i] = TokenAmount(
        vars.priceCache.collPrices[i].tokenAddress,
        vars.tokensToRedistribute[i].amount // works because of the initialisation of the array (first debts, then colls) /// @audit Wrong comment
      );
    }
```

The comment looks wrong since the array fills in collaterals first and then debts

**Mitigation**

Check the comments and fix them

### [L-02] `LiquidationOperations` loop should `break` on last Trove

**Impact**

Apollon will not liquidate the last trove in `batchLiquidateTroves` 
However, the code is using a `continue` which means that the code will loop multiple times while performing no-ops

```solidity
     if (troveManager.getTroveOwnersCount() <= 1) continue; // don't liquidate if last trove
```

**Mitigation**

Change the code to

```solidity
      if (troveManager.getTroveOwnersCount() <= 1) break; // no more troves to liquidate
```

### [L-03] `massUpdatePools` needs to be capped due to OOG reverts

**Impact**

`massUpdatePools` looks as follows:

```solidity
 function massUpdatePools() public {
    uint length = pools.length;
    for (uint n = 0; n < length; n++) {
      updatePool(pools[n]);
    }
  }
```

Meaning it will iterate over all known pools

The gas limit on SEI is 10MLN gas per block

Assuming around 25k gas per update, that's 400 pools before the function reverts

I just did some quick napkin math on the amount of storage slots used, you should write a test to verify the limit as to avoid getting reverts in prod

That said, anything below 100 pools will have a high margin of safety

**Mitigation**

Ensure you do not surpass 100 pools as to avoid consuming too much gas which could cause reverts

### [L-04] `StakingOperation` will drip rewards to no-one if rewards are queued before any deposit

**Impact**

I believe they will have modest losses until someone deposits

This is further corroborated by the fact that the first deposit will continue instead of going through the early return path

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/StakingOperations.sol#L219-L230

```solidity
  function updatePool(ISwapPair _pid) public {
    PoolInfo storage pool = poolInfo[_pid];

    // check
    if (block.timestamp <= pool.lastRewardTime) return;

    // update
    uint tokenSupply = _pid.balanceOf(address(this));
    if (tokenSupply == 0 || totalAllocPoint == 0) {
      pool.lastRewardTime = block.timestamp;
      return;
    }
```

**Mitigation**

Simply delay the first emission by a few hours or days to ensure someone has staked

### [L-05] `SwapERC20` Hardcoded DOMAIN_SEPARATOR will cause issues and replay on a chain split

**Impact**

**DOMAIN_SEPARATOR** being hardcoded will either:
- Stop working if the chain hardforks with a new ID
- Will open up for replays if the chain forks

**Mitigation**

The code in `DebtToken` already deals with these issues, you can re-use that

### [L-06] Riskiest Trove can be made to pay compound interest cheaply

**Impact**

Apollon charges simple interest on Trove Accrual

In general Troves can only be accrued by their owners during state-changing operations


```solidity
    for (uint i = 0; i < _iterations.length; i++) {
      RedeemIteration memory iteration = _iterations[i];
      checkValidRedemptionHint(vars.priceCache, iteration.trove);
      troveManager.applyPendingRewards(iteration.trove, vars.priceCache); /// @audit The Hint may be wrong NOW, the CR may be underwater now
      SingleRedemptionVariables memory troveRedemption = _calculateTroveRedemption( /// TODO: KEY
        vars.priceCache,
        iteration.trove,
        _stableCoinAmount - vars.totalRedeemedStable,
        false // without pending rewards, because they got applied above
      );

      // resulting CR differs from the expected CR, we bail in that case, because all following iterations will consume too much gas by searching for a updated hints
      // allowing 1% deviation, because of time based borrowing interests
      if (troveRedemption.resultingCR > iteration.expectedCR) {
        if ((troveRedemption.resultingCR * DECIMAL_PRECISION) / iteration.expectedCR > 1.01e18) break; /// @audit Risk of overflow
      } else {
        if ((iteration.expectedCR * DECIMAL_PRECISION) / troveRedemption.resultingCR > 1.01e18) break;
      }
```

However, the riskiest trove can be made to accrue by redeeming a very small amount of debt (e.g. 1 wei)

**Mitigation**

Either document this risk, or enforce a minimum redemption size

### [L-07] Redemptions that redeem close to 100% of the Trove Debt may revert when the hint is inaccurate

**Impact**

After a redemption, this comparison is performed:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/RedemptionOperations.sol#L123-L129

```solidity
      // resulting CR differs from the expected CR, we bail in that case, because all following iterations will consume too much gas by searching for a updated hints
      // allowing 1% deviation, because of time based borrowing interests
      if (troveRedemption.resultingCR > iteration.expectedCR) {
        if ((troveRedemption.resultingCR * DECIMAL_PRECISION) / iteration.expectedCR > 1.01e18) break; /// @audit Risk of overflow
      } else {
        if ((iteration.expectedCR * DECIMAL_PRECISION) / troveRedemption.resultingCR > 1.01e18) break;
      }
```

Whenever expectedCR or resultingCR are greater than type(uint256).max / 1e18 the multiplication will overflow, causing a revert

This may be used to grief other people, or can happen naturally due to slight changes in prices, or other redemptions

**Mitigation**

It may be best to use a smaller factor such as `100` as to reduce the likelyhood of an overflow

### [L-08] Liquidation Logic will not work on all troves when the system is underwater

**Impact**

`batchLiquidateTroves` allows for out of order liquidations

In the edge case of all troves being underwater, meaning all troves should be liquidated, the following check will allow liquidations only starting from the riskiest trove

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/LiquidationOperations.sol#L196-L197

```solidity
    if (vars.ICR > outerVars.TCR) return false; /// @audit Looks wrong in edge cases

```

This is more of a gotcha than a real risk as having all Troves underwater is a failure scenario

**Mitigation**

The check could be refactored to ensure that if all Troves are underwater, the vast majority of the debt can still be liquidated

### [L-09] Up to (1e18 - 1) loss in interest paid due to rounding down

**Impact**

`stableInterest` in `_calculatePendingBorrowingInterest` rounds down the amount to be paid due to division before multiplication 

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/TroveManager.sol#L459-L462

```solidity
      stableInterest += /// Can find cheaper price to pay less, ultimately subject to "downward spiral" and other risks
        (((_priceFeed.getUSDValue(_priceCache, address(debtToken), debtTokenAmount) * borrowingInterestRate) /
          DECIMAL_PRECISION) * timePassed) /
        SECONDS_PER_YEAR;
```

There is no particular risk of overflow and the loss is up to `DECIMAL_PRECISION - 1`

**Mitigation**

Change the formula to:

```solidity
      stableInterest +=
        (((_priceFeed.getUSDValue(_priceCache, address(debtToken), debtTokenAmount) * borrowingInterestRate) * timePassed) / DECIMAL_PRECISION /
        SECONDS_PER_YEAR;
```

### [L-10] Bad Debt Redistribution can be avoided by removing collaterals

**Impact**

The logic for `redistributeDebtAndColl` is as follows

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/TroveManager.sol#L310-L337

```solidity
  function redistributeDebtAndColl(PriceCache memory _priceCache, CAmount[] memory toRedistribute) external override {
    _requireCallerIsBorrowerOpsOrRedemptionOpsOrLiquidationOps();

    // sum up all coll usd values
    uint totalRedistributedCollInUsd;
    uint[] memory collCacheInUSD = new uint[](toRedistribute.length);
    for (uint i = 0; i < toRedistribute.length; i++) {
      CAmount memory collEntry = toRedistribute[i];
      if (!collEntry.isColl || collEntry.amount == 0) continue; /// collCacheInUSD[i] = collInUSD || 0 when skipped

      uint collInUSD = priceFeed.getUSDValue(_priceCache, collEntry.tokenAddress, collEntry.amount);
      collCacheInUSD[i] = collInUSD;
      totalRedistributedCollInUsd += collInUSD;
    }

    // iterate over the coll entries and process the debt relative to the coll percentage
    uint[] memory debtsToDefaultPool = new uint[](toRedistribute.length);
    for (uint i = 0; i < toRedistribute.length; i++) { /// @audit technically same loop as above, since it skips the same entries
      CAmount memory collEntry = toRedistribute[i];
      if (!collEntry.isColl || collEntry.amount == 0) continue;

      // patch the liquidated coll tokens
      uint collTotalStake = totalStakes[collEntry.tokenAddress]; /// @audit Can you have 0 stakes by having a Trove that didn't redistribute?
      PoolType targetPool;
      if (collTotalStake == 0) {
        // the last trove with that coll type was liquidated/close, moving the assets into a claimable (unassigned) pool
        targetPool = PoolType.Unassigned;
      } else {
```

Specifically when `collTotalStake` is zero a redistribution for that collateral is skipped, sending debt and collateral to the `Unassigned` pool

This means that some users will be able to skip the redistribution by re-organizing their collateral before performing liquidations

It's worth noting that liquidations present a race condition against this behaviour meaning this can theoretically happen, but in some scenarios, an attacker won't be able to pull it off unless they are performing the liquidations as well

### [L-11] Stablecoin interest being lower than jTokens seems inconsistent + Tokens with different volatilities pay the same fee

**Impact**

Generally speaking jAssets will be more volatile and riskier than a stablecoin that denominates them

The logic for `getBorrowingRate` charges more for stableCoins than for jAssets

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/TroveManager.sol#L898-L901

```solidity
  function getBorrowingRate(bool isStableCoin) public view override returns (uint) {
    if (!isStableCoin) return borrowingFeeFloor;
    return _calcBorrowingRate(stableCoinBaseRate);
  }
```

It's also worth noting that all assets pay the same interest rate which means that they don't pay based on risk

**Mitigation**

Consider whether you should charge different fees for different assets so that the system is compensated for the additional risk it's taking

### [L-12] `SwapOperation` First LP pays no fee and can set the price to an incorrect value, causing losses to traders, and higher fees

**Impact**

`addLiquidity` looks as follows

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L228-L246

```solidity
  function addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin,
    PriceUpdateAndMintMeta memory _priceAndMintMeta,
    uint deadline
  ) public payable virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
    ProvidingVars memory vars;
    vars.pair = getPair[tokenA][tokenB]; /// @audit QA not sorting tokens means this can revert | No fix is ok but it's a gotcha
    if (vars.pair == address(0)) revert PairDoesNotExist();
    /// @audit not checking that min < desired - S
    {
      (vars.reserveA, vars.reserveB) = getReserves(tokenA, tokenB);
      if (vars.reserveA == 0 && vars.reserveB == 0) {
        (amountA, amountB) = (amountADesired, amountBDesired); /// @audit First LP Can attack by massively imbalancing by performing a single sided swap or similar
      } else {
```

When no liquidity was previously added, the ratio for the LP will be taken at face value

This could be used by the first depositor to purposefully imbalance the pool, as a means to take on debts that either allow them to be closer to delta neutral, or as a means to grief other users

**Mitigation**

Consider using a ratio that is taken from the oracles

### [L-13] `SwapOperations` Swap Fees may add up to more than 100%

**Impact**

`swapBaseFee` is capped to 1e18, but it is added to `Pair.getSwapFee` meaning that it may result in a value higher than 100%



**Mitigation**

Cap fees at a smaller value, such as at most 10%

### [L-14] `claimUnassigned` may result in a slight difference between debt and coll percentages being claimed due to rounding errors

**Impact**

Slight difference in debt and coll claimed due to rounding error

May be abused to get free coll due to coll rounding up to a unit or more and debts rounding down to 0

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L630-L637

```solidity
    DebtTokenAmount[] memory debtsToAdd = new DebtTokenAmount[](vars.priceCache.debtPrices.length);
    for (uint i = 0; i < vars.priceCache.debtPrices.length; i++) {
      address debtToken = vars.priceCache.debtPrices[i].tokenAddress;

      uint unassignedDebt = contractsCache.storagePool.getValue(debtToken, false, PoolType.Unassigned);
      if (unassignedDebt == 0) continue;

      uint toClaim = (unassignedDebt * _percentage) / DECIMAL_PRECISION;
```

**Mitigation**

You may want to at least require that if collateral is 1, then debt is at least 1 as well

You could also add explicit rounding up of debt, and rounding down of collateral

Please keep in mind that this could introduce more bugs so this is a delicate change

### [L-15] `claimUnassignedAsset` `_percentage` can be more than 1e18

**Impact**

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L614-L620

```solidity
  function claimUnassignedAssets(
    uint _percentage,
    address _upperHint,
    address _lowerHint,
    bytes[] memory _priceUpdateData
  ) external payable override {
    if (_percentage == 0) revert ZeroDebtChange();
```

Maybe abused for exploit, I haven't spent a lot of time on this, it's best to cap it to 1e18 to avoid any additional risk

**Mitigation** 

```solidity
if (_percentage > DECIMAL_PRECISION) revert Above100Pct();
```

### [L-16] Basic Style Guide advice

**Impact**

The impact from the current formatting is that manual review takes longer, and as such will be more expensive and error prone

**Specific Code Smells**

- Not using curly braces for `if / else`
- Using `ii` in favour of mathematical notation `n, l, m` `i, j, k`, `x, y, z`
- Breaking `CEI`
- Ping ponging of external calls
- Subtle changes against Liquity's version


**Breaking CEI**

`openTrove` shows an example of a seemingly innocuous way to break CEI

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L199-L224

```solidity
    vars.arrayIndex = contractsCache.troveManager.addTroveOwnerToArray(borrower);
    /// @audit move the transfer of coll to the bottom to avoid CEI issues
    // Move the coll to the active pool
    for (uint i = 0; i < vars.colls.length; i++) {
      TokenAmount memory collTokenAmount = vars.colls[i]; /// @audit CEI / Reentrancy
      _poolAddColl(
        borrower,
        contractsCache.storagePool,
        collTokenAmount.tokenAddress,
        collTokenAmount.amount,
        PoolType.Active
      );
    }
    /// @audit Where is the trove? 
    // Move the stable coin gas compensation to the Gas Pool
    contractsCache.storagePool.addValue(
      address(stableCoinAmount.debtToken),
      false,
      PoolType.GasCompensation,
      stableCoinAmount.netDebt
    ); /// @audit Not minting principal on open
    stableCoinAmount.debtToken.mint(address(contractsCache.storagePool), stableCoinAmount.netDebt);

    emit TroveCreated(borrower, _colls);
  }

```


With `_poolAddColl` looking as follows

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L840-L851

```solidity

  function _poolAddColl(
    address _borrower,
    IStoragePool _pool,
    address _collAddress,
    uint _amount,
    PoolType _poolType
  ) internal {
    _pool.addValue(_collAddress, true, _poolType, _amount);
    IERC20(_collAddress).transferFrom(_borrower, address(_pool), _amount); /// @audit FOT / SafeTransfer
  }

```

This would make it so that the system is recording the increase in collateral, but not the increase in debt, which would allow to drag the TCR below the Recovery Mode threshold, and liquidate all Troves that have a ICR < CCR

Liquity like systems tend to blur the idea of an external call as they effectively rely on external calls for accounting, but those can be viewed as trusted external calls

Whereas moving tokens should be viewed as an untrusted external call under the vast majority of circumnstances

You should refactor not to break CEI as it will:
- Ensure you cannot introduce exploits tied to ordering of external calls
- Make future review cheaper as CEI concerns will not take a considerable amount of time for review

### [L-17] `_getCurrentPythResponse` can benefit by having more validation

**Impact**

Pyth works as follows:
- It returns an integer
- It returns an exponent by which to scale the integer for the price
- it returns the publishTime
- and a confidence interval

The reason why Pyth includes a confidence interval is due to the impossibility of chosing a single price that is the "correct" price

**Mitigation**

Consider implementing additional checks, you could take inspiration from Euler's multi-audited feeds:
https://github.com/euler-xyz/euler-price-oracle/blob/master/src/adapter/pyth/PythOracle.sol

### [L-18] WIP - Invariants 

**Troves should never break the check `_debtTokenUsedAsCollRatio(contractsCache, vars) > MAX_DEBTS_AS_COLLATERAL`**

**There should be n Trove with 0 net debt in the Sorted Trove**

**At no time I can perform an operation, then update the oracle, and trigger recover mode**

**At no time, I should be able to open a Trove, and liquidate it in the same block**

**I should not be able to skip Bad Debt Redistributions**

**Trove Manager**

**Post accrual `getTroveColl` is == pre-accrual `getTroveWithdrawableColls`**

**Liquidation Operation**

**Liquidations should raise the TCR of the system, unless the liquidation is done on a Trove with Bad Debt**

**This line should never execute (as the logic above doesn't allow it)**

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/LiquidationOperations.sol#L352-L353

```solidity
      if (collToLiquidate > rAmount.toLiquidate) collToLiquidate = rAmount.toLiquidate; // in case of IMCR > ICR, but the trove still gets liquidated because ICR < TCR in recovery mode

```

### [L-19] `RedeptionOperations.checkValidRedemptionHint` check should use `>=`

**Impact**

The check in `checkValidRedemptionHint` for the current hint is as follows:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/RedemptionOperations.sol#L212

```solidity
if (hintCR < hintIMCR) revert HintBelowMCR(); // should be liquidated, not redeemed from
```

Which asserts that the hint is not liquidatable outside of recovery mode (which is checked in `redeemCollateral`)

The check below is to ensure that the trove being redeemed is the riskiest trove that is not underwater

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/RedemptionOperations.sol#L217-L218

```solidity
    if (nextTrove != address(0) && nextTroveCR > nextTroveMCR) revert InvalidHintLowerCRExists(); /// @audit TO CHECK

```

The check is the opposite of the above, therefore the comparison: `nextTroveCR > nextTroveMCR` should be `nextTroveCR >= nextTroveMCR`

**Mitigation**

Change `nextTroveCR > nextTroveMCR` to `nextTroveCR >= nextTroveMCR`


### [L-20] `enableLiquidationAndRedeeming` pauses liquidations which can be problematic

**Impact**

`enableLiquidationAndRedeeming` is pausing both redemptions and liquidations

Those 2 operations have a very different purpose, with redemptions helping with the peg and acting as a pre-liquidation and liquidations being a necessary tool to make the system work

Due to this, liquidations should never be disabled unless a major bug was discovered

**Mitigation**

Separate `enableLiquidationAndRedeeming` to disable redemptions and liquidations separately



### [L-21] `SwapPair.getSwapFee` charges for crossing the middle price

**Impact**

`getSwapFee` computes the fee to be paid as follows:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L196-L199

```solidity
      if ( /// @audit No deviation threshold | Spot vs Oracle
        (postDexPrice > oraclePrice && postDexPrice > preDexPrice) || /// @audit Logical mechanism of swapping?
        (postDexPrice < oraclePrice && preDexPrice > postDexPrice)
      ) {
```

We can chart out the logic as follows:

- Post  > Oracle
- Post > Prev > Oracle
- Post > Oracle > Prev

- Post < Oracle
- Post < Prev < Oracle 
- Post < Oracle < Prev

Meaning this is charging a fee whenever the absolute postDexPrice surpasses the absolute oracle price

So when crossing the impact is "halved"

When not crossing the middle price, the average between the two dex prices will be very distant from the oracle price, causing the fee to be higher, whereas when crossing the fee would effectively be halved

**Mitigation**

I don't believe there's any need for a specific mitigation

### [L-22] Incompatibility with tokens that charge a fee on transfer

**Impact**

The system stores balances in storage and then updates them

Some tokens will charge a fee on transfer

Meaning that `_amount` stored in the `StoragePool` will be higher than the actual balance received

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L841-L851

```solidity
  function _poolAddColl(
    address _borrower,
    IStoragePool _pool,
    address _collAddress,
    uint _amount,
    PoolType _poolType
  ) internal {
    _pool.addValue(_collAddress, true, _poolType, _amount);
    IERC20(_collAddress).transferFrom(_borrower, address(_pool), _amount); /// @audit FOT / SafeTransfer
  }

```

These types of tokens are pretty rare, but this is a very common finding that you should think about

**Mitigation**

Imo acknowledge this and make sure not to use these tokens

### [L-23] Analysis - Some thoughts, consideration and advice for next steps

Logically you can chunk the code into 3 components

- Feeds
- Swap and Stake Operations
- Core

-----

It may be best to write fuzz and invariant tests on these separately and then you can consider adding a new 

**Generic Gas Advice**

**Use Immutables**

The codebase relies on caching storage values into memory to reduce SLOADs

You could also deterministically deploy each contract (since contract addresses are derived from address + nonce) and set those up as immutable variables



**Alex's Review Status**

- [x] AlternativePriceFeed
- [x] BorrowerOperations
- [x] CollSurplusPool
- [X] DebtToken*
- [X] HintHelpers*
- [x] LiquidationOperations
- [x] PriceFeed
- [x] RedemptionOperations
- [x] ReservePool
- [x] SortedTroves
- [x] StabilityPool
- [x] StabilityPoolManager
- [x] StakingOperations
- [x] StoragePool
- [x] SwapERC20
- [X] SwapOperations
- [x] SwapPair
- [x] TokenManager
- [X] TroveManager

**Breaking CEI**

CEI stands for `Check Effects Interactions` which ensures that no state changing operations are done after external calls that the system may rely on

Transferring collateral before updating internals should be considered as highly risky, not just in lack of `nonReentrant` guards but in general due to the system reliance on cross contract operations

In my opinion all CEI concerns should be addressed, unless you can ensure that no token will ever have hooks

But in general, solving CEI even in those cases is a better choice as it will save time from auditors looking into those concerns and will ensure that the system is safe even if a token with hooks were to be introduced

**Price Staleness**

Relative staleness is imo not a valid idea, all prices must be validated and trove status must be checked against the latest available prices

In lack of that, a Trove could end up generating bad debt due to having it's CR computed with stale prices


**Alternative Price Feed**

Fundamentally it hardcodes prices, offers a guarantee for staleness

No guarantee for deviation threshold (how accurate, and how fast it will react to sudden price changes)

An hardcoded price should be viewed as a massive liability and risk, it's a fundamental issue for developers as well as for end users

It may be best to have 2 prices:
- A borrow price 
- A liquidation price

Fundamentally making the assets:
- Less valuable for borrowing (So you can borrow less)
- Overpriced for liquidations (so liquidations happen faster)


**StakingOperations**

This contract is effectively a Masterchef and can be reviewed fairly independently

Race conditions around interactions with the pool are pretty delicate

Beside that, the code is separated so this contract can be re-reviewed independently

**SwapPair**

Fundamentally there are ways to sidestep the intended `_requireCallerIsOperations` via direct transfer + sync

The added fee is the most delicate aspect and I believe tighter checks to ensure that only an intended amount is paid should be added

The x * y = k formula is highly capital inefficient in this case but that's not something easily fixable

I think the way `_getDexPrice` is calculate is a bit naive as this is not a price but rather the ratio of reserves any purchase will incur an additional price impact that is not accounted there, additionally taking the average of the before and after makes it so that some swaps that push the price out of the oracle price do not pay a sufficiently high fee


**SwapOperations**

- Donation to force a ratio
- Fee math / Path Math

**TokenManager**

Fundamentally some small risks tied to token config

The biggest risk is tied to changing risk config params

**CollSurplusPool**

The observations around `CollSurplusPool` are:
- Double loop vs mapping, this is generally a pattern you should avoid
- Breaking CEI - you should consider CEI a key part of your security posture, breaking CEI should not be underestimated as a "basic mistake"

**StoragePool**

The main risks around `storagePool` are tied to a desynchronization between the value that it tracks, and the values being tracked in other contracts, such as `TroveManager`

I believe invariant testing should be added to ensure that these values are synchronized at all times 

**DebtToken**

Fundamentally just tracks the debt and is also used as a scaling factor for oracle

I believe that the logic for Stock Splits can be massively simplified by hardcoding the divisor and the dividend, with no particular change in risk profile

**HintHelpers**

The main risk I see is how the SortedTroves uses cached values, but HH is using live values

The discrepancy may be used to trigger Recovery Mode via Redemptions

**SortedTroves**

The main changes are tied to how data is stored as well as the removal of insert / remove, in favour of an update function

The sorting is done via cached CR, but that's not a distinctive change as ultimately even Liquidity is just sorting based on a CR value that is accepted at face value

This seems to be a good candidate for differential fuzzing to compare the two implementations

To be honest I'm not sure why the implementation was changed as this ultimately behaves very similarly

**Swap Operations**

The main risks are tied to manipulating the debt being taken by users, as well as possible race conditions
For these operations, oracle staleness is also an important factor

I believe these contracts can be mostly reviewed separately in the future as they feel pretty separated

If you can end up separating the debts math from swaps, I think these contracts can be fully separated long term

**StabilityPool**

CEI concerns, code is fairly complex but is pretty much the same logic as in Liquity

**Stability Pool Manager**

Fundamentally tracks accounting, for liquidation, which looks fine

The main aspect that I'm not fully convinced by is the usage of the Reserve Pool, which seems not be taken into account by the LiquidationOperations

**Redemption Operations**

Round downs seem to be in favour of the protocol which doesn't seem to create a risk

Due to the fact that Troves are sorted by cached CRs, some Troves "in the middle" may be underwater, meaning that redeeming them will revert, this will cause reverts but it shouldn't cause major issues as you could simply perform a redemption, then another one, etc..



**Recommended Next Steps**

I recommend spending quite some time into the economic and technical implications that come with:
- Oracle staleness
- Redemptions, Liquidations and Swap Fees

I also recommend adding a few invariant tests for system wide invariants that would guarantee:
- Intended execution
- Inability to attack the system (self-liquidations, triggering RM)
- Ability to execute operations when they are necessary (e.g. liquidations)

Some suggested invariants are listed here:
https://github.com/GalloDaSballo/Apollon-Review/issues/44

### [L-24] `SwapOperations.addLiquidity` is not validating desired and min amounts

**Impact**

`addLiquidity` bases it's logic on the implicit invariant that `amountDesired` are greater than `minAmounts`

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L242-L258

```solidity
    {
      (vars.reserveA, vars.reserveB) = getReserves(tokenA, tokenB);
      if (vars.reserveA == 0 && vars.reserveB == 0) {
        (amountA, amountB) = (amountADesired, amountBDesired); /// @audit First LP Can attack by massively imbalancing by performing a single sided swap or similar
      } else {
        uint amountBOptimal = quote(amountADesired, vars.reserveA, vars.reserveB);
        if (amountBOptimal <= amountBDesired) {
          if (amountBOptimal < amountBMin) revert InsufficientBAmount();
          (amountA, amountB) = (amountADesired, amountBOptimal);
        } else {
          uint amountAOptimal = quote(amountBDesired, vars.reserveB, vars.reserveA);
          assert(amountAOptimal <= amountADesired); /// @audit Looks wrong
          if (amountAOptimal < amountAMin) revert InsufficientAAmount();
          (amountA, amountB) = (amountAOptimal, amountBDesired);
        }
      }
    }
```

However this check is missing from `addLiquidity`

**Mitigation**

Add the following checks

```solidity
        if (amountADesired < amountAMin) revert InsufficientAmountADesired();
        if (amountBDesired < amountBMin) revert InsufficientAmountBDesired();
```

Also see: [Velodrome Router](https://github.com/velodrome-finance/contracts/blob/9e5a5748c3e2bcef7016cc4194ce9758f880153f/contracts/Router.sol#L172-L182)

### [L-25] `SwapOperations` computes the swap fees without accounting for how fees will alter reserves

**Impact**

At a broadstroke The formula to compute the post-swap reserves used by `getAmountsOut` and `getAmountsIn` is as follows:

```
uint(reserveA) * reserveB) / (reserveB + amtB),
```

However, the math is not accounting for fees that will be taken on each swap, meaning that the post-swap reserves will not match this amount


**Mitigation**

You should technically also account for the updated price post taking fees, as that will be the price that the next person will pay

I don't have sufficient time to fully explore this issue and I highly recommend you simulate swaps to determine if this can cause significant losses



### [L-26] `ERC20.Permit` can be front-runned and should be try/catched

**Impact**

Permit can be broadcasted by anyone and will revert if using the incorrect nonce

Due to this you should use try/catch it

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L225-L245

```solidity
  function openTroveWithPermit(
    TokenAmount[] memory _colls,
    bytes[] memory _priceUpdateData,
    uint deadline,
    uint8[] memory v,
    bytes32[] memory r,
    bytes32[] memory s
  ) external payable {
    for (uint i = 0; i < _colls.length; i++)
      IERC20Permit(_colls[i].tokenAddress).permit( /// @audit Should use Permit
        msg.sender,
        address(this),
        _colls[i].amount,
        deadline,
        v[i],
        r[i],
        s[i]
      );

    openTrove(_colls, _priceUpdateData);
  }
```

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L317-L318

```solidity
    IERC20Permit(tokenA).permit(msg.sender, address(this), amountADesired, deadline, v[0], r[0], s[0]);
    IERC20Permit(tokenB).permit(msg.sender, address(this), amountBDesired, deadline, v[1], r[1], s[1]);
```

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L511

```solidity
IERC20Permit(path[0]).permit(msg.sender, address(this), amountIn, deadline, v, r, s);
```

See this article for more info:
https://www.trust-security.xyz/post/permission-denied

### [L-27] `SwapPair` events use `msg.sender` but they should use the user taking on debt

**Impact**

Events in SwapPair are logging `msg.sender` which will always be `swapOperations`

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L125-L126

```solidity
    emit Mint(msg.sender, amount0, amount1); /// @audit QA: Mint should change to `to`

```

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L169

```solidity
emit Burn(msg.sender, amount0, amount1, to); /// @audit QA: Mint should change to `to` or to the cdp Owner
```

**Mitigation**

Either change the log to `to` (the recipient), or add a `initiator` parameter and log that one

Alternatively, move the events in the `SwapOperations`

### [L-28] `SwapPair` may revert on 0 transfer tokens

**Impact**
Certain tokens revert on a 0 amount transfer

The code also performs the check to prevent zero transfers in many parts, but in these 2 instances the check was missed

[Note that for OZ tokens, the check is not necessary](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/c304b6710b4b5fcf2a319ad28c36c49df6caef14/contracts/token/ERC20/ERC20.sol#L183-L211)

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L234-L238

```solidity
    if (amount0InFee > 0) {
      uint amount0GovFee = (amount0InFee * swapOperations.getGovSwapFee()) / DECIMAL_PRECISION;
      _safeTransfer(token0, tokenManager.govPayoutAddress(), amount0GovFee); /// @audit 0 revert
      balance0 -= amount0GovFee;
    } 
```

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L161-L163

```solidity
    // payout whats left
    _safeTransfer(token0, to, amount0 - burned0);
    _safeTransfer(token1, to, amount1 - burned1);
```

**Mitigation**

Check that each transfer is done only on non-zero amounts

### [L-29] `SwapPair` is intended to overflow, but is not using `unchecked`

**Overflow logic from UniV2 no longer applies**

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L84-L93

```solidity
  function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
    if (balance0 > type(uint112).max || balance1 > type(uint112).max) revert Overflow();

    uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
    uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
      // * never overflows, and + overflow is desired | /// @audit Overflow is wrong, this is solidity 0.8!!
      price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
      price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
    }
```

The code asserts that `+= overflow is desired` but this cannot happen on the current solidity version
This means that after (I think) more than a billion of volume, the pair can stop working

I can crunch the math, but the overflow is extremely unlikely

See: Velodrome (Which also ignores the overflow):
https://github.com/velodrome-finance/contracts/blob/9e5a5748c3e2bcef7016cc4194ce9758f880153f/contracts/Pool.sol#L212-L230

**Reserve overflow checks using `uint112` effectively cap any token max total supply**

The maximum amount possible is 2^122 - 1 

Which is basically `5.1922969e+15 * 1e18`

This should be sufficient, but it's worth keeping it in mind as you do not want to cross this value as it will break all pool operations for a pair

**Overflow exactly at time 2^32**

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L87-L88

```solidity
    uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);

```

This line being unchecked means the code will stop working at the timestamp 2^32

Which should be around Sun Feb 07 2106 06:28:16 GMT+0000

```solidity
    // forge test --match-test test_theOverflow -vv
    function test_theOverflow() public {
        vm.warp(type(uint32).max + 1);
        uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);

    }
```

### [L-30] Donation on Pairs, in conjunction with lax slippage could be used to take on unintended ratios of debt

**Executive Summary**

Pair.synch allows synching without a swap being performed

This is a donation to the Pair reserves, which changes it's price

**Impact**

When providing liquidity the ratio of the two tokens is taken into account

Via donation attacks we can force people to take on incorrect ratios of debt

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L228-L302

```solidity
  function addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin,
    PriceUpdateAndMintMeta memory _priceAndMintMeta,
    uint deadline
  ) public payable virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
    ProvidingVars memory vars;
    vars.pair = getPair[tokenA][tokenB]; /// @audit QA not sorting tokens means this can revert | No fix is ok but it's a gotcha
    if (vars.pair == address(0)) revert PairDoesNotExist();
    /// @audit not checking that min < desired - S
    {
      (vars.reserveA, vars.reserveB) = getReserves(tokenA, tokenB);
      if (vars.reserveA == 0 && vars.reserveB == 0) {
        (amountA, amountB) = (amountADesired, amountBDesired); /// @audit First LP Can attack by massively imbalancing by performing a single sided swap or similar
      } else {
        uint amountBOptimal = quote(amountADesired, vars.reserveA, vars.reserveB);
        if (amountBOptimal <= amountBDesired) {
          if (amountBOptimal < amountBMin) revert InsufficientBAmount();
          (amountA, amountB) = (amountADesired, amountBOptimal);
        } else {
          uint amountAOptimal = quote(amountBDesired, vars.reserveB, vars.reserveA);
          assert(amountAOptimal <= amountADesired); /// @audit Looks wrong
          if (amountAOptimal < amountAMin) revert InsufficientAAmount();
          (amountA, amountB) = (amountAOptimal, amountBDesired);
        }
      }
    }
    // AmountA and B = Amts after optimal math

    vars.senderBalanceA = IERC20(tokenA).balanceOf(msg.sender);
    vars.senderBalanceB = IERC20(tokenB).balanceOf(msg.sender);
    // Pick Min between available and optimal
    vars.fromBalanceA = LiquityMath._min(vars.senderBalanceA, amountA); /// @audit WHY?
    vars.fromBalanceB = LiquityMath._min(vars.senderBalanceB, amountB);
    // Optimal - From Balance
    vars.fromMintA = amountA - vars.fromBalanceA;
    vars.fromMintB = amountB - vars.fromBalanceB;

    // mint new tokens if the sender did not have enough
    if (vars.fromMintA != 0 || vars.fromMintB != 0) {
      TokenAmount[] memory debtsToMint;
      if (vars.fromMintA != 0 && vars.fromMintB != 0) {
        // mint both
        debtsToMint = new TokenAmount[](2);
        debtsToMint[0] = TokenAmount(tokenA, vars.fromMintA);
        debtsToMint[1] = TokenAmount(tokenB, vars.fromMintB);
      } else {
        // mint only 1 token
        debtsToMint = new TokenAmount[](1);
        debtsToMint[0] = (
          vars.fromMintA != 0
            ? TokenAmount(tokenA, vars.fromMintA) // mint A
            : TokenAmount(tokenB, vars.fromMintB) // mint B
        );
      }
      borrowerOperations.increaseDebt{ value: msg.value }(
        msg.sender, // OK 
        vars.pair, // OK
        debtsToMint, // OK - See above
        _priceAndMintMeta.meta, /// @audit this looks griefable | TODO
        _priceAndMintMeta.priceUpdateData // TODO
      );
    }

    // transfer tokens sourced from senders balance
    if (vars.fromBalanceA != 0) safeTransferFrom(tokenA, msg.sender, vars.pair, vars.fromBalanceA);
    if (vars.fromBalanceB != 0) safeTransferFrom(tokenB, msg.sender, vars.pair, vars.fromBalanceB);

    // deposit into staking
    liquidity = ISwapPair(vars.pair).mint(msg.sender); /// @audit-ok this doesn't send tokens, it credits the stakingPool Deposit to msg.sender
  }
```

This can be abused to cause losses whenever the slippage checks are not set tight enough


**Mitigation**

Setting highly accurate slippage settings will prevent this

### [L-31] Incorrect encoding for Permit

**Impact**

```solidity
  function permit(
    address owner,
    address spender,
    uint amount,
    uint deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
  ) external override {
    if (deadline < block.timestamp) revert ExpiredDeadline();
    bytes32 digest = keccak256(
      abi.encodePacked(
        '\x19\x01',
        this.domainSeparator(),
        keccak256(abi.encode(_PERMIT_TYPEHASH, owner, spender, amount, _nonces[owner]++, deadline))
      )
    );

    bytes32 signedMsg = MessageHashUtils.toEthSignedMessageHash(digest); /// @audit this is not necessary /  incorrect, it's a double encoding
    address recoveredAddress = ECDSA.recover(signedMsg, v, r, s); /// @audit QA: This is not Permit EIP!
    if (recoveredAddress != owner) revert InvalidSignature();
    _approve(owner, spender, amount);
  }
```

The additional `MessageHashUtils.toEthSignedMessageHash(digest);` is not necessary and is re-encoding the user signature

Per the EIP, you're supposed to simply sign on the digest
https://eips.ethereum.org/EIPS/eip-2612

Also check this implementation of a JS signature library:
https://github.com/dmihal/eth-permit/blob/34f3fb59f0e32d8c19933184f5a7121ee125d0a5/src/rpc.ts#L59-L71

As you can see the signature is done on the digest, the additional encoding you're doing would require re-encoding with ethSignedMessage on the encoded digest

Even when relying on RPC the method is `eth_signTypedData_v4`
https://github.com/dmihal/eth-permit/blob/34f3fb59f0e32d8c19933184f5a7121ee125d0a5/src/rpc.ts#L73-L93


See: https://docs.metamask.io/wallet/reference/eth_signtypeddata_v4/

**Mitigation**


Change the code to compare the signature against the digest

```solidity
  function permit(
    address owner,
    address spender,
    uint amount,
    uint deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
  ) external override {
    if (deadline < block.timestamp) revert ExpiredDeadline();
    bytes32 digest = keccak256(
      abi.encodePacked(
        '\x19\x01',
        this.domainSeparator(),
        keccak256(abi.encode(_PERMIT_TYPEHASH, owner, spender, amount, _nonces[owner]++, deadline))
      )
    );

    address recoveredAddress = ECDSA.recover(digest, v, r, s);
    if (recoveredAddress != owner) revert InvalidSignature();
    _approve(owner, spender, amount);
  }
```


