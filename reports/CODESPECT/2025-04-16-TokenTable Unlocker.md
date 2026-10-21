**Auditors**

JecikPo

Shaflow01

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/011_CODESPECT_TOKENTABLE_SOLANA_UNLOCKER_V2_FOLLOW_UP.pdf)

# Findings

## Medium Risk

### [M-01] The set_default_fee_collector instruction cannot be executed

**Files:** [set_fee_collector.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/2025f68a4d699cc4997c133775f26f2768aba7e6/programs/unlocker-v2-solana/src/instructions/set_fee_collector.rs)

**Description:**

The `set_default_fee_collector` instruction is used to modify the `default_fee_collector`. Since it requires `config.admin` for permission validation, the `config` account should have already been initialized when calling the instruction. However, due to the incorrect assignment of the `init` attribute to the `config` account in the `ctx`, the `set_default_fee_collector` instruction fails to execute successfully.

```rust
#[derive(Accounts)]
#[instruction(_default_fee_collector: Pubkey)]
pub struct SetDefaultFeeCollector<'info> {
    #[account(
        init,
        seeds = [b"config".as_ref()],
        bump,
        payer = authority,
        space = 8 + Config::INIT_SPACE
    )]
    pub config: Account<'info, Config>,
    //...
}
```

**Impact:** The `default_fee_collector` cannot be successfully set.

**Recommendation:** It is recommended to remove the `init` attribute from the `config` account in the `ctx`.

**Status:** Fixed

**Update from TokenTable:** Removed `init` attribute from the config account in the Anchor context in [e2cf5fbc8802845c56d0e0ab48c874c0000ce015](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/e2cf5fbc8802845c56d0e0ab48c874c0000ce015) and added `mut` attribute in [8edb2ab7e2a63c37258b78f365bce2d43db3403f](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/8edb2ab7e2a63c37258b78f365bce2d43db3403f).

## Low Risk

### [L-01] Rent is refunded to the wrong address when the pending_amount_claimable_for_cancelled_actuals account is closed

**Files:** [claim_cancelled_actual_tokens.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/claim_cancelled_actual_tokens.rs#L154)

**Description:**

The `pending_amount_claimable_for_cancelled_actuals` account is created in the `cancel` instruction, with rent paid by `unlocker.owner`, and is closed in the `claim_cancelled_actual_tokens` instruction, with rent mistakenly refunded to the `receipt` address.

```rust
#[derive(Accounts)]
#[instruction(_project_id: String, _preset_id: u64, _actual_id: u64)]
pub struct ClaimCancelledActualTokens<'info> {
    //...
    #[account(
        mut,
        seeds = [
            b"pending_claimable".as_ref(),
            unlocker.key().as_ref(),
            _preset_id.to_le_bytes().as_ref(),
            _actual_id.to_le_bytes().as_ref(),
        ],
        bump,
        close = recipient // NOTE: We are closing the account here.
    )]
    pub pending_amount_claimable_for_cancelled_actuals: Box<
        Account<'info, PendingAmountClaimableForCancelledActualsAccount>
    >,
    //...
}
```

**Impact:** The `recipient` address will receive the additional rent that should have been refunded to `unlocker.owner`.

**Recommendation:** When closing the `pending_amount_claimable_for_cancelled_actuals` account, the rent should be refunded to `unlocker.owner`.

**Status:** Fixed

**Update from TokenTable:** Funds are now returned to `unlocker.owner` in [a71216f80e80fe9c1e6c62ce8d786d3522a572f7](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/a71216f80e80fe9c1e6c62ce8d786d3522a572f7).

### [L-02] The _preset_is_empty function should not consider num_of_unlocks_for_each_linear

**Files:** [create_actual.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/create_actual.rs#L62C14-L62C44)

**Description:**

When `preset.stream` is true, the length of `preset.num_of_unlocks_for_each_linear` is not strictly limited to save rent. Therefore, in a successfully created `preset` account, `preset.num_of_unlocks_for_each_linear` may be 0. As a result, when `_preset_is_empty` checks whether the `preset` account is initialized, it should not consider the `num_of_unlocks_for_each_linear` field.

```rust
#[allow(unused_parens)] // Allowing unused_parens to ignore Prettier formatting
fn _preset_is_empty(preset: anchor_lang::prelude::Account<'_, PresetAccount>) -> bool {
    return (
        preset.linear_bips.len() *
            preset.linear_start_timestamps_relative.len() *
            preset.num_of_unlocks_for_each_linear.len() *
            (preset.linear_end_timestamp_relative as usize) == 0
    );
}
```

**Impact:** An already initialized `preset` account may be mistakenly considered uninitialized, preventing the creation of its corresponding `actual` account.

**Recommendation:** In the `_preset_is_empty` function, the `num_of_unlocks_for_each_linear` field should not be considered when `preset.stream` is true.

**Status:** Fixed

**Update from TokenTable:** We now take `preset.stream` into account when determining if a preset is empty in [b98faf703c27399307598bcb21bf6f647fc143bf](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/b98faf703c27399307598bcb21bf6f647fc143bf).

### [L-03] The creation of pending_amount_claimable_for_cancelled_actuals account may lead to rent loss

**Files:** [cancel.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/cancel.rs#L72)

**Description:**

The `pending_amount_claimable_for_cancelled_actuals` account is always created with rent paid by `unlocker.owner`. However, when `should_wipe_claimable_balance` is true or `delta_amount_claimable` is 0, the account cannot be closed in the `claim_cancelled_actual_tokens` instruction.

```rust
fn _claim_pending_amount(
    ctx: Context<ClaimCancelledActualTokens>,
    project_id: String,
    actual_id: u64,
    batch_id: u64
) -> Result<()> {
    let delta_amount_claimable =
        ctx.accounts.pending_amount_claimable_for_cancelled_actuals.pending_amount_claimable_for_cancelled_actuals;
    require!(delta_amount_claimable != 0, TokenTableError::NotClaimable);
    // ...
}
```

**Impact:** The `pending_amount_claimable_for_cancelled_actuals` account cannot be closed, causing the rent to be locked.

**Recommendation:** It is recommended not to create the `pending_amount_claimable_for_cancelled_actuals` account when `should_wipe_claimable_balance` is true and to directly close the account when `delta_amount_claimable` is 0.

**Status:** Fixed

**Update from TokenTable:** In [9f97ad413026962fd0d544be537d634aebba401c](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/9f97ad413026962fd0d544be537d634aebba401c), automatically close `pending_amount_claimable_for_cancelled_actuals` account in `cancel()` if `should_wipe_claimable_balance` is true or `delta_amount_claimable` is 0. Also, allow `claim_cancelled_actual_tokens()` to close the `pending_amount_claimable_for_cancelled_actuals` account if `delta_amount_claimable` is 0 and the account already exists.

## Informational

### [I-01] Allow the fee_collector to be set arbitrarily during the initialization of the unlocker account

**Files:** [initialize.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/initialize.rs#L24)

**Description:**

In the `initialize` instruction, if `init_fee_account` is set to false, then the check `ctx.accounts.fee_collector.as_ref().unwrap().key() == fee_collector.key()` is skipped. This means that it allows the unlocker owner to initialize any `fee_collector`.

```rust
pub fn initialize(...) -> Result<()> {
    //...
    ctx.accounts.unlocker.fee_collector = fee_collector;

    if init_fee_account {
        // Before we init the fee account, ensure we are calling the expected fee_collector program from
        // the parameters and that all required accounts are provided.
        require!(
            ctx.accounts.fee_collector.is_some() &&
                ctx.accounts.fee.is_some() &&
                ctx.accounts.fee_collector_storage.is_some() &&
                ctx.accounts.fee_collector.as_ref().unwrap().key() == fee_collector.key(),
            TokenTableError::InvalidFeeCollector
        );
```

**Impact:** In the current system, allowing the `fee_collector` account to be set arbitrarily during initialization does not cause any loss, because the no-fee claim, as designed by the protocol, fails due to a constraint in the `ctx`. However, the `unlocker.owner` may have the motivation to initialize the `fee_collector` as `pubkey::default` during initialization. This would enable claims related to that account to be processed without any fees.

**Recommendation:** It is recommended not to allow the `unlocker.owner` to arbitrarily initialize the `fee_collector`.

**Status:** Fixed

**Update from TokenTable:** In [aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461), `fee_collector` program account verification is handled manually. Anchor now expects an `UncheckedAccount<>`, and in all instructions where `fee_collector` can be set, we verify that the provided account matches the instruction parameter value and that the provided account is executable.

### [I-02] Changing the fee_collector to a different program will cause instructions to fail

**Files:** [set_fee_collector.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/set_fee_collector.rs#L49)

**Description:**

The `set_fee_collector` instruction allows to set a different `fee_collector` program for Unlocker’s fee processing capabilities, as per TokenTable’s feedback from the previous audit:

> Added the ability to change the `fee_collector` for a project rather than adding verification in a80d3c31d. We may need to change this address at some point in the future. This function is only callable by an admin (read: one of our wallet accounts), so errors should not happen in setting these values, and we would be able to fix any errors if need be.

The problem arises when the Fee Collector `program_id` is changed and certain instructions which take the `fee_collector` program account are called. The Anchor implementation under the hood will validate the program account against the `program_id` of the `FeeCollector` which was placed there at compile time:

```rust
pub fee_collector: Option<Program<'info, FeeCollector>>,
```

**Impact:** It will not be possible to update the `fee_collector` program account without also updating the entire Unlocker program.

**Recommendation:** Remove the `fee_collector` account from Anchor’s context structs and handle it manually within the instruction code.

**Status:** Fixed

**Update from TokenTable:** In [aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461), `fee_collector` program account verification is handled manually. Anchor now expects an `UncheckedAccount<>`, and in all instructions where `fee_collector` is used, we verify that the provided account matches the expected unlocker’s/airdrop’s `fee_collector` but skip the account executable check, since this would have already been checked when the account was set.

### [I-03] Lack of Option wrapper on fee account

**Files:** [claim.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/claim.rs#L291), [claim_cancelled_actual_tokens.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/claim_cancelled_actual_tokens.rs#L185)

**Description:**

Both claiming instructions - `claim` and `claim_cancelled_actual_tokens` are invoked with few accounts related to fee collection:

- `authority_fee_ata`;
- `fee_collector_storage`;
- `fee_collector_vault`;
- `fee_collector`;
- `fee_token_mint`;
- `fee`;
- `fee_token_program`.

The fee collection mechanism is optional, hence, the design allows skipping them if they are unnecessary through the `Option` wrapper on the account type in the instructions contexts.

The `fee` account however is not:

```rust
/// CHECK: The account is checked in the FeeCollector, not here.
#[account(mut)]
pub fee: UncheckedAccount<'info>,
```

**Impact:** Expected difficulties in building the fee-less transactions as the `fee` account still needs to be provided to the instruction call.

**Recommendation:** Wrap the `fee` account type in `Option`.

**Status:** Fixed

**Update from TokenTable:** As of [78051afb53579a4e6558519000d6c35f510a5533](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/78051afb53579a4e6558519000d6c35f510a5533), the fee collection mechanism is no longer optional. `fee` is a required account and the documented structure here is needed to support the updated fee collection mechanism.

### [I-04] Redundant code

**Files:** [utils.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/utils.rs#L54), [collect_fee.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/fee-collector/src/instructions/collect_fee.rs#L29), [get_fee.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/fee-collector/src/instructions/get_fee.rs#L17)

**Description:**

The protocol contains multiple pieces of redundant code.

1. When obtaining the special configuration `fee_bips` if `fee_bips` equals `BIPS_PRECISION(10000)` it will be set to 0;

```rust
pub fn collect_fee(
    ctx: Context<CollectFee>,
    fee_token: Pubkey,
    _project_id: String,
    token_transferred: u64
) -> Result<u64> {
    //...
    let mut fee_bips = ctx.accounts.fee.bips;

    if fee_bips == BIPS_PRECISION {
        fee_bips = 0;
    }
```

However, this logic is redundant because `fee_bips` is already restricted to not exceed `MAX_FEE(1000)` during configuration.

2. In the `claim()` instruction of the Unlocker program it is imperative to validate the provided `fee_collector` program account if it matches the one held within the unlocker account. This is done directly within the `claim()` inside `claim.rs` file:

```rust
if ctx.accounts.unlocker.fee_collector != Pubkey::default() {
    require!(
        ctx.accounts.unlocker.fee_collector == ctx.accounts.fee_collector.key(),
        TokenTableError::InvalidFeeCollector
    );
}
```

Later in the code, `_claim()` is called, which calls `_after_claim()`, which calls `_charge_fees()`. Inside it we can find the same unnecessary validation:

```rust
require!(
    fee_collector.key() == storage.fee_collector.key(),
    TokenTableError::UnsupportedOperation
);
```

Where the `storage` is in fact the unlocker account. What is more the Airdrop program, does not contain such a redundant check in its `_charge_fees()` equivalent. It is recommended to remove the check from the `_charge_fees()` function.

**Impact:** Redundant code hinders readability and increases deployment costs.

**Recommendation:** It is recommended to optimize the redundant code.

**Status:** Fixed

**Update from TokenTable:** Redundant code for checking `fee_collector` removed in [78051afb53579a4e6558519000d6c35f510a5533](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/78051afb53579a4e6558519000d6c35f510a5533), validating fees removed in [8c443f729b4eefd491832bcac71d996c317f8252](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/8c443f729b4eefd491832bcac71d996c317f8252),

### [I-05] The fee_collector constraint prevents the unlocker from claiming without a fee

**Files:** [claim_cancelled_actual_tokens.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/claim_cancelled_actual_tokens.rs#L179), [claim.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/claim.rs#L285)

**Description:**

The following code indicates that when `unlocker.fee_collector` is set to `pubkey::default()`, the claim will not incur any fees.

```rust
// Fee collector
if ctx.accounts.unlocker.fee_collector != Pubkey::default() {
    require!(
        ctx.accounts.unlocker.fee_collector == ctx.accounts.fee_collector.key(),
        TokenTableError::InvalidFeeCollector
    );
}
```

```rust
pub fn _charge_fees<'info>(...) -> Result<u64> {
    let mut fee_collected: u64 = 0;
    if storage.fee_collector != Pubkey::default() {
        //...
    }

    Ok(fee_collected)
}
```

However, the `fee_collector` constraint in the `claim` and `claim_cancelled_actual_tokens` instructions causes the `unlocker.fee_collector` to fail the constraint check if it is set to `pubkey::default()`, thus preventing the fee-less claim from passing.

```rust
#[account(constraint = fee_collector.key() == unlocker.fee_collector.key())]
pub fee_collector: Program<'info, FeeCollector>,
```

**Impact:** The code’s intended fee-less claim cannot be achieved.

**Recommendation:** Remove the `fee_collector` constraint.

**Status:** Fixed

**Update from TokenTable:** As of [78051afb53579a4e6558519000d6c35f510a5533](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/78051afb53579a4e6558519000d6c35f510a5533), the fee collection mechanism is no longer optional. In [aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461), `fee_collector` is an `UncheckedAccount<>` with manual account verification checks.

### [I-06] Redundant check

**Files:** [claim.rs](https://github.com/CODESPECT-security/011-TokenTable-Solana-UnlockerV2-FollowUp-Merkle/blob/2025f68a4d699cc4997c133775f26f2768aba7e6/programs/unlocker-v2-solana/src/instructions/claim.rs#L285), [claim_cancelled_actual_tokens.rs](https://github.com/CODESPECT-security/011-TokenTable-Solana-UnlockerV2-FollowUp-Merkle/blob/2025f68a4d699cc4997c133775f26f2768aba7e6/programs/unlocker-v2-solana/src/instructions/claim_cancelled_actual_tokens.rs#L184)

**Description:**

The `claim` and `claim_cancelled_actual_tokens` instructions contain redundant checks for `fee_collector`. The `fee_collector` is checked in the `ctx` and then checked again in the execution logic.

```rust
/// CHECK: Checked in the function call.
#[account(constraint = fee_collector.key() == airdrop.fee_collector.key())]
pub fee_collector: UncheckedAccount<'info>,

// ...

pub fn claim_cancelled_actual_tokens(...) -> Result<()> {
    // Fee collector
    require!(
        ctx.accounts.unlocker.fee_collector == ctx.accounts.fee_collector.key(),
        TokenTableError::InvalidFeeCollector
    );

pub fn claim(...) -> Result<()> {
    // Fee collector
    require!(
        ctx.accounts.unlocker.fee_collector == ctx.accounts.fee_collector.key(),
        TokenTableError::InvalidFeeCollector
    );
```

**Impact:** Redundant checks increase the execution overhead of the transaction call.

**Recommendation:** It is recommended to remove the redundant checks.

**Status:** Fixed

**Update from TokenTable:** Redundant checks removed in [1aed8dad5fca73dd7e7b3d2a666c939a39a37be6](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/1aed8dad5fca73dd7e7b3d2a666c939a39a37be6).
