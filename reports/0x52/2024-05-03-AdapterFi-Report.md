**Auditor**

[@IAm0x52](https://twitter.com/IAm0x52)

# Findings

## High Risk

### [H-01] Malicious proposer can use pregen_info to drain vault assets through sandwich attack

**Details**

[AdapterVault.vy#L662](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L662)

    self._balanceAdapters(claim_amount, pregen_info)

When claiming fees, funds must be removed from adapters to pay the claimants.

[PendleAdapter.vy#L313-L328](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/adapters/PendleAdapter.vy#L313-L328)

    pg: PregenInfo = empty(PregenInfo)
    if len(pregen_info) > 0:
        pg = _abi_decode(pregen_info, PregenInfo)
    else:
        #Info not provided, compute it expensively
        pg.approx_params_swapExactYtForPt = self.default_approx_params()
        pg.approx_params_swapExactTokenForPt = self.default_approx_params()
        ytToPTL: uint256 = 0
        pg.mint_returns, ytToPTL = self.estimate_mint_returns(asset_amount)
        pg.spot_returns = self.estimate_spot_returns(asset_amount)
        #we already paid the tax... why not reuse it...
        pg.approx_params_swapExactTokenForPt.guessOffchain = pg.spot_returns
        pg.approx_params_swapExactYtForPt.guessOffchain = ytToPTL

    #mint if minting price is better, then sell the YT.
    if pg.mint_returns > pg.spot_returns:

pregen_info allows the caller to provide data to control whether an adapter deposits via swapping or minting. This data must be trusted as it can force the vault to swap at very low prices. The proposer is NOT a trusted party and can easily abuse this.

The attack would be structured as follows on an imbalanced pool:

1. Swap a large amount of PT to asset to lower the price
2. Force vault to rebalance by swapping at low prices
3. Swap back into the pool and return the price to normal

Performing a sandwich attack allows the attacker to buy all the pools PT at a fraction of what they are worth to profit from the difference.

**Lines of Code**

[AdapterVault.vy#L645-L692](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L645-L692)

**Recommendation**

The \_balanceAdapters call should be set to `withdraw_only == true`

**Remediation**

Fixed are recommended.

### [H-02] Failure to pass \_withdraw_only to \_getBalanceTXs allows any withdrawing user to drain vault assets through sandwich attack

**Details**

[AdapterVault.vy#L1045-L1060](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L1045-L1060)

    def _balanceAdapters( _target_asset_balance: uint256, pregen_info: DynArray[Bytes[4096], MAX_ADAPTERS], _withdraw_only : bool = False, _max_txs: uint8 = MAX_BALTX_DEPOSIT ):
        # Make sure we have enough assets to send to _receiver.
        txs: BalanceTX[MAX_ADAPTERS] = empty(BalanceTX[MAX_ADAPTERS])
        blocked_adapters: address[MAX_ADAPTERS] = empty(address[MAX_ADAPTERS])


        # If there are no adapters then nothing to do.
        if len(self.adapters) == 0: return


        # Setup current state of vault & adapters & strategy.
        d4626_assets: uint256 = 0
        adapter_states: BalanceAdapter[MAX_ADAPTERS] = empty(BalanceAdapter[MAX_ADAPTERS])
        total_assets: uint256 = 0
        total_ratios: uint256 = 0
        d4626_assets, adapter_states, total_assets, total_ratios = self._getCurrentBalances()


        txs, blocked_adapters = self._getBalanceTxs(_target_asset_balance, _max_txs, self.min_proposer_payout, total_assets, total_ratios, adapter_states )

A typographical error in \_balanceAdapter fails to pass \_withdraw_only to \_getBalanceTxs.

[AdapterVault.vy#L1032-L1034](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L1032-L1034)

    def _getBalanceTxs( _target_asset_balance: uint256, _max_txs: uint8, _min_proposer_payout: uint256, _total_assets: uint256, _total_ratios: uint256, _adapter_states: BalanceAdapter[MAX_ADAPTERS], _withdraw_only : bool = False) -> (BalanceTX[MAX_ADAPTERS], address[MAX_ADAPTERS]):
        current_local_asset_balance : uint256 = ERC20(asset).balanceOf(self)
        return FundsAllocator(self.funds_allocator).getBalanceTxs( current_local_asset_balance, _target_asset_balance, _max_txs, _min_proposer_payout, _total_assets, _total_ratios, _adapter_states, _withdraw_only)

This is problematic as the default value is FALSE. This means that anyone withdrawing can freely rebalance the vault. As describe in H-01 as pregen_info must be trusted making this highly dangerous and allowing anyone to drain assets from an imbalanced vault.

**Lines of Code**

[AdapterVault.vy#L1045-L1098](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L1045-L1098)

**Recommendation**

Pass \_withdraw_only to \_getBalanceTxs as intended

**Remediation**

Fixed as recommended

## Medium Risk

### [M-01] Numerical error in \_claim_fees double counts yield fees in some scenarios

**Details**

[AdapterVault.vy#L681-L686](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L681-L686)

    if strat_fee_amount > 0 and self.owner != self.current_proposer:
        ERC20(asset).transfer(self.current_proposer, strat_fee_amount)

    # Is there anything left over to transfer for Yield? (Which might also include strat)
    if claim_amount > 0:
        ERC20(asset).transfer(self.owner, claim_amount + strat_fee_amount)

When claiming both types of fees in a single transaction, strat_fee_amount is double counted. First sent to the proposer then sent again to the owner of the vault.

**Lines of Code**

[AdapterVault.vy#L645-L692](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L645-L692)

**Recommendation**

strat_fee_amount should be set to 0 after it is sent to proposer:

        if strat_fee_amount > 0 and self.owner != self.current_proposer:
            ERC20(asset).transfer(self.current_proposer, strat_fee_amount)
    +       strat_fee_amount = 0

**Remediation**

Fixed as recommended

### [M-02] total_assets_deposited includes pre-slippage values leading to incorrect fee calculations over time

**Details**

[AdapterVault.vy#L1238-L1252](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L1238-L1252)

    assert total_after_assets > total_starting_assets, "ERROR - deposit resulted in loss of assets!"
    real_shares : uint256 = convert(convert((total_after_assets - total_starting_assets), decimal) * spot_share_price, uint256)

    if real_shares < transfer_shares:
        assert real_shares >= min_transfer_shares, "ERROR - unable to meet minimum slippage for this deposit!"

        # We'll transfer what was received.
        transfer_shares = real_shares
        log SlippageDeposit(msg.sender, _receiver, _asset_amount, ideal_shares, transfer_shares)

    # Now mint assets to return to investor.
    self._mint(_receiver, transfer_shares)

    # Update all-time assets deposited for yield tracking.
    self.total_assets_deposited += _asset_amount

When depositing asset into the vault, the user will likely experience slippage. Above however we see that the full asset amount is added to total_assets_deposited rather than the amount after slippage. This causes fees to be miscalculated as slippage is considered to be a "loss" of the vault which is incorrect.

**Lines of Code**

[AdapterVault.vy#L1209-L1257](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/AdapterVault.vy#L1209-L1257)

**Recommendation**

Post-slippage valuation should be used when calculating deposit amounts rather than pre-slippage amounts

**Remediation**

Fixed by redesigning the fee claiming functions. It now functions similarly to a withdraw with a specified amount of slippage allowed.

### [M-03] ADAPTER_BREAKS_LOSS_POINT is too tight for volatile assets leading to near constant adapter blocking

**Details**

[FundsAllocator.vy#L14](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/FundsAllocator.vy#L14)

    ADAPTER_BREAKS_LOSS_POINT : constant(decimal) = 0.00001

As seen above the stop loss point has been set exceedingly tight

[FundsAllocator.vy#L133-L138](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/FundsAllocator.vy#L133-L138)

    adapter_brakes_limit : uint256 = adapter.last_value - convert(convert(adapter.last_value, decimal) * ADAPTER_BREAKS_LOSS_POINT, uint256)
    if adapter.current < adapter_brakes_limit:
        # We've lost value in this adapter! Don't give it more money!
        blocked_adapters[blocked_pos] = adapter.adapter
        blocked_pos += 1
        adapter.delta = 0 # This will result in no tx being generated.

The PT tokens held by the contract are highly volatile and will trip this stop loss near constantly. The result is that the vault will be effectively nonfunctional for any volatile asset.

**Lines of Code**

[FundsAllocator.vy#L14](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/FundsAllocator.vy#L14)

**Recommendation**

The stoploss functionality is vestigial from when the fund allocator was meant to function with non-volatile lending platforms. This should either be removed entirely or widened significantly.

**Remediation**

Fixed by updating ADAPTER_BREAKS_LOSS_POINT to 0.05 (5%)

### [M-04] Withdraw and deposit methodology leads to valuation exploits under certain circumstances

**Details**

[FundsAllocator.vy#L107-L117](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/FundsAllocator.vy#L107-L117)

    for pos in range(MAX_ADAPTERS):
        adapter : BalanceAdapter = _adapter_balances[pos]
        if adapter.adapter == empty(address): break

        # If the adapte has been removed from the strategy then we must empty it!
        if adapter.ratio == 0 and adapter.current > 0:
            adapter.target = 0
            adapter.delta = max(convert(adapter.current, int256)*-1, adapter.max_withdraw) # Withdraw it all! (max_withdraw is a negative number)
        else:
            adapter.target = (total_adapter_target_assets * adapter.ratio) / _total_ratios
            adapter.delta = convert(adapter.target, int256) - convert(adapter.current, int256)

When depositing asset we see above that funds are distributed according to the adapter ratios.

[FundsAllocator.vy#L52-L79](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/FundsAllocator.vy#L52-L79)

    target_withdraw_balance : uint256 = _d4626_asset_target - _vault_balance

    # We're just going to walk through and empty adapters until we have
    # adequate funds in the vault for this withdraw.
    for pos in range(MAX_ADAPTERS):
        # Anything left to withdraw?
        if target_withdraw_balance == 0: break
        adapter : BalanceAdapter = _adapter_balances[pos]

        # End of adapters?
        if adapter.adapter == empty(address): break

        # If the adapte has been removed from the strategy then we must empty it!
        if adapter.ratio == 0 and adapter.current > 0:
            adapter.target = 0
            adapter.delta = max(convert(adapter.current, int256)*-1, adapter.max_withdraw) # Withdraw it all!

        elif adapter.current > 0:
            withdraw : uint256 = min(target_withdraw_balance, adapter.current)
            target_withdraw_balance = target_withdraw_balance - withdraw
            adapter.delta = convert(withdraw, int256) * -1

        if adapter.delta != 0:
            adapter_assets_allocated += convert(adapter.delta * -1, uint256)    # TODO : eliminate adapter_assets_allocated if never used.
            d4626_delta += adapter.delta * -1
            tx_count += 1

    assert target_withdraw_balance != 0, "ERROR - Unable to fulfill this withdraw!"

This is slightly different from how withdrawals are distributed. Above we see that withdrawals are serviced by simply removing funds from the first eligible adapter. This difference is the root cause of the issue. Take the following example:

Assume there are two adapters (50/50) currently operating on the vault and there is an increase of 10% to the assets held by adapter 1. Since the TWAP lags behind during sharp price movements the assets will be valued at ~90% of their actual value. An attacker can take the following steps to extract value from the vault:

1. Deposit 100 asset to the vault, depositing 50 to adapter 1 and 50 to adapter 2
2. Due to the valuation difference the vault with perceive a total value of 95 (50 \* 90% + 50) deposited
3. Withdraw all 95 assets. This will withdrawal all funds from adapter 1 which currently has a depressed valuation
4. This will result in the attacker receiving 105.55 (95 / 90%) for a profit of 5.55 assets

The larger the attacker's deposit the more funds it is possible to steal this way up to the value of assets held in adapter 1

**Lines of Code**

[FundsAllocator.vy#L36-L81](https://github.com/adapter-fi/AdapterVault/blob/3c2895a69ad5eb2c4be16d454f63a6f2f074f351/contracts/FundsAllocator.vy#L36-L81)

**Recommendation**

Although impossible to eliminate the risk completely the likelihood of profitability can be reduced significantly with one of the following changes:

Recommended - Withdraw assets evenly across all adapters. This adds an extra condition to the attack that it must be out of balance prior to the large shift in pricing. The downside is that gas costs are inflated

Simple - Add a withdrawal timelock. This prevents atomic deposits and withdrawals, giving time for the TWAP to self correct and dramatically reduce (or even eliminate) the discrepancy. The downside is the need to make a second transaction to claim the timelocked withdrawal

**Remediation**

**Dev team has stated that although possible, the conditions listed above will never occur under intended management of adapters**
