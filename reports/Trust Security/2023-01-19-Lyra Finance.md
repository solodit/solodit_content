**Auditors**

[Trust Security](https://twitter.com/trust__90)


---


# Findings

## High Risk
### TRST-H-1 canHedge may return wrong result when there is a pending position request
**Description:**
Lyra’s security model relies on being able to hedge and achieve delta-neutrality when opening 
a user position. The check is done in `canHedge()` in GMXFuturesPoolHedger. The current 
hedge is calculated using `_getCurrentHedgedNetDeltaWithSpot()`. However, this function only 
takes the current position and ignores the pending increase/decrease position request. 
Therefore, `canHedge` result can be wrong - users may be rejected from interacting with the 
market, while attackers or innocent users may put the protocol in an unchangeable position.

**Recommended Mitigation:**
Including the pending position in the current hedge calculation.

**Team response:**
While this is a valid issue, `canHedge` is an added safety rail rather than a critical component 
of the system. Unwanted option positions being opened to expose LPs to unwanted delta risk 
will cost attackers the fees to open option positions and all they achieve is exposing LPs to 
some limited directional/delta risk. Adding additional complexity to an already complex 
system feels unnecessary at this stage so this will not be implemented.



### TRST-H-2 canHedge will return true when hedging requirement would be above the defined hedgeCap
**Description:**
Lyra’s security model relies on being able to hedge and achieve delta-neutrality when opening 
a user position. The check is done in `canHedge()` in GMXFuturesPoolHedger. The expected 
hedge is fetched using `_getCappedExpectedHedge()`. However, it is never checked that the 
hedge has reached capacity, which should disqualify the hedge from taking place. Any hedge 
above the cap will never be hedged, so whenever `canHedge()` wrongly approves a hedge that 
is beyond the cap, the protocol will be guaranteed not to be delta-neutral.

**Recommended Mitigation:**
If **expectedHedge** is, in absolute value, equal to the **hedgeCap**, return false

**Team Response:**
This is more a design choice than an issue. To account for all cases (as in the flagged issue) an 
additional parameter would need to be added to allow opening above the cap which is the 
current intended design.


### TRST-H-3 disordered fee calculated causes collateral changes to be inaccurate
**Description:**
`_increasePosition()` changes the Hedger’s GMX position by **sizeDelta** amount and 
**collateralDelta** collateral. There are two **collateralDelta** corrections - one for swap fees and 
one for position fees. Since the swap fee depends on up-to-date **collateralDelta**, it’s important 
to calculate it after the position fee, contrary to the current state. In practice, it may lead to 
the leverage ratio being higher than intended as **collateralDelta** sent to GMX is lower than it 
should be.
```solidity
      if (isLong) {
          uint swapFeeBP = getSwapFeeBP(isLong, true, collateralDelta);
           collateralDelta = (collateralDelta * (BASIS_POINTS_DIVISOR + swapFeeBP)) / BASIS_POINTS_DIVISOR;
      }
      // add margin fee
      // when we increase position, fee always got deducted from collateral
          collateralDelta += _getPositionFee(currentPos.size, sizeDelta, currentPos.entryFundingRate);
``` 

**Recommended Mitigation:**
Flip the order of `getSwapFeeBP()` and `_getPositionFee()`. 

**Team response:**
Fixed


## Medium Risk
### TRST-M-1 small LP providers may be unable to withdraw their deposits
**Description:**
In LiquidityPool’s initiateWithdraw(), it’s required that withdrawn value is above a minimum 
parameter, or that withdrawn tokens is above the minimum parameter.
```solidity 
      if (withdrawalValue < lpParams.minDepositWithdraw && 
          amountLiquidityToken < lpParams.minDepositWithdraw) {
      revert MinimumWithdrawNotMet(address(this), withdrawalValue, lpParams.minDepositWithdraw);
      }
```
The issue is that **minDepositWithdraw** is measured in dollars while **amountLiquidityToken** is 
LP tokens. The intention was that if LP tokens lost value and a previous deposit is now worth 
less than **minDepositWithdraw**, it would still be withdrawable. However, the current 
implementation doesn’t check for that correctly, since the LP to dollar exchange rate at 
deposit time is not known, and is practically being hardcoded as 1:1 here. The impact is that 
users may not be able to withdraw LP with the token amount that was above the minimum at 
deposit time, or vice versa

**Recommended Mitigation:**
Consider calculating an average exchange rate at which users have minted and use it to verify 
withdrawal amount is satisfactory.

**Team Response:**
While valid, the proposed solution adds far more complexity to the system than the benefit it 
would provide. Small (<$1) LPs will need to find an alternative place to liquidate their holdings 
like a uniswap pool. This will not be resolved at the protocol level.
As keepers process deposits and withdrawals, the minimums are necessary to prevent 
unwanted spam.


### TRST-M-2 ShortCollateral settleOptions may fail due to insolvency settled out of loop
**Description:**
settleOptions() loops over the input **positionIds** array and either sends proceeds to the user 
or consumes their collateral. If there’s any insolvency, it is appended to 
**baseInsolventAmount/quoteInsolventAmount**. Only after the settlement loop is the entire 
insolvent portion claimed from the liquidity pool. The issue is that delaying the insolvency 
collection may cause ShortCollateral to have insufficient funds to pay for user proceeds.

**Recommended Mitigation:**
Insert `_reclaimInsolvency()` call to the end of the for loop.

**Team Response:**
Not really an issue if insolvent positions are settled in a separate transaction prior to solvent 
positions. Keepers can easily handle this situation. As insolvent positions are very rare (0 in all 
of the 6 months of the Avalon release) no changes to this logic seem appropriate.


### TRST-M-3 base to quote swaps trust GMX-provided minPrice and maxPrice to be correct, which may be manipulated
**Description:**
exchangeFromExactBase() in GMXAdapter converts an amount of base to quote. It 
implements slippage protection by using the GMX vault’s getMinPrice() and getMaxPrice() 
utilities. However, such protection is insufficient because GMX prices may be manipulated. 
Indeed, GMX supports “AMM pricing” mode where quotes are calculated from Uniswap 
reserves. A possible attack would be to drive up the base token (e.g. ETH) price, sell a large 
ETH amount to the GMXAdapter, and repay the flashloan used for manipulation. 
exchangeFromExactBase() is attacker-reachable from LiquidityPool’s exchangeBase().
```solidity
    uint tokenInPrice = _getMinPrice(address(baseAsset));
        uint tokenOutPrice = _getMaxPrice(address(quoteAsset));
    ...
    uint minOut = tokenInPrice
      .multiplyDecimal(marketPricingParams[_optionMarket].minReturnPercent)
        .multiplyDecimal(_amountBase)
          .divideDecimal(tokenOutPrice);
```

**Recommended Mitigation:**
Verify `getMinPrice()`, `getMinPrice()` outputs are close to Chainlink-provided prices as done in 
`getSpotPriceForMarket()`.

**Team Response:**
Fixed for `exchangeFromExactBase()` here, by using Chainlink price instead of **gmxMinPrice** of 
**baseAsset**. This way if the price is favorable for the LPs (given they rely on CL) it will not revert.


### TRST-M-4 sendAllFundsToLP() does not handle popular ERC20 tokens like BNB
**Description:**
sendAllFundsToLP() is used to transfer quote and base tokens to the LP after interaction with 
GMX. It uses an unsafe transfer call:
```solidity
    if (baseBal > 0) {
       if (!baseAsset.transfer(address(liquidityPool), baseBal)) {
    revert AssetTransferFailed(address(this), baseAsset, baseBal, 
         address(liquidityPool));
     }
    emit BaseReturnedToLP(baseBal);
```
There are a great many tokens such as BNB and USDT that for historical reasons, don’t return 
a value in transfer(). Since Lyra aims to support blue-chip tokens, it should refactor and use 
the safe transfer variant.

**Recommended mitigation:**
Use Open Zeppelin’s SafeERC20 encapsulation of ERC20 transfer functions.

**Team response:**
USDT/BNB will not be supported for this set of contracts. Tokens supported will be limited to 
only those that do not return boolean values.

### TRST-M-5 recoverFunds() does not handle popular ERC20 tokens like BNB
**Description:** 
recoverFunds() is used for recovery in case of mistakenly-sent tokens. However, it uses unsafe 
transfer to send tokens back, which will not support 100s of non-compatible ERC20 tokens. 
Therefore it is likely unsupported tokens will be unrecoverable.
```solidity
  if (token == quoteAsset || token == baseAsset || token == weth) {
      revert CannotRecoverRestrictedToken(address(this));
    }
        token.transfer(recipient, token.balanceOf(address(this)));
```

**Recommended Mitigation:**
Use Open Zeppelin’s SafeERC20 encapsulation of ERC20 transfer functions.

**Team response:**
This function exists purely as an additional recovery mechanism that should never really be 
used - it is not core to the functionality of the protocol. Will not be changed at this stage.


### TRST-M-6 option board will be settled with incorrect prices when settled after a delay
**Description:** 
`settleExpiredBoard()` runs after a board expires to perform accounting. It uses 
`getSettlementPriceForMarket()` to get the settlement price, which simply returns the current 
price. It does not account for the possibility that the function was called after some delay, and 
that the current price does not reflect the option’s expiry value. This situation could arise from 
many different reasons. For example, keepers may have been offline, or the network was 
halted for some time.

**Recommended Mitigation:**
Only accept the spot price if the time elapsed since expiry is smaller than some parameter. 
Otherwise, update the settlement value using a gov-only function.

**Team response:**
This is more a design choice. The first seen spot price after a large outage feels like a better 
alternative than to rely on a centralized source for the settlement price.

### TRST-M-7 attackers can delay or disrupt hedging activity by abusing mutual exclusion with updateCollateral()
**Description:**
`hedgeDelta()` is the engine behind the PoolHedger.sol used to offset exposure.
 GMX increase / decrease position requests are performed in two steps to minimize slippage attacks. Firstly, 
users call `increasePositionRequest()`. Every short period (usually several seconds), GMX keeper 
will execute all requests in a batch. The PoolHedger deals with this pending state using the 
**pendingOrderKey** parameter. When it is not 0, it is the key received from the last GMX 
position request. When there is a pending action, `hedgeDelta()` as well as `updateCollateral()` 
cannot be called. The latter function is another permissionless entry point, which triggers the 
correction of the leverage ratio on GMX to the target. The issue stems from the fact there are 
no DOS-preventions put in place, which allow attackers to continually call `updateCollateral()` 
as soon as the previous request completes, keeping the Hedger ever busy perfecting the 
leverage ratio, albeit not hedging properly. If done for a long enough period, the impact is an 
increased insolvency risk for the protocol as it is not delta-neutral.

**Recommended mitigation:**
One option is to make sure the delta correction is significant for it to succeed, preventing the 
DOS. Another option is to refactor the code to have only one entry point. This will guarantee 
the prioritization of delta-neutrality over reaching the target leverage ratio.

**Team response:**
Fixed

### TRST-M-8 protectedQuote can be manipulated by calling processDepositQueue when large price moves in base asset occur
**Description:**
The protectedQuote storage variable ensures that a portion of the liquidity pool is preserved 
even in the case of a “contract adjustment event” (i.e. when the pool has become insolvent). 
However, the protectedQuote is updated on deposits and withdraws using the following 
calculation:
```solidity
       protectedQuote = (liquidity.NAV - withdrawalValue).multiplyDecimal(
       DecimalMath.UNIT - lpParams.adjustmentNetScalingFactor
      );
```
The new value depends on the Net Asset Value (NAV) of the pool which, in turn, depends on 
the current hedge and price of the base asset. If the base asset moves sharply the NAV can 
drop, which can lead to a large drop in the **protectedQuote**. 
An attacker could call **initiateDeposit** with a small amount periodically. After a week’s delay 
they are then able to call **processQueuedDeposit** with the same periodicity. They can watch 
for a sharp drop in NAV and manipulate the protectedQuote down. If the price of the base 
asset moves even further this will most likely trigger the circuit breaker for contract 
adjustments and then lock in the reduced **protectedQuote** leading to losses for LPs. 

**Recommended mitigation:**
It’s not entirely clear what a good mitigation would be. The **protectedQuote** needs to be 
updated as the pool makes profits or losses. Perhaps a calculation that limits the magnitude 
by which it can change in a single step should be considered.

**Team response:**
Not really problematic that the values are being updated. The idea behind the timing of 
updates being when deposits/withdrawals are processed is they will be blocked when circuit 
breakers are running. The free liquidity circuit breaker in particular really helps with 
preventing any potential value manipulation as flagged in this issue.

### TRST-M-9 canHedge may return true when there is insufficient GMX liquidity to facilitate hedging, causing insolvency risks
**Description:**
Lyra’s security model relies on being able to hedge and achieve delta-neutrality when opening 
a user position. The check is done in canHedge() in GMXFuturesPoolHedger. After expected 
hedge delta and current hedge delta are fetched, remainingDeltas is assigned the amount of 
liquidity of the side that will be bought. Then this check is made:
```solidity
      uint absHedgeDiff = (Math.abs(expectedHedge) - Math.abs(currentHedge));
      if (remainingDeltas < 
           absHedgeDiff.multiplyDecimal(futuresPoolHedgerParams.marketDepthBuffer)) {
        return false;
      }

```
The issue is that the checked requirement for GMX liquidity is not strict enough. If 
**expectedHedge** and **currentHedge** have different signs, **remainingDeltas** needs to be above 
**expectedHedge**. That’s because the current holdings can’t be deducted from the necessary 
delta. The impact is that the function would approve sign-switching hedges more leniently 
than it should.

**Recommended mitigation:**
Check if **expectedHedge** and **currentHedge** have different signs and change logic accordingly.

**Team response:**
Valid; but similar to **H-1** canHedge is more a safety rail than a core requirement of system 
operation. The **deltaThreshold** parameter will cause the hedger to be updated more 
frequently if the hedged delta and expected delta diverge by a large enough amount - which 
will make these checks operate as expected. Will not be resolved at this stage.


### TRST-M-10 canHedge will return true when trade moves delta in same direction as expectedHedge even when that leads to more risk
**Description:**
Lyra’s security model relies on being able to hedge and achieve delta-neutrality when opening 
a user position. The check is done in canHedge() in GMXFuturesPoolHedger. The function will 
return true when the delta of the trade has the same sign as the expectedHedge. For example:
```solidity 
    // expected hedge is positive, and trade increases delta of the pool - risk is reduced, so accept trade
    if (increasesPoolDelta && expectedHedge >= 0) {
        return true;
          }
```
However, this will not always reduce the pool delta. For example an **expectedHedge** of 5 
would indicate the pool delta is -5. A trade which increases pool delta by 11 would mean the 
subsequent **expectedHedge** would be -6. In general, if the **expectedHedge** is equal to n/-n 
then a trade which increases/decreases pool delta by greater than 2*n will increase risk. 

**Recommended mitigation:**
Take the magnitude of the trade size into account when performing this check.

**Team response:**
Valid, but the additional complexity is too much to add at this stage when benefits are 
minimal. The inclusion of `strikeId` to the canHedge function will enable more detailed checks 
that check the exact delta risk added to be added in the future.

### TRST-M-11 Decreasing a losing hedge position could make it overly-leveraged
**Description:**
It may be necessary to decrease a position in order to be delta neutral. When 
GMXFuturesPoolHedger does that, it also decreases the collateral so that the leverage ratio 
would equal the set **targetLeverage**. In GMX, fees are deducted from collateral when losses 
are realized. Therefore, the code takes into account that additional collateral needs to be sent, 
to make up for the fees deducted. It’s done in this block:
```solidity
      if (currentPos.unrealisedPnl < 0) {
          uint adjustedDelta = Math.abs(currentPos.unrealisedPnl).multiplyDecimal(sizeDelta)divideDecimal      (currentPos.size);
      if (adjustedDelta > collateralDelta) {
          collateralDelta = 0;
      } else {
            collateralDelta -= adjustedDelta;
           }
       }
```
Notably, when **adjustedDelta > collateralDelta** is true, **collateralDelta** is zero-ed out. Since 
GMX decreasePositionRequest() receives a uint as the collateral delta and decreases by that 
amount, the function is not able to add the delta difference. However, that collateral debt is 
effectively forgotten, and as a result, the leverage ratio could be higher than intended. The 
impact is an increased liquidation risk which is not part of the Lyra risk model.

**Recommended mitigation:**
If adjustedDelta > collateralDelta holds, make a positionRouter.createIncreasePosition() call 
with the difference in deltas.

**Team response:**
If the hedger is over-leveraged after the decrease position request; keepers will be able to 
follow up with a updateCollateral() request almost immediately to increase the collateral.
Whilst valid, it is a very unlikely case with minimal impact. The recommendation would break 
the concept of only one pending position request for the hedger at any time. As the risk is very 
temporary and there is no easy fix, this will not be resolved.



## Low Risk
### TRST-L-1 Attacker can take over GMXAdapter implementation contract

**Description:**
GMXAdapter inherits from BaseExchangeAdapter. It is an implementation contract for a 
transparent proxy and has the following initializer:
```solidity
       function initialize() external initializer {
         __Ownable_init();
       }
``` 
Therefore, an attacker can call initialize() on the implementation contract and become the 
owner. At this point they can do just about anything to this contract, but it has no impact on 
the proxy as it is using separate storage. If there was a delegatecall coded in GMXAdapter, 
attacker could have used it to call an attacker’s contract and execute the SELFDESTRUCT 
opcode, killing the implementation. With no implementation, the proxy itself would not be 
functional until it is updated to a new implementation. It is ill-advised to allow anyone to have 
control over implementation contracts as future upgrades may make the attack surface 
exploitable.

**Recommended Mitigation:**
The standard approach is to call from the constructor the _disableInitializers() from Open 
Zeppelin’s Initializable module

**Team Response:**
As this has no real impact on the protocol this will not be resolved. Recommendation will be 
followed in subsequent upgrades.


### TRST-L-2 setPositionRouter leaks approval to previous positionRouter
**Description:**
positionRouter is used to change GMX positions in GMXFuturesPoolHedger. It can be replaced 
by a new router if GMX redeploys, for example if a bug is found or the previous one is hacked. 
The new positionRouter receives approval from the contract. However, approval to the 
previous positionRouter is not revoked.
```solidity
    function setPositionRouter(IPositionRouter _positionRouter) external onlyOwner {
      positionRouter = _positionRouter;
        router.approvePlugin(address(positionRouter));
    emit PositionRouterSet(_positionRouter);
       }
```
A number of unlikely, yet dire scenarios could occur.

**Recommended Mitigation:**
Use router.denyPlugin() to remove privileges from the previous **positionRouter**.

**Team Response:**
By denying the plugin you may also introduce potential issues where a pending position is 
blocked and funds are lost. In the case of a bugged/hacked router, it makes more sense to 
shut off the existing hedger completely and redeploy with relevant changes.

**Trust Security response:**
Point is somewhat correct. Would advise for default behavior to be denyPlugin() in 
setPositionRouter(), with an permissioned approvePlugin() method for unblocking funds.



### TRST-L-3 PoolHedger can receive ETH directly from anyone
**Description:**
A `receive()` function has been added to GMXFuturesPoolHedger, so that it is able to receive 
ETH from GMX as request refunds. However, it is not advisable to have an open receive() 
function if it is not necessary. Users may wrongly send ETH directly to PoolHedger and lose it 
forever.
```solidity
           receive() external payable {}
``` 
**Recommended Mitigation:**
Add a msg.sender check in the receive() function, and make sure sender is positionRouter.

**Team response:**
Will not be resolved at this stage.


### TRST-L-4 GMXAdapter minReturnPercent can be over 100%, requiring positive slippage for exchange to execute
**Description:**
In GMXAdapter `exchangeFromExactBase()`, minReturnPercent is used to set the minimum 
output tokens for the trade:
```solidity
    uint minOut = tokenInPrice.multiplyDecimal(marketPricingParams[_optionMarket].minReturnPercent)
      .multiplyDecimal(_amountBase)
       .divideDecimal(tokenOutPrice);
```
Since the rest of the calculation is done precisely the same as in positionRouter, except 
imposed fees, if minReturnPercent is above 100% transactions will never succeed. However, 
it’s allowed to be up to 120%. It appears the reason for that is to protect against any 
discrepancy that may occur in the future. It still does not seem to be correct to fix 
discrepancies with this parameter, as any change that causes the formula to not work needs 
to be examined closely rather than suppressed.

**Recommended mitigation:**
Allow **minReturnPercent** up to 100% (1e18).

**Team response:**
**minReturnPercent** has been removed and the functionality merged with the variable 
**staticSwapFeeEstimate**


### TRST-L-5 Slippage in hedger may be double what is set in slippage parameter
**Description:** 
`_decreasePosition()` in GMXFuturesPoolHedger calls the GMX `createDecreasePosition()` entry 
point, passing **minOut** = 0. According to docs, 0 should only be used if there are no swaps. But 
in the case of a long position, there is a swap (base to quote). The swap direction has the same 
exposure to base/quote as position decrease exposure. This means the slippage is applied 
twice. Therefore, acceptableSpotSlippage needs to be set to half of the intended slippage. 
Similar situation for createIncreasePosition, again with double the exposure. 

**Recommended mitigation:**
Make sure acceptableSpotSlippage is set so that the protocol is satisfied with losing the 
slippage amount twice per cycle.

**Team response:**
Acknowledged.

### TRST-L-6 LiquidityToken owner can freeze all token interactions and suspend the platform
**Description:** 
Owner can freely set the **liquidityTracker** variable in LiquidityToken. It is used in 
`_afterTokenTransfer()`:
```solidity
    function _afterTokenTransfer(address from, address to, uint amount) internal override {
      if (address(liquidityTracker) != address(0)) {
        if (from != address(0)) {
    liquidityTracker.removeTokens(from, amount);
    }
    if (to != address(0)) {
        liquidityTracker.addTokens(to, amount);
           }
         }
       }
```
If owner is compromised, the platform can be immediately shut down by passing an invalid 
address as **liquidityTracker**.

 ### TRST-L-7 Owner can manipulate board’s baseIv
 **Description:** 
In OptionMarket, board characteristics can only be changed when they are frozen. However, 
freezing and unfreezing are done instantly using `setBoardFrozen()`. Therefore, the owner can 
abuse a temporary freeze-unfreeze and inject any sort of malicious behavior, such as setting 
a low baseIv to make personal profit from the protocol. It is advised to put a time lock on 
unfreeze for greater trustability.

### TRST-L-8 Owner can drain liquidity pool instantly
**Description:** 
In LiquidityPool, `transferQuoteToHedge()` allows the defined poolHedger to pull up to the 
entire available liquidity. Since poolHedger can be set instantly using `setPoolHedger()`, it 
represents significant damage potential. The recommendation is to impose a timelock, either 
at the contract level or at the owner level (time-locked governance contract).


## Informational
### Implement input validation when possible:
It’s always worthwhile to check for potential run-time issues during assignment time. For 
example, it is clear that **priceVarianceCBPercent > gmxUsageThreshold** should always hold. 
Similarly, **maxLeverage > targetLeverage** must be true. It’s advisable to go through all 
parameter assignments and make sure insensible values are disallowed.

### Improve future-proofing by explicitly matching types
There are some instances in the code base where a long if/else clause ends with an else that 
assumes the enum value is the remaining value after matching all other options. This pattern 
is prone to future errors as additional types are added, and the existing code would not throw 
any warnings but behave incorrectly. For example, in `_updateExposure()` the last else clause 
catches OptionType.SHORT_PUT_QUOTE, but if new options are added this would introduce 
a bug.

### Misleading variable names
There are some instances where possibly code has been duplicated without proper renaming. 
For example, in GMXAdapter, _getMaxPrice() is implemented as:
```solidity
    function _getMaxPrice(address asset) internal view returns (uint) {
        uint minPrice = vault.getMaxPrice(asset);
       return ConvertDecimals.normaliseTo18(minPrice, GMX_PRICE_PRECISION);
         }
```
Another example is the function`_getPendingIncreaseCollateralDelta()` which actually returns 
the **amountIn** variable.
Such patterns are prone to the introduction of bugs. 

### Redundant event parameters
Some events have unuseful parameters. For example, in `cancelPendingOrder()`, the event 
OrderCanceled can only be emitted if **success** is true (otherwise the transaction would revert). 

### Adopt single-validation pattern
Some parameters are checked in multiple places for the same conditions, but are only 
changeable from a single gateway. It is preferable to move the check to the gateway and avoid 
SLOAD checks at runtime. For example, **staticSwapFeeEstimate** is checked in 
`_estimateExchangeFee()` and `exchangeFromExactBase()`, but it’s only set at 
`setMarketPricingParams()`.

### Model-related risks
**Description:**
Although the Black-Scholes-Merton equation is regularly used to price options, the formula 
itself predicts that the “implied volatility for a particular stock would be the same for all strikes 
and maturities” [1]. The Lyra project is aware that this is incorrect [3] and uses the notion of 
an Implied Volatility Surface to account for the true behavior of options, which deviates from 
the Black-Scholes-Merton model.
However, even this may not be enough. One notable criticism comes from Haug and Taleb [2] 
who argue that even with Implied Volatility Surface modifications the model should not be 
used for options pricing since it assumes that the underlying probability distribution of real 
options prices is a Gaussian distribution. This assumption is unlikely to be true and leads to 
“tail risk”. The section “On the Mathematical Impossibility of Dynamic Hedging” is also worth 
reading given that it is a foundational assumption of the Lyra project.
Also, section “Black-Scholes in practice” of the Wikipedia entry [1] outlines a number of risks 
that are not handled by the Black-Scholes-Merton model. In addition to the volatility risk –
which Lyra attempts to hedge using implied volatility surfaces – the section mentions tail risk, 
liquidity risk, gap risk, and incorrect pricing for deep OTM and deep ITM options. 
Given that Lyra’s model for reducing risk leaves out these additional forms of risk there is a 
chance that one or more of them may cause insolvency under the right conditions.
[1] https://en.wikipedia.org/wiki/Black%E2%80%93Scholes_model
[2] “Why We Have Never Used the Black-Scholes-Merton Option Pricing Formula” Espen 
Gaarder Haug and Nassim Nicholas Taleb
[3] In section “Volatality Smiles” they say “One of the incorrect assumptions that the BlackScholes model makes is that the implied volatility is constant across all strikes within the same 
expiry” https://docs.lyra.finance/overview/how-does-lyra-work/options-pricing-and-theamm