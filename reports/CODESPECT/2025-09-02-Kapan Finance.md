**Auditors**

[Kalogerone](https://x.com/kalogerone)

[Shaflow01](https://x.com/shaflow01)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/022_CODESPECT_KAPAN_FINANCE.pdf)

# Findings

## High Risk

### [H-01] Certain combinations of instructions can lead to token loss

**Files:** [`RouterGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/RouterGateway.cairo)

**Description:**

Through the RouterGateway contract, users can choose to execute multiple sequential instructions on a single gateway. Before executing the instructions, the contract checks its token balances. After execution, it calculates the balance changes and transfers tokens to the user accordingly. However, since the pre-execution balance includes tokens that are meant to be input (e.g., for deposit or repay), certain combinations of instructions may result in token loss.

```cairo
fn before_send_instructions(...) -> Span<u256> {
    let mut i: usize = 0;
    let mut balancesBefore = array![];
    while i != instructions.len() {
        match instructions.at(i) {
            LendingInstruction::Deposit(deposit) => {
                let basic = *deposit.basic;
                let erc20 = IERC20Dispatcher { contract_address: basic.token };
                if should_transfer {
                    assert(
                        erc20.transfer_from(
                            get_caller_address(), get_contract_address(), basic.amount,
                        ),
                        'transfer failed',
                    );
                }
                assert(erc20.approve(gateway, basic.amount), 'approve failed');
                let balance = erc20.balance_of(get_contract_address());
                balancesBefore.append(balance);
            },
            LendingInstruction::Repay(repay) => {
                let basic = *repay.basic;
                let erc20 = IERC20Dispatcher { contract_address: basic.token };
                if should_transfer {
                    assert(
                        erc20.transfer_from(
                            get_caller_address(), get_contract_address(), basic.amount,
                        ),
                        'transfer failed',
                    );
                }
                assert(erc20.approve(gateway, basic.amount), 'approve failed');
                let balance = erc20.balance_of(get_contract_address());
                balancesBefore.append(balance);
            },
            //...
```

Some combinations of instructions may lead to token loss. For example:

1. repay token1 with 100;
2. withdraw to retrieve 110 of token1;

Since 100 token1 tokens are input into the contract in advance, `balanceBefore = [100, 100]`. During execution, 100 token1 tokens are used to repay the debt, and the withdraw retrieves 110 tokens. `balanceAfter = 110 - 100 = 10`. Only 10 token1 tokens are sent to the user, while the remaining 100 tokens remain locked in the contract.

**Impact:** Certain instructions combinations that use the same token can lead to permanent loss.

**Recommendation:** It is recommended that `after_send_instructions` fetch the token balances before any token inputs occur, and then iterate through the instructions to execute token inputs.

**Status:** Fixed

### [H-02] Vesu Gateway uses the same default pool ID for every withdrawal

**Files:** [`vesu_gateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/vesu_gateway.cairo#L147)

**Description:**

During withdrawals from Vesu, users specify from which pool they want to withdraw using the `context` field in the `Withdraw` struct:

```cairo
fn withdraw(ref self: ContractState, instruction: @Withdraw) {
    // ...
    if instruction.context.is_some() {
        let mut context_bytes: Span<felt252> = (*instruction.context).unwrap();
        let vesu_context: VesuContext = Serde::deserialize(ref context_bytes).unwrap();
        if vesu_context.pool_id != Zero::zero() {
            pool_id = vesu_context.pool_id;
        }
        if vesu_context.position_counterpart_token != Zero::zero() {
            debt_asset = vesu_context.position_counterpart_token;
        }
    }
    // ...
```

Later, contract needs to call the correct vToken address to convert user’s shares to assets. However, the vToken retrieved is not from the user’s specified `pool_id`:

```cairo
fn modify_collateral_for(
    ref self: ContractState,
    pool_id: felt252,
    collateral_asset: ContractAddress,
    debt_asset: ContractAddress,
    user: ContractAddress,
    collateral_amount: i257,
) -> UpdatePositionResponse {
    // ...
    // If this is negative, it means withdraw
    if collateral_amount.is_negative() {
        // @audit doesn't pass user's pool id
        let vtoken = self.get_vtoken_for_collateral(collateral_asset);

        let erc4626 = IERC4626Dispatcher { contract_address: vtoken };
        let requested_shares = erc4626.convert_to_shares(collateral_amount.abs());
        let available_shares = vesu_context.position.collateral_shares;
        assert(available_shares > 0, 'No-collateral');
        // ...
}
```

```cairo
fn get_vtoken_for_collateral(
    self: @ContractState, collateral: ContractAddress,
) -> ContractAddress {
    let vesu_singleton_dispatcher = ISingletonDispatcher {
        contract_address: self.vesu_singleton.read(),
    };
    // @audit uses contract's default pool id
    let poolId = self.pool_id.read();
    let extensionForPool = vesu_singleton_dispatcher.extension(poolId);
    let extension = IDefaultExtensionCLDispatcher { contract_address: extensionForPool };
    extension.v_token_for_collateral_asset(poolId, collateral)
}
```

As a result, the wrong vToken address is used to retrieve information about the user’s available shares and final withdraw amount.

**Impact:** Users are unable to withdraw from their required pools as the transaction will revert if they don’t have any shares in the default `pool_id`. Also, users who have shares in that pool will withdraw assets from that pool even if they specified another pool.

**Recommendation:** Pass users’ `pool_id` to the `get_vtoken_for_collateral(...)` function and use that to retrieve the extension contract address.

**Status:** Fixed

## Medium Risk

### [M-01] Certain instruction combos create negative balancesAfter and will revert

**Files:** [`RouterGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/RouterGateway.cairo)

**Description:**

Through the RouterGateway contract, users can choose to execute multiple sequential instructions on a single gateway. Before executing the instructions, the contract checks its token balances. After execution, it checks its token balances again and calculates the difference from the first check. However, this difference may be a negative value, which will result in the transaction reverting since `balancesAfter` is an array of `u256`.

For example:

1. Deposit 100 of token1;
2. Borrow 100 of token1;

Since 100 tokens are transferred into the contract in advance, `balanceBefore = [100, 100]`.

Looking at `balancesAfter` calculation for these 2 instructions:

```cairo
fn after_send_instructions(
    ref self: ContractState,
    gateway: ContractAddress,
    instructions: Span<LendingInstruction>,
    balancesBefore: Span<u256>,
    should_transfer: bool,
) -> Span<u256> {
    let mut i: usize = 0;
    let mut balancesAfter = array![];
    while i != instructions.len() {
        match instructions.at(i) {
            LendingInstruction::Borrow(borrow) => {
                let basic = *borrow.basic;
                let erc20 = IERC20Dispatcher { contract_address: basic.token };
                if should_transfer {
                    assert(
                        erc20
                            .transfer(
                                basic.user, erc20.balance_of(get_contract_address()),
                            ),
                        'transfer failed',
                    );
                }
                let balance = erc20.balance_of(get_contract_address());

                balancesAfter.append(balance - *balancesBefore.at(i));
            },
            ...

            LendingInstruction::Deposit(deposit) => {
                let basic = *deposit.basic;
                let erc20 = IERC20Dispatcher { contract_address: basic.token };
                let balance = erc20.balance_of(get_contract_address());
                balancesAfter.append(*balancesBefore.at(i) - balance);
            },
            _ => {},
        }
        i += 1;
    }
    balancesAfter.span()
}
```

Here Borrow will first transfer the 100 borrowed tokens to the user and then track the balance of the contract. As a result, `balanceAfter` here will be attempted to be `0 - 100` and transaction will revert.

**Impact:** Certain instruction combinations will have a negative value for `balanceAfter` and will result in the transaction reverting.

**Status:** Fixed

### [M-02] Repay may fail due to insufficient tokens approval

**Files:** [`NostraGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/NostraGateway.cairo), [`RouterGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/RouterGateway.cairo)

**Description:**

If the `repay_all` field is enabled in the repay instruction, it is expected to repay all outstanding debt. However, since this value does not overwrite the `amount` field in the repay instruction within `instructions`.

```cairo
fn before_send_instructions(...) -> Span<u256> {
    //...
    LendingInstruction::Repay(repay) => {
        let basic = *repay.basic;
        let erc20 = IERC20Dispatcher { contract_address: basic.token };
        if should_transfer {
            assert(
                erc20.transfer_from(
                    get_caller_address(), get_contract_address(), basic.amount,
                ),
                'transfer failed',
            );
        }
        assert(erc20.approve(gateway, basic.amount), 'approve failed');
        let balance = erc20.balance_of(get_contract_address());
        balancesBefore.append(balance);
    },
```

**Impact:** It may result in a failed repayment in `before_send_instructions` due to insufficient approval or token transfer. For example, if the `repay_all` flag is enabled but the `repay.amount` is arbitrarily set to a value like 1, then `before_send_instructions` will not transfer a sufficient amount of tokens to the Router, and the Router will not approve enough allowance to the Gateway, resulting in the repay operation failing.

**Recommendation:** In `before_send_instructions`, before transferring and approving tokens for a repay instruction, check if `repay_all` is enabled. If it is, replace the operation amount with the corresponding total debt amount instead of using `repay.amount`.

**Status:** Fixed

### [M-03] The on_flash_loan(...) function lacks a caller verification check

**Files:** [`RouterGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/RouterGateway.cairo)

**Description:**

The `on_flash_loan` function is the flash loan callback function. After receiving the flash loan, the contract executes instruction logic within `on_flash_loan`. However, since there is no check to ensure that the caller is the `flashloan_provider`, a malicious actor can arbitrarily call this function to bypass the `ensure_user_matches_caller` check and execute instructions.

```cairo
fn on_flash_loan(...) {
    assert(sender == get_contract_address(), 'sender mismatch');
    println!("Received flash loan");
    //...
}
```

**Impact:** This could potentially lead to the theft of tokens that users have approved for the Router contract — For example, if a user wants to withdraw from NostraGateway via the router, they need to approve `nibcollateral` to the router. A malicious actor can check for such approvals. Then call `on_flash_loan` to execute the withdraw instruction, and then call the deposit instruction to steal those tokens.

**Recommendation:** It is recommended to check whether the caller is the flashloan provider.

**Status:** Fixed

## Low Risk

### [L-01] Nostra positions are not getting tracked correctly

**Files:** [`NostraGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/NostraGateway.cairo#L312)

**Description:**

In the Nostra protocol there are 2 types of collateral that users can have, Interest Bearing collateral and Non-Interest Bearing collateral. It is possible that users hold both of these debt tokens simultaneously. However, the `get_user_positions(...)` function doesn’t account for this scenario:

```cairo
fn get_user_positions(
    self: @ContractState, user: ContractAddress,
) -> Array<(ContractAddress, felt252, u256, u256)> {
    let mut positions = array![];
    let mut i = 0;
    while i != self.supported_assets.len() {
        let underlying = self.supported_assets.at(i).read();
        let symbol = IERC20SymbolDispatcher { contract_address: underlying }.symbol();

        let debt = self.underlying_to_ndebt.read(underlying);
        let collateral = self.underlying_to_ncollateral.read(underlying);
        let ibcollateral = self.underlying_to_nibcollateral.read(underlying);

        let debt_balance = IERC20Dispatcher { contract_address: debt }.balance_of(user);
        let collateral_raw = IERC20Dispatcher { contract_address: collateral }
            .balance_of(user);

        // @audit User can have collateral in both the collateral and ibcollateral tokens
        let collateral_balance = if collateral_raw == 0 {
            IERC20Dispatcher { contract_address: ibcollateral }.balance_of(user)
        } else {
            collateral_raw
        };
        positions.append((underlying, symbol, debt_balance, collateral_balance));
        i += 1;
    };
    return positions;
}
```

The function checks the user’s Non-Interest Bearing collateral balance and only if it’s 0 it checks for user’s Interest Bearing collateral balance.

**Impact:** The function always returns the balance of 1 type of collateral that the user holds and never the total of both of them. If a user is holding Non-Interest Bearing collateral tokens, his Interest Bearing collateral balance will not be accounted for.

**Recommendation:** Check for both of the balances and add them.

**Status:** Acknowledged

**Client response:** This one won’t be fixed in the release version as the UI does not support the nostra semantics. Users would be required to go through their portal to setup the tokens correctly for transfering debt for the time being.

### [L-02] The on_flash_loan(...) function does not take the repay_all flag into account when handling the repay instruction

**Files:** [`RouterGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/RouterGateway.cairo)

**Description:**

In the `on_flash_loan` function, funds are obtained via flash loan to prepare for debt repayment. The total repay amount is recalculated and the repay instruction is reconstructed. However, since the `repay_all` flag is not taken into account, some of these checks may become invalid.

```cairo
fn on_flash_loan(...) {
    //...
    for protocolInstruction in protocol_instructions {
        for instruction in protocolInstruction.instructions {
            if let LendingInstruction::Repay(repay) = instruction {
                let repay = *repay;
                repay_amounts.append(repay.basic.amount);
                total_repay_amount += repay.basic.amount;
                repay_count += 1;
            }
        }
    }

    // Calculate remaining amount to distribute
    let remaining_amount = amount - total_repay_amount;
    assert(remaining_amount >= 0, 'flashloan insufficient');
    //...
    for instruction in protocol_instructions {
        //...
        basic: BasicInstruction {
            token: repay.basic.token,
            amount: modified_amount,
            user: repay.basic.user,
        },
        repay_all: false, // Force explicit amount
        context: repay.context,

        //...
```

When determining the flash loan amount, if the `repay_all` flag is enabled in the repay instruction, the flash loan amount is treated as the full debt rather than `repay.amount`.

However, in the `on_flash_loan` function, during validation and instruction reconstruction, `repay.amount` is used without handling the `repay_all` flag explicitly. This may lead to invalid or ineffective checks.

**Impact:** If `repay_all` is enabled and `repay.amount` does not match the full debt amount, then the `move_debt` function may fail.

**Recommendation:** The `on_flash_loan` function should consider the `repay_all` flag when accumulating amounts and reconstructing repay instructions.

**Status:** Fixed

### [L-03] get_flash_loan_amount(...) may fail to return the correct amount

**Files:** [`RouterGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/RouterGateway.cairo)

**Description:**

In the `get_flash_loan_amount(...)` function, if a repay instruction has the `repay_all` flag enabled, the function will immediately return the flash loan amount for the gateway. However, the function does not consider whether the other `ProtocolInstructions` also contains a repay instruction.

```cairo
fn get_flash_loan_amount(...) {
    let mut flash_loan_amount: u256 = 0;
    let mut token: ContractAddress = Zero::zero();
    for protocolInstruction in instructions {
        for instruction in protocolInstruction.instructions {
            if let LendingInstruction::Repay(repay) = instruction {
                assert(*repay.basic.amount != 0, 'repay-amount-is-zero');
                if *repay.repay_all {
                    let gateway = ILendingInstructionProcessorDispatcher {
                        contract_address: self.gateways.read(*protocolInstruction.protocol_name),
                    };
                    return (*repay.basic.token, gateway.get_flash_loan_amount(*repay));
                }
                flash_loan_amount += *repay.basic.amount;
                token = *repay.basic.token;
            }
        };
    };
    //...
}
```

**Impact:** If there are multiple `ProtocolInstructions` in the array and one of the repay instructions has the `repay_all` flag enabled, due to the absence of the required amount in other repay instructions, the flash loan amount will be insufficient, causing the `move_debt` instruction to fail.

**Recommendation:** When the `repay_all` flag is enabled, do not consider the flash loan amount required for just a single market.

**Status:** Fixed

## Informational

### [I-01] Revoke excess approvals in after_send_instructions(...)

**Original severity:** Best Practices

**Files:** [`RouterGateway.cairo`](https://github.com/StefanIliev545/kapan/tree/83a7747df3350c2b23d747d52bdd998adbc8812d/packages/snfoundry/contracts/src/gateways/NostraGateway.cairo)

**Description:**

In `after_send_instructions`, there may be cases where not all tokens are used during repayment because `repay.amount > debt`. The unused tokens will be transferred from the router to the user. However, the approval for these tokens granted to the gateway in `before_send_instructions` has not yet been revoked.

```cairo
fn after_send_instructions(
    ref self: ContractState,
    gateway: ContractAddress,
    instructions: Span<LendingInstruction>,
    balancesBefore: Span<u256>,
    should_transfer: bool,
) -> Span<u256> {
    // ...
    LendingInstruction::Repay(repay) => {
        let basic = *repay.basic;
        let erc20 = IERC20Dispatcher { contract_address: basic.token };
        let balance = erc20.balance_of(get_contract_address());
        let diff = *balancesBefore.at(i) - balance;
        balancesAfter.append(diff);
        if basic.amount > diff {
            let erc20 = IERC20Dispatcher { contract_address: basic.token };
            erc20.transfer(basic.user, basic.amount - diff);
        }
    },
    // ...
}
```

**Impact:** A user can manipulate the system to cause the router to grant an excessively large approval to the gateway. For example, if `user1` only has a debt of 10 but sets `repay.amount` to 10,000 during repayment, the unused 9,990 tokens will be returned to `user1`. However, the router’s approval of 9,990 tokens to the gateway remains in place.

While this may not have an immediate visible impact, it is recommended to revoke this unused approval to reduce the potential attack surface in future updates.

**Recommendation:** It is recommended to revoke the router’s approval to the gateway for the refunded tokens.

**Status:** Fixed

### [I-02] Some transfers don’t confirm the return boolean

**Original severity:** Best Practices

**Description:**

Throughout the protocol, `transfer(...)` and `transfer_from(...)` functions returns are checked to be true. However, there are a few instances where this doesn’t happen:

1. `RouterGateway.cairo`:
   - In `after_send_instructions(...)` function under `Withdraw` and `Repay` instructions.
2. `NostraGateway.cairo`:
   - In `repay(...)` function when transferring the `underlying_token`.
3. `vesu_gateway.cairo`:
   - In `repay(...)` function when transferring any `remainder` amount.

**Status:** Fixed
