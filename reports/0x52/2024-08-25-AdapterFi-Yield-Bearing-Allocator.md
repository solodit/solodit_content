**Auditor**

[@IAm0x52](https://twitter.com/IAm0x52)

# Findings

## High Risk

### [H-01] Compromised or faulty neutral adapter will bypass loss protection and continue to receive funding

**Details**

[YieldBearingAssetFundsAllocator.vy#L407-L417](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L407-L417)

    should_we_block_adapter : bool = False
    if _balance_adapter.current < _balance_adapter.last_value:
        # There's an unexpected loss of value. Let's try to empty this adapter and stop
        # further allocations to it by setting the ratio to 0 going forward.
        # This will not necessarily result in any "leftovers" unless withdrawing the full
        # balance of the adapter is limited by max_withdraw limits below.
        _balance_adapter.ratio = 0
        should_we_block_adapter = True

    target : uint256 = _ratio_value * _balance_adapter.ratio
    delta : int256 = convert(target, int256) - convert(_balance_adapter.current, int256)

The current methodology for blocking adapters functions by setting the ratio for the affected adapter to 0. For normal adapters this triggers an immediate withdraw of all funds and prevent any additional deposits.

[YieldBearingAssetFundsAllocator.vy#L188-L209](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L188-L209)

        if _adapter_states[max_delta_deposit_pos].max_deposit < convert(_vault_balance, int256):
            _adapter_states[max_delta_deposit_pos].delta = _adapter_states[max_delta_deposit_pos].max_deposit
            adapter_txs.append( BalanceTX({qty: _adapter_states[max_delta_deposit_pos].max_deposit,
                                            adapter: _adapter_states[max_delta_deposit_pos].adapter}) )

            # # Do we have a neutral adapter to take the rest?
            if neutral_adapter_pos != MAX_ADAPTERS:
                adapter_txs.append( BalanceTX({qty: convert(_vault_balance, int256) - _adapter_states[max_delta_deposit_pos].max_deposit,
                                                adapter: _adapter_states[neutral_adapter_pos].adapter}) )

        # Great - it can take the whole thing.
        else:
            adapter_txs.append( BalanceTX({qty: convert(_vault_balance, int256),
                                            adapter: _adapter_states[max_delta_deposit_pos].adapter}) )

    # No normal adapters available to take our deposit.
    else:
        # Do we have a neutral adapter to take the rest?
        if neutral_adapter_pos != MAX_ADAPTERS:
            assert convert(_vault_balance, int256) <= _adapter_states[neutral_adapter_pos].max_deposit, "Over deposit on neutral vault!"
            adapter_txs.append( BalanceTX({qty: convert(_vault_balance, int256),
                                            adapter: _adapter_states[neutral_adapter_pos].adapter}) )

As seen above, the neutral adapter functions different and is able to receive funds with a ratio of 0. As a result, a faulty or compromised adapter will still receive funds from the vault. By manipulating withdraws and deposits to the vault an attacker could take advantage of this to repeatedly receive vault deposits to the compromised adapter, draining the vault.

**Lines of Code**

[AdapterVault.vy#L1111-L1122](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/AdapterVault.vy#L1111-L1122)

**Recommendation**

In the case of the neutral adapter, setting the ratio to 0 is not enough and it needs to be disabled in another way.

**Remediation**

Acknowledged. Although a concern for generic ERC4626 adapters, devs have stated the adapter that will be used is very basic and functionality cannot fail in this manner.

## Medium Risk

### [M-01] Withdraw limits are not properly considered during balancing transactions and can lead to vault DOS

**Details**

When withdrawing from an adapter, current balance and max withdraw must be considered. If either is exceeded then the rebalance transaction will revert during execution and will revert the entire transaction causing a vault DOS.

[YieldBearingAssetFundsAllocator.vy#L244-L254](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L244-L254)

    if shortfall > 0 and min_delta_withdraw_pos != MAX_ADAPTERS:
        if _adapter_states[min_delta_withdraw_pos].current > shortfall:
            # Got it all!
            adapter_txs.append( BalanceTX({qty: convert(shortfall, int256) * -1,
                                            adapter: _adapter_states[min_delta_withdraw_pos].adapter}) )
            shortfall = 0
        else:
            # Got some...
            adapter_txs.append( BalanceTX({qty: convert(_adapter_states[min_delta_withdraw_pos].current, int256) * -1,
                                            adapter: _adapter_states[min_delta_withdraw_pos].adapter}) )
            shortfall -= _adapter_states[min_delta_withdraw_pos].current

Above we see that only the current balance of the adapter is considered. This leads and edge case in which `current balance > shortfall > max withdraw`. Although the adapter has enough funds to cover the withdrawal, the withdrawal will revert during execution due to the amount being higher than the max withdraw amount.

[YieldBearingAssetFundsAllocator.vy#L290-L299](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L290-L299)

    if convert(shortfall, int256) > max(_adapter_states[i].max_withdraw, _adapter_states[i].max_withdraw):
        # Got it all!
        adapter_txs.append( BalanceTX({qty: convert(shortfall, int256) * -1,
                            adapter: _adapter_states[i].adapter}) )
        shortfall = 0
    else:
        # Got some...
        adapter_txs.append( BalanceTX({qty: convert(_adapter_states[i].current, int256) * -1,
                            adapter: _adapter_states[i].adapter}) )
        shortfall -= _adapter_states[i].current

Here we have another similar situation but in reverse. The max withdraw is checked but then current balance or shortfall is withdrawn. This leads to the following two edge cases. The first is `max withdraw > shortfall > current balance` which will result in failure to cover the shortfall during execution. The other is `shortfall > current > max withdraw`. In this case it will attempt to withdraw the current balance which is higher than the max withdraw causing it to revert.

**Lines of Code**

[YieldBearingAssetFundsAllocator.vy#L245-L254](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L245-L254)

[YieldBearingAssetFundsAllocator.vy#L268-L282](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L268-L282)

[YieldBearingAssetFundsAllocator.vy#L285-L299](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L285-L299)

**Recommendation**

Whenever checking these bounds first construct a variable defined as:

    min(_adapter_states.max_withdraw * -1, _adapter_states.current)

Whenever checking limits and formulating withdrawal amounts, use this variable to account for both current balance and max withdraw at the same time.

**Remediation**

Fix in commit [5996492](https://github.com/adapter-fi/AdapterVault/commit/5996492b8bdfbc1b544b4e12e0a7966c381cba50) as recommended.

### [M-02] `YieldBearingAssetFundsAllocator#_allocate_balance_adapter_tx` fails to properly apply `ADAPTER_BREAKS_LOSS_POINT`

**Details**

[YieldBearingAssetFundsAllocator.vy#L407-L414](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L407-L414)

    should_we_block_adapter : bool = False
    if _balance_adapter.current < _balance_adapter.last_value:
        # There's an unexpected loss of value. Let's try to empty this adapter and stop
        # further allocations to it by setting the ratio to 0 going forward.
        # This will not necessarily result in any "leftovers" unless withdrawing the full
        # balance of the adapter is limited by max_withdraw limits below.
        _balance_adapter.ratio = 0
        should_we_block_adapter = True

When allocating funds to adapters, the above code checks if the current value is lower than the previous value. If there is even a single wei below it will block the adapter. This is especially problematic for adapters that are based on the market prices (such as the pendle adapter) as they will be frequently disabled.

`ADAPTER_BREAKS_LOSS_POINT` was present in previous version but is notably absent in this one.

**Lines of Code**

[YieldBearingAssetFundsAllocator.vy#L403-L434](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L403-L434)

**Recommendation**

Reintroduce `ADAPTER_BREAKS_LOSS_POINT` and only block the adapter if losses exceed that margin.

**Remediation**

Fix in commit [b344fd3](https://github.com/adapter-fi/AdapterVault/commit/b344fd3c129afdd42dcdbc7026b1a0c39c6ac29c) as recommended.

### [M-03] Incorrect inequality in `YieldBearingAssetFundsAllocator#_generate_balance_txs` can lead to withdrawal DOS

**Details**

[YieldBearingAssetFundsAllocator.vy#L273-L277](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L273-L277)

    if convert(shortfall, int256) > max(_adapter_states[i].max_withdraw, _adapter_states[i].max_withdraw):
        # Got it all!
        adapter_txs.append( BalanceTX({qty: convert(shortfall, int256) * -1,
                            adapter: _adapter_states[i].adapter}) )
        shortfall = 0

In the above check, shortfall is compared against max_withdraw to determine if the adapter is able to cover the shortfall. The problem is that shortfall is a positive number (higher signifies greater) while max_withdraw is a negative number (lower signifies greater). Since shortfall > 0 and max_withdraw < 0, this will always return true. In the event that `|max_withdraw| < shortfall` or `current < shortfall` the withdrawal will revert during execution, which will DOS withdrawals from the vault.

**Lines of Code**

[YieldBearingAssetFundsAllocator.vy#L273-L277](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L273-L277)

[YieldBearingAssetFundsAllocator.vy#L290-L299](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L290-L299)

**Recommendation**

abs(max_withdraw) should be used when comparing with shortfall

**Remediation**

Fix in commit [5996492](https://github.com/adapter-fi/AdapterVault/commit/5996492b8bdfbc1b544b4e12e0a7966c381cba50) as recommended.

### [M-04] Blocked adapters will cause full rebalances to be DOS'd

**Details**

[YieldBearingAssetFundsAllocator.vy#L117-L136](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L117-L136)

    for i in range(MAX_ADAPTERS):
        rtx : BalanceAdapter = _blocked_adapters[i]
        if rtx.adapter == empty(address): break
        assert tx_pos < MAX_ADAPTERS, "Too many transactions #10!"
        result_txs[tx_pos] = BalanceTX({qty: rtx.delta, adapter: rtx.adapter}) <- @audit blocked adapter tx #1
        result_blocked[tx_blocked] = rtx.adapter
        tx_pos += 1
        tx_blocked += 1

    for i in range(MAX_ADAPTERS):
        rtx : BalanceAdapter = _adapter_states[i]
        if rtx.adapter == empty(address): break
        assert tx_pos < MAX_ADAPTERS, "Too many transactions #20!"

        # Deposits are set aside as withdraws must complete first.
        if rtx.delta > 0 and rtx.delta >= convert(_min_proposer_payout, int256) and not _withdraw_only:
            deposits_last.append(BalanceTX({qty: rtx.delta, adapter: rtx.adapter}))
        elif rtx.delta < 0:
            result_txs[tx_pos] = BalanceTX({qty: rtx.delta, adapter: rtx.adapter}) <- @audit blocked adapter tx repeat
            tx_pos += 1

When adapters lose value they are blocked and the contract attempts to withdraw all funds. In the code above we see that for a blocked adapter one `result_tx` is added during the loop through blocked adapters but then when cycling through `adapter_states` in the next block, a second `result_tx` is added for the same adapter.

When these txs are processed, the rebalance will inevitably revert leading to a DOS of the vault.

**Lines of Code**

[YieldBearingAssetFundsAllocator.vy#L108-L144](https://github.com/adapter-fi/AdapterVault/blob/a5172a63abedd4e19c9e1a17b06d579760b4aba6/contracts/YieldBearingAssetFundsAllocator.vy#L108-L144)

**Recommendation**

Adapters that are present in blocked_adapters should be skipped during the `_adapter_states` loop.

**Remediation**

Fix in commit [165cbe3](https://github.com/adapter-fi/AdapterVault/commit/165cbe34d11da444ae468693ae1d20ce9bf5c53c) as recommended.
