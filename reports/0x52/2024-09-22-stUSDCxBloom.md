**Auditor**

[@IAm0x52](https://twitter.com/IAm0x52)

# Findings

## High Risk

### [H-01] BloomPool#redeemBorrower fails to decrement \_idToTotalBorrowed leading to large portions of borrower funds being permanently trapped in the contract

**Details**

[BloomPool.sol#L133-L136](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L133-L136)

    uint256 totalBorrowAmount = _idToTotalBorrowed[id];
    uint256 borrowAmount = _borrowerAmounts[msg.sender][id];

    reward = (_tbyBorrowerReturns[id] * borrowAmount) / totalBorrowAmount;

Rewards for borrower is calculated by taking borrower's share of total borrow and dividing returns evenly.

[BloomPool.sol#L137-L144](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L137-L144)

    require(reward > 0, Errors.ZeroRewards());

    _idToCollateral[id].assetAmount -= uint128(reward);
    _tbyBorrowerReturns[id] -= reward;
    _borrowerAmounts[msg.sender][id] -= borrowAmount;

    emit BorrowerRedeemed(msg.sender, id, reward);
    IERC20(_asset).safeTransfer(msg.sender, reward);

We see above that `_tbyBorrowerReturns` are decremented but `_idToTotalBorrowed` is not. This leads to diminishing rewards share. Take the following example, if there is 1000 borrowed evenly between two users and there are 100 returns. We would expect that 50 would be claimable by each user though this will not be the case. For first user claim:

    reward = 100 * 500 / 1000 = 50

After the claim `_tbyBorrowerReturns` == 50 `_idToTotalBorrowed` == 1000 so for the second claim:

    reward = 50 * 500 / 1000 = 25

We see the reward is incorrect and 25% of total rewards are now trapped permanently in the contract

**Lines of Code**

[BloomPool.sol#L132-L145](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L132-L145)

**Recommendation**

\_idToTotalBorrowed should be decremented by borrowAmount to keep reward rate constant across

**Remediation**

Fixed as recommended in bloom-v2 [PR#14](https://github.com/Blueberryfi/bloom-v2/pull/14)

### [H-02] BloomPool#\_normalize price will cause severe mis-pricing for RWAs that are not 18 dp

**Details**

[BloomPool.sol#L349-L351](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L349-L351)

    uint256 totalValue = (existingCollateral.mulWad(startPrice) + amount.mulWad(currentPrice)) / _rwaScalingFactor;
    uint256 totalCollateral = existingCollateral + amount;
    return uint128(totalValue.divWad(totalCollateral));

When calculating the totalValue, \_rwaScalingFactor is used above even though it is not necessary. Both `currentPrice` and `startPrice` are 18 dp and `amount` and `existingCollateral` are the same precision since the measure the same asset. Without `_rwaScalingFactor` total value shares the same precision as `totalCollateral`. This make any decimal scaling unnecessary.

Assuming we have a RWA that is less than 18 dp, our totalValue will be scaled downward by `18 - rwa dp`. This would propagate through to the final price calculation, resulting in a completely incorrect price.

**Lines of Code**

[BloomPool.sol#L344-L352](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L344-L352)

**Recommendation**

`_rwaScalingFactor` should be completely removed from the `totalValue` calculation

**Remediation**

Fixed as recommended in bloom-v2 [PR#17](https://github.com/Blueberryfi/bloom-v2/pull/17)

### [H-03] Malicious borrower can repeatedly fill then kill orders to permanently lock funds of lenders and other borrowers

**Details**

[Orderbook.sol#L118-L133](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/Orderbook.sol#L118-L133)

    function killBorrowerMatch(address lender) external returns (uint256 lenderAmount, uint256 borrowerReturn) {
        MatchOrder[] storage matches = _userMatchedOrders[lender];

        uint256 len = matches.length;
        for (uint256 i = 0; i != len; ++i) {
            if (matches[i].borrower == msg.sender) {
                lenderAmount = uint256(matches[i].lCollateral);
                borrowerReturn = uint256(matches[i].bCollateral);
                // Zero out the match order to preserve the array's order
                matches[i] = MatchOrder({lCollateral: 0, bCollateral: 0, borrower: address(0)});
                // Decrement the matched depth and open move the lenders collateral to an open order.
                _matchedDepth -= lenderAmount;
                _openOrder(lender, lenderAmount);
                break;
            }
        }

When a borrower kills a match, all values are zeroed out inside the MatchOrder struct and it is kept in the lender's matches array. If this borrower was to match again, they would add yet another entry to the array.

[Orderbook.sol#L252-L265](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/Orderbook.sol#L252-L265)

    if (remainingAmount != 0) {
        uint256 amountToRemove = Math.min(remainingAmount, matches[index].lCollateral);
        uint256 borrowAmount = uint256(matches[index].bCollateral);


        if (amountToRemove != matches[index].lCollateral) {
            borrowAmount = amountToRemove.divWad(_leverage);
            matches[index].lCollateral -= uint128(amountToRemove);
            matches[index].bCollateral -= uint128(borrowAmount);
        }
        remainingAmount -= amountToRemove;
        _idleCapital[matches[index].borrower] += borrowAmount;


        emit MatchOrderKilled(account, matches[index].borrower, amountToRemove);
        if (matches[index].lCollateral == amountToRemove) matches.pop();

When attempting to cancel or match orders, this array is cycled through until the specified amount is canceled/filled. The issue is that if this array is inflated by a malicious borrower then it would cause a OOG DOS. This would result in a permanent loss of funds for the lender as their orders could not be matched or cancelled.

**Lines of Code**

[Orderbook.sol#L118-L139](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/Orderbook.sol#L118-L139)

**Recommendation**

borrower address inside the matches mapping shouldn't be set to address(0). This why repeated fills and kills would only ever result in a single entry per borrower.

**Remediation**

Fixed as recommended in bloom-v2 [PR#15](https://github.com/Blueberryfi/bloom-v2/pull/15)

### [H-04] Yield spread cannot be decrease without causing significant loss to the stUSDC pool

**Details**

[StUsdc.sol#L313-L324](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L313-L324)

    function _liveTbyValue(IBloomPool pool) internal view returns (uint256 value) {
        uint256 startingId = lastRedeemedTbyId();
        // Because we start at type(uint256).max, we need to increment and overflow to 0.
        unchecked {
            startingId++;
        }
        uint256 lastMintedId = pool.lastMintedId();
        if (lastMintedId == type(uint256).max) return 0;
        for (uint256 i = startingId; i <= lastMintedId; ++i) {
            value += pool.getRate(i).mulWad(_tby.balanceOf(address(this), i)); <- @audit rate pulled from bloomPool.sol
        }
    }

[BloomPool.sol#L476-L492](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L476-L492)

    function getRate(uint256 id) public view override returns (uint256) {
        TbyMaturity memory maturity = _idToMaturity[id];
        RwaPrice memory rwaPrice = _tbyIdToRwaPrice[id];


        if (rwaPrice.startPrice == 0) {
            revert Errors.InvalidTby();
        }
        // If the TBY has not started accruing interest, return 1e18.
        if (block.timestamp <= maturity.start) {
            return Math.WAD;
        }


        // If the TBY has matured, and is eligible for redemption, calculate the rate based on the end price.
        uint256 price = rwaPrice.endPrice != 0 ? rwaPrice.endPrice : _rwaPrice();
        uint256 rate = (uint256(price).divWad(uint256(rwaPrice.startPrice)));
        return _takeSpread(rate); <- @audit rate is dependent on spread
    }

When the value of the tbys are calculated, the rate is adjusted by the current spread specified. The issue is that the spread can be changed at any time by the owner. Since spread is a contract wide variable, rather than tby specific, any adjustment would lead to a retroactive change in the valuation of all tbys. The result is that any change to this value would lead to a sudden change in the valuation of the tbys and by extend the value of the stUSDC pool. Decreasing it would cause a large loss to all stUSDC holders.

**Lines of Code**

[PoolStorage.sol#L149-L151](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/PoolStorage.sol#L149-L151)

**Recommendation**

\_spread should be cached during the creation of the tby.

**Remediation**

Fixed as recommended in bloom-v2 [PR#20](https://github.com/Blueberryfi/bloom-v2/pull/20). \_spread is now cached in the rwaPrice struct for each tby upon creation.

### [H-05] The yield distribution methodology for stUSDC#poke will lead to substantial loss of yield due to MEV for both stUSDC and StakeUpStaking

**Details**

[StUsdc.sol#L166-L185](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L166-L185)

    uint256 globalShares_ = _globalShares;
    uint256 protocolValue = _protocolValue(pool);
    uint256 newUsdPerShare = protocolValue.divWad(globalShares_);
    uint256 lastUsdPerShare = _lastUsdPerShare;

    if (newUsdPerShare > lastUsdPerShare) {
        uint256 yieldPerShare = newUsdPerShare - lastUsdPerShare;
        // Calculate performance fee
        uint256 fee = _calculateFee(yieldPerShare, globalShares_);
        // Calculate the new total value of the protocol for users
        uint256 userValue = protocolValue - fee;
        newUsdPerShare = userValue.divWad(globalShares_);
        // Update state to distribute yield to users
        _setUsdPerShare(newUsdPerShare);
        // Process fee to StakeUpStaking
        _processFee(fee);
    } else if (newUsdPerShare < lastUsdPerShare) {
        // If the protocol has lost value, we need to update the USD per share to reflect the loss.
        _setUsdPerShare(newUsdPerShare);
    }

We see above that stUSDC#poke calculates the change in value of the last 24 hours and distributes it atomically to all current stakers. This leads to a significant opportunity to MEV the contract. Since there is no timelock or fees a user can do the following:

1. Borrow a large amount of USDC
2. Deposit the entire amount into stUSDC to own 95%+ of the pool
3. Call poke, receiving 95% of the rewards for themselves
4. Withdraw USDC from stUSDC
5. Repay flashloan

This allows them to steal nearly all the yield of the contract.

**Lines of Code**

[StUsdc.sol#L154-L190](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L154-L190)

**Recommendation**

Yield should be dripped to the contract over the course of 24 hours. Additionally deposits must be updated to account for the yield being dripped so that they do not accrue more value than they should. StakeUpStaking also suffers from a similar issue and I would recommend adding a deposit timelock.

**Remediation**

Fixed as recommended in stakeup-contracts [PR#89](https://github.com/stakeup-protocol/stakeup-contracts/pull/89) (reward drip), [PR#91](https://github.com/stakeup-protocol/stakeup-contracts/pull/91) (deposit timelock) and [PR#96](https://github.com/stakeup-protocol/stakeup-contracts/pull/96) (tby discount)

### [H-06] Shares of stUSDC can be lost/gained during cross chain transfers due to differing conversion rates across chains

**Details**

[StUsdcLite.sol#L418-L428](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdcLite.sol#L418-L428)

    function _debit(uint256 _amountLD, uint256 _minAmountLD, uint32 _dstEid)
        internal
        override
        returns (uint256 amountSentLD, uint256 amountReceivedLD)
    {
        (amountSentLD, amountReceivedLD) = _debitView(_amountLD, _minAmountLD, _dstEid);


        uint256 shares = sharesByUsd(amountSentLD);
        _burnShares(msg.sender, shares);
        _setTotalUsd(_getTotalUsd() - amountSentLD);
    }

[StUsdcLite.sol#L430-L438](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdcLite.sol#L430-L438)

    function _credit(address _to, uint256 _amountToCreditLD, uint32 /*_srcEid*/ )
        internal
        override
        returns (uint256 amountReceivedLD)
    {
        _mintShares(_to, sharesByUsd(_amountToCreditLD));
        _setTotalUsd(_getTotalUsd() + _amountToCreditLD);
        return _amountToCreditLD;
    }

We see above that when sending payments that value is converted to shares to burn but then the value is the amount transferred. This is confirmed by \_credit where we see that the \_amountToCreditLD is converted to shares.

This creates an issue when if the if the transfer is in progress when poke is called. This means the conversion rate will higher or lower than when it was sent, resulting in shares being lost or gained during transit. Shares gained can result in the contract being undercollateralized.

**Lines of Code**

[StUsdcLite.sol#L418-L438](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdcLite.sol#L418-L438)

**Recommendation**

Cross chain transfers should send shares NOT value since shares are constant and value is not.

**Remediation**

Fixed as recommended in stakeup-contracts [PR#93](https://github.com/stakeup-protocol/stakeup-contracts/pull/93)

### [H-07] SUP rewards will be completely lost when depositing TBYs to stUSDC via wstUSDC

**Details**

[StUsdc.sol#L118-L130](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L118-L130)

    function depositTby(uint256 tbyId, uint256 amount) external nonReentrant returns (uint256 amountMinted) {
        IBloomPool pool = _bloomPool;
        require(amount > 0, Errors.ZeroAmount());
        require(!pool.isTbyRedeemable(tbyId), Errors.RedeemableTbyNotAllowed());
        // If the token is a TBY, we need to get the current exchange rate of the token
        //     to accurately calculate the amount of stUsdc to mint.
        amountMinted = pool.getRate(tbyId).mulWad(amount) * _scalingFactor;
        _deposit(amountMinted);
        _mintRewards(amountMinted);

        emit TbyDeposited(msg.sender, tbyId, amount, amountMinted);
        _tby.safeTransferFrom(msg.sender, address(this), tbyId, amount, "");
    }

When minting stUSDC with tbys the depositor will receive rewards from stUSDC, which are sent to msg.sender.

[WstUsdc.sol#L51-L55](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/WstUsdc.sol#L51-L55)

    function depositTby(uint256 tbyId, uint256 amount) external override returns (uint256 amountMinted) {
        _tby.safeTransferFrom(msg.sender, address(this), tbyId, amount, "");
        amountMinted = _stUsdc.depositTby(tbyId, amount);
        amountMinted = _mintWstUsdc(amountMinted);
    }

We see above that wstUSDC allows direct minting of wstUSDC with tbys. In this case, the wstUSDC contract will be msg.sender and they will be the recipient of the rewards. These rewards are never transferred to the user and will be permanently lost.

**Lines of Code**

[WstUsdc.sol#L51-L55](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/WstUsdc.sol#L51-L55)

**Recommendation**

wstUSDC should check for any rewards and forward them to the user

**Remediation**

Fixed as recommended in stakeup-contracts [PR#94](https://github.com/stakeup-protocol/stakeup-contracts/pull/94)

### [H-08] TBY#burn incorrectly burns id 0 for all burns causing complete loss of funds for lenders

**Details**

[Tby.sol#L78-L82](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/token/Tby.sol#L78-L82)

    function burn(uint256 id, address account, uint256 amount) external onlyBloom {
        _totalSupply[id] -= amount;
        _burn(account, 0, amount);
        emit Burn(account, id, amount);
    }

The burn function is misconfigured and will always attempt to burn id == 0 from the user. For the first round of tbys the burn function will work but after that it will result in all tbys being unredeemable and causing complete loss of funds to LPs.

**Lines of Code**

[Tby.sol#L78-L82](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/token/Tby.sol#L78-L82)

**Recommendation**

\_burn should be correct to take id rather than 0 as the input

**Remediation**

Fixed as recommended in bloom-v2 [PR#13](https://github.com/Blueberryfi/bloom-v2/pull/13/)

## Medium Risk

### [M-01] \_convertMatchOrders will fail to function properly for lenders if any borrower has canceled a matched order

**Details**

    function killBorrowerMatch(address lender) external returns (uint256 lenderAmount, uint256 borrowerReturn) {
        MatchOrder[] storage matches = _userMatchedOrders[lender];

        uint256 len = matches.length;
        for (uint256 i = 0; i != len; ++i) {
            if (matches[i].borrower == msg.sender) {
                lenderAmount = uint256(matches[i].lCollateral);
                borrowerReturn = uint256(matches[i].bCollateral);
                // Zero out the match order to preserve the array's order
                matches[i] = MatchOrder({lCollateral: 0, bCollateral: 0, borrower: address(0)});
                // Decrement the matched depth and open move the lenders collateral to an open order.
                _matchedDepth -= lenderAmount;
                _openOrder(lender, lenderAmount);
                break;
            }
        }

When a borrower kills a match, all values are zeroed out inside the MatchOrder struct and it is kept in the lender's matches array. This leads to a problem later when trying to match the lenders orders.

[BloomPool.sol#L382-L415](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L382-L415)

    uint256 length = matches.length;
    for (uint256 i = length; i != 0; --i) {
        uint256 index = i - 1;


        if (remainingAmount != 0) {
            (uint256 lenderFunds, uint256 borrowerFunds) =
                _calculateRemovalAmounts(remainingAmount, matches[index].lCollateral, matches[index].bCollateral);
            uint256 amountToRemove = lenderFunds + borrowerFunds;

            if (amountToRemove == 0) break; <- @audit matching loop breaks when collateral in match is 0
            remainingAmount -= amountToRemove;

            _borrowerAmounts[matches[index].borrower][id] += borrowerFunds;
            borrowerAmountConverted += borrowerFunds;

            emit MatchOrderConverted(id, account, matches[index].borrower, lenderFunds, borrowerFunds);

            if (lenderFunds == matches[index].lCollateral) {
                matches.pop();
            } else {
                matches[index].lCollateral -= uint128(lenderFunds);
            }
        } else {
            break;
        }
    }

We see above that when the collateral is 0 as it is with a canceled match, the matching loop breaks prematurely. Given this is FILO matching, all deposits made prior the cancelled order will be impossible to match. Those borrowers will need to all manually cancel their orders and resubmit. This presents a high potential for griefing as a malicious borrower can DOS lenders and borrowers alike.

**Lines of Code**

[BloomPool.sol#L377-L416](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L377-L416)

**Recommendation**

BloomPool#\_convertMatchOrders should pop empty matches and continue rather than breaking.

**Remediation**

Fixed as recommended in bloom-v2 [PR#15](https://github.com/Blueberryfi/bloom-v2/pull/15)

### [M-02] In the event of a partial match inside \_convertMatchOrders, borrower funds will be over-allocated

**Details**

[BloomPool.sol#L399-L403](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L399-L403)

    if (lenderFunds == matches[index].lCollateral) {
        matches.pop();
    } else {
        matches[index].lCollateral -= uint128(lenderFunds);
    }

Above is the final portion of the matching loop in which filled orders are handled. `lenderFunds == matches[index].lCollateral` indicates the order has been fully matched. In this scenario it is correctly popped. The other case is when the the match is partially filled. We see that lCollateral is decreased but bCollateral is not. The result is that borrower funds can be double filled. This creates a shortfall in funds that can only be remedied by donating the over-allocated funds to the contract.

**Lines of Code**

[BloomPool.sol#L377-L416](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L377-L416)

**Recommendation**

`matches[index].lCollateral` should also be decremented

**Remediation**

Fixed as recommended in bloom-v2 [PR#16](https://github.com/Blueberryfi/bloom-v2/pull/16)

### [M-03] Using address rather than bytes32 for destinationAddress in wstUsdcBridge#bridgeWstUsdc causes incompatibility with chains that utilize 32 byte addresses

**Details**

[WstUsdcBridge.sol#L44-L49](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/messaging/WstUsdcBridge.sol#L44-L49)

    function bridgeWstUsdc(
        address destinationAddress, <- @audit uses address instead of bytes32
        uint256 wstUsdcAmount,
        uint32 dstEid,
        LzSettings calldata settings
    ) external payable returns (LzBridgeReceipt memory bridgingReceipt) {

wstUsdcBridge#bridgeWstUsdc utilizes type `address` for destinationAddress which breaks LZ compatibility with chains that use a 32 byte address instead of a 20 byte address like evm chains.

**Lines of Code**

[WstUsdcBridge.sol#L44-L58](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/messaging/WstUsdcBridge.sol#L44-L58)

**Recommendation**

Change destinationAddress from `address` to `bytes32`

**Remediation**

Fixed as recommended in stakeup-contracts [PR#92](https://github.com/stakeup-protocol/stakeup-contracts/pull/92)

### [M-04] stUSDC#deposit compares current tby price with potentially stale pool price which can lead to small yield loss for pool

**Details**

[StUsdc.sol#L154-L157](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L154-L157)

    function poke() external nonReentrant {
        uint256 currentTimestamp = block.timestamp;
        uint256 lastUpdate = _lastRateUpdate;
        if (currentTimestamp - lastUpdate < 24 hours) return;

stUSDC#poke syncs the value of the pool and as we see above, this is limited to one update every 24 hours.

[StUsdc.sol#L118-L130](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L118-L130)

    function depositTby(uint256 tbyId, uint256 amount) external nonReentrant returns (uint256 amountMinted) {
        IBloomPool pool = _bloomPool;
        require(amount > 0, Errors.ZeroAmount());
        require(!pool.isTbyRedeemable(tbyId), Errors.RedeemableTbyNotAllowed());
        // If the token is a TBY, we need to get the current exchange rate of the token
        //     to accurately calculate the amount of stUsdc to mint.
        amountMinted = pool.getRate(tbyId).mulWad(amount) * _scalingFactor;
        _deposit(amountMinted);
        _mintRewards(amountMinted);

        emit TbyDeposited(msg.sender, tbyId, amount, amountMinted);
        _tby.safeTransferFrom(msg.sender, address(this), tbyId, amount, "");
    }

This is generally fine but can lead to tbys being relatively overvalued. If the pool has gone 12 hours without syncing, there will be 12 hours worth of value that has accumulated but that has not been accounted for. Meanwhile we see that during deposit, it pulls a fresh exchange rate from the pool. This difference will cause the tby being deposited to be slightly overvalued compared to an exact same tby in the pool.

**Lines of Code**

[StUsdc.sol#L118-L130](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L118-L130)

**Recommendation**

The tby rate should be adjusted to exclude yield that has accumulated since the last poke (i.e. if it's been 12 hours since poke, 12 hours of yield should be deducted).

**Remediation**

Fixed in stakeup-contracts [PR#96](https://github.com/stakeup-protocol/stakeup-contracts/pull/96)

### [M-05] In the event of yield loss, yield will be double counted leading to excess fees

**Details**

[StUsdc.sol#L171-L181](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L171-L181)

    if (newUsdPerShare > lastUsdPerShare) {
        uint256 yieldPerShare = newUsdPerShare - lastUsdPerShare;
        // Calculate performance fee
        uint256 fee = _calculateFee(yieldPerShare, globalShares_);
        // Calculate the new total value of the protocol for users
        uint256 userValue = protocolValue - fee;
        newUsdPerShare = userValue.divWad(globalShares_);
        // Update state to distribute yield to users
        _setUsdPerShare(newUsdPerShare);
        // Process fee to StakeUpStaking
        _processFee(fee);

Whenever yield is accumulated by the contract, a fee is taken. In itself this is fine but it can lead to double counted yield in the event that the share price decrease. Take the following example, the share price increases from 1 -> 1.1. A fee will be taken on the 0.1 share price gain. Now the token decreases and increases again as follows 1.1 -> 1.05 -> 1.1. On the move from 1.05 -> 1.1 the fee will be taken again, effectively double charging fees to the users.

**Lines of Code**

[StUsdc.sol#L154-L190](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L154-L190)

**Recommendation**

The contract should implement a highwater tracker for the share price and only pay out fees for increase above that mark.

**Remediation**

Fixed as recommended in stakeup-contracts [PR#89](https://github.com/stakeup-protocol/stakeup-contracts/pull/89)

### [M-06] SUP rewards for depositing TBYs can be gamed by depositing TBYs that are close to expiration

**Details**

[StUsdc.sol#L118-L130](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L118-L130)

    function depositTby(uint256 tbyId, uint256 amount) external nonReentrant returns (uint256 amountMinted) {
        IBloomPool pool = _bloomPool;
        require(amount > 0, Errors.ZeroAmount());
        require(!pool.isTbyRedeemable(tbyId), Errors.RedeemableTbyNotAllowed());
        // If the token is a TBY, we need to get the current exchange rate of the token
        //     to accurately calculate the amount of stUsdc to mint.
        amountMinted = pool.getRate(tbyId).mulWad(amount) * _scalingFactor;
        _deposit(amountMinted);
        _mintRewards(amountMinted);

        emit TbyDeposited(msg.sender, tbyId, amount, amountMinted);
        _tby.safeTransferFrom(msg.sender, address(this), tbyId, amount, "");
    }

When minting via `stUSDC#depositTBY`, rewards are paid out based on `amountMinted`. The goal of this reward is to incentivize users to deposit, thereby generating increased TVL for the protocol. This mechanism, however, can be gamed by depositing tbys that are very close to expiration. Due to the rate scaling these tbys will qualify for a large amount of shares while contributing very little to the yield.

**Lines of Code**

[StUsdc.sol#L118-L130](https://github.com/stakeup-protocol/stakeup-contracts/blob/b4d8a83e9455efb8c7543a0fc62b5aea598c7f49/src/token/StUsdc.sol#L118-L130)

**Recommendation**

Rewards paid should be proportional to the length of time left until maturity. The closer they are to maturity, the less rewards should be paid.

**Remediation**

Fixed as recommended in stakeup-contracts [PR#87](https://github.com/stakeup-protocol/stakeup-contracts/pull/87)
