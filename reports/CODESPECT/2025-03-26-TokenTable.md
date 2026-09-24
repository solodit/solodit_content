**Auditors**

JecikPo

Shaflow01

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/009_CODESPECT_TOKENTABLE_SOLANA_UNLOCKER_V2.pdf)

# Findings

## Medium Risk

### [M-01] Arithmetic overflow in claim instruction

**Files:** [`claim.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/claim.rs#L246)

**Description:**

The `claim` instruction uses the internal function `_simulate_amount_claimable()` to calculate the amount of claimable tokens based on given actual and preset accounts.

When calculating `updated_amount_claimed`, which represents the new amount of claimed tokens, it multiplies two values, which are both 9 decimals big:

```rust
updated_amount_claimed =
  (updated_amount_claimed * actual.total_amount) / BIPS_PRECISION / TOKEN_PRECISION;
```

Provided that the `total_amount` is big enough, the calculation may overflow as it needs to fit into the `u64` variable size before division.

**Impact:** Under certain big enough values representing token amounts, claims will fail.

**Recommendation:** Make the calculations on at least `u128` variable sizes.

**Status:** Fixed

**Update from TokenTable:** Switched to `u128` for calculations in [d7357087b3a46d0c6eed3c240239d35d2f7ddc15](https://github.com/EthSign/tokentable-unlocker-solana/tree/d7357087b3a46d0c6eed3c240239d35d2f7ddc15).

### [M-02] Collision between PendingAmountClaimableForCancelledActualsAccount may lead to stolen funds

**Files:** [`claim.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/claim.rs)

**Description:**

The `PendingAmountClaimableForCancelledActualsAccount` account is used to store amount assets that a recipient of a cancelled Actual can still claim. This account is provided to the `claim` instruction, and a recipient should be able to claim it.

The problem is that the account seeds don’t contain the `preset_id` where the claim was present. Hence the following scenario is possible:

1. Unlocker is created with two Presets;
2. For each Preset one Actual is created: Actual-1 and Actual-2, for recipient-1 and recipient-2. As those Actuals are created for different Presets, they share the same `actual_id` (e.g. 1);
3. Unlocker’s owner cancels Actual-1 and hence `PendingAmountClaimableForCancelledActualsAccount` is created using `actual_id` 1 as its seed;
4. The recipient-2 can call `claim`, and provide the above `PendingAmountClaimableForCancelledActualsAccount` account to claim the amount stored there, which does not belong to him;

**Impact:** Cancelled amounts can be claimed by other recipients.

**Recommendation:** Add the `preset_id` to the `PendingAmountClaimableForCancelledActualsAccount` seed.

**Status:** Fixed

**Update from TokenTable:** Added `preset_id` to the list of seeds used when deriving `PendingAmountClaimableForCancelledActualsAccount` in [3510bac21534d846d7f117ac89b11d83478a529f](https://github.com/EthSign/tokentable-unlocker-solana/tree/3510bac21534d846d7f117ac89b11d83478a529f).

### [M-03] Incorrect Preset input parameters validation

**Files:** [`create_preset.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/8dc0aa15e5cf78611a7bf7a5aa5ab1ca793ac108/programs/unlocker-v2-solana/src/instructions/create_preset.rs)

**Description:**

The `create_preset` instruction is called to create a `PresetAccount` account which holds the airdrop schedule information. The instruction caller provides a `Preset` struct which is used to populate values for the `PresetAccount`. The input `Preset` struct is validated using `_preset_has_valid_format()` function. The function ensures the following:

- `preset.linear_bips` vector sum of values is equal to `BIPS_PRECISION` const;
- All vectors must be of the same length;
- The `preset.linear_start_timestamps_relative` vector last element must be smaller than `preset.linear_end_timestamp_relative`;
- Each `preset.num_of_unlocks_for_each_linear` element value must be smaller than relative timestamp spacing;

The first three conditions from the above list are enclosed within the following conditional instruction:

```rust
if
  !(total == BIPS_PRECISION) &&
  preset.linear_bips.len() == preset.linear_start_timestamps_relative.len() &&
  preset.linear_start_timestamps_relative[preset.linear_start_timestamps_relative.len() - 1] <
    preset.linear_end_timestamp_relative &&
  preset.num_of_unlocks_for_each_linear.len() == preset.linear_start_timestamps_relative.len()
{
  return false;
}
```

We can see that the logical operator `!` is not applied correctly, and hence it is possible that all the remaining checks after BIPS verification can be bypassed.

**Impact:** A preset account could be created with incorrect values. Depending on which specific values are incorrect (several are possible), it may become impossible to claim from the affected Preset.

**Recommendation:** Fix the logical expression.

**Status:** Fixed

**Update from TokenTable:** Fixed the parenthesis location in [c5a96b7c73e0b37678436b4bce9adf9e3c8bf78f](https://github.com/EthSign/tokentable-unlocker-solana/tree/c5a96b7c73e0b37678436b4bce9adf9e3c8bf78f).

### [M-04] The pending_amount_claimable accumulated in the cancel instruction cannot be claimed

**Files:** [`cancel.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/cancel.rs#L37)

**Description:**

In the `cancel()` instruction, if `should_wipe_claimable_balance` is false, the unclaimed rewards of the closed `ActualAccount` will be accumulated into the `PendingAmountClaimableForCancelledActualsAccount` account.

```rust
fn _cancel(
  ctx: Context<Cancel>,
  actual_id: u64,
  should_wipe_claimable_balance: bool,
  batch_id: u64
) -> Result<()> {
  let delta_amount_claimable = _calculate_amount_claimable(
    ctx.accounts.actual.clone(),
    ctx.accounts.preset.clone()
  )?.delta_amount_claimable;

  if !should_wipe_claimable_balance {
    ctx.accounts.pending_amount_claimable_for_cancelled_actuals.pending_amount_claimable_for_cancelled_actuals +=
      delta_amount_claimable;
  }
  //...
}
```

However, the user is unable to claim the accumulated balance in the `PendingAmountClaimableForCancelledActualsAccount` because the balance can only be claimed through the `claim()` and `delegate_claim()` instructions, both of which require the `ActualAccount` that has been closed by the `cancel()` instruction. Additionally, the corresponding `ActualAccount` cannot be re-init.

**Impact:** The user may lose the unclaimed tokens from before the execution of the cancel instruction.

**Recommendation:** It is recommended to separate the logic for claiming the tokens accumulated in `PendingAmountClaimableForCancelledActualsAccount` from the `claim()` and `delegate_claim()` instructions.

**Status:** Fixed

**Update from TokenTable:** Restructured the claiming process in [f851e215e19904ea9a1d07cfa44b9cc34afce11e](https://github.com/EthSign/tokentable-unlocker-solana/tree/f851e215e19904ea9a1d07cfa44b9cc34afce11e). `delegate_claim()` has been combined into `claim()` with logic changes pertaining to authority, recipient, and `recipient_ata` accounts. Split `claim()` logic into `claim()` and `claim_cancelled_actual_tokens()` in order to allow claiming the previously orphaned tokens present in `PendingAmountClaimableForCancelledActualsAccount`.

## Low Risk

### [L-01] Claims fail when different Token program is used for fees and claimable tokens

**Files:** [`collect_fee.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/fee-collector/src/instructions/collect_fee.rs#L97), [`claim.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/claim.rs#L335)

**Description:**

The claim instruction is used to make two token transfers:

- The claimable token to the recipient from the unlocker vault;
- The fee token from the recipient to the protocol fee-collector vault;

The problem is that in the claim instruction, those two transfers are handled by a single program account specified in the instruction context. If those two tokens are of two different kinds, i.e. Token and Token2022, then it will be impossible to claim the tokens as the transfer of the other token will fail due to an incorrect program applied in the CPI.

**Impact:** A certain token/fee-token combination won’t be possible if they are of two different types. The protocol owner can always adjust the fee-token to match the token in an already set up unlocker.

**Recommendation:** Provide two Token Program account inputs in the claim context definition. One for the distributed token and the other for the fee token. The CPI for the `collect_fee` should use the fee token program account.

**Status:** Fixed

**Update from TokenTable:** Added `fee_token_program` account to `claim()` and `claim_cancelled_actual_tokens()` which is used in the CPI to the fee collector program in [22e51755ffe9cf1ca3a3e1a5c56bef2d80a618aa](https://github.com/EthSign/tokentable-unlocker-solana/tree/22e51755ffe9cf1ca3a3e1a5c56bef2d80a618aa).

### [L-02] Lack of token_mint validation in the Deposit instruction

**Files:** [`deposit.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/deposit.rs)

**Description:**

The Deposit instruction in the Unlocker program is used by the owner of the unlocker account to deposit the initial amount of tokens for later claiming. When this instruction is called for the first time a new vault account is created for a specified `token_mint` account to hold the assets tied to the specific unlocker account (identified by `_project_id`). The Deposit instruction can be called by anyone as there is no validation of the owner.

A malicious user could call the Deposit instruction and provide a junk `token_mint` account and hence a vault tied to that unlocker will be created. Further deposits of the intended SPL token become impossible because the vault is already created and the entire unlocker becomes useless.

**Impact:** Malicious user could DoS a created unlocker account before the initial funds are deposited. No funds are lost, but another unlocker needs to be created for the owner by the program’s owner.

**Recommendation:** Validate the provided `token_mint` against the `unlocker.project_token` either through Anchor context definition (recommended for clarity) or inside the handler.

**Status:** Fixed

**Update from TokenTable:** Added `token_mint` constraint in [6bccb7d0ebefd3cdaaa82278c9b0567c4f01f13e](https://github.com/EthSign/tokentable-unlocker-solana/tree/6bccb7d0ebefd3cdaaa82278c9b0567c4f01f13e).

### [L-03] Missing check whether fee_token and token_mint are consistent in the init_fee_token instruction

**Files:** [`init_fee_token.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/fee-collector/src/instructions/init_fee_token.rs)

**Description:**

In the `init_fee_token` instruction, there is no check to ensure that `fee_token` and `token_mint` are consistent. This may result in the vault account’s seed not matching the corresponding mint. Causing the vault account to be unable to properly collect fees.

```rust
pub struct InitFeeToken<'info> {
  //...
  #[account(
    init_if_needed,
    payer = authority,
    seeds = [b"vault".as_ref(), fee_token.as_ref()],
    bump,
    token::mint = token_mint,
    token::authority = storage
  )]
  pub vault: Option<InterfaceAccount<'info, TokenAccount>>,
  pub token_mint: Option<InterfaceAccount<'info, Mint>>,
  //...
}
```

**Impact:** When `fee_token` and `token_mint` are inconsistent, the vault account created by the `init_fee_token` instruction will be unable to properly receive fees.

**Recommendation:** It is recommended to check that `fee_token` and `token_mint` are the same.

**Status:** Fixed

**Update from TokenTable:** Added constraint to `token_mint` in [eb09b6b3d1da4774d72cd75b99e7eb9cf86bb7e9](https://github.com/EthSign/tokentable-unlocker-solana/tree/eb09b6b3d1da4774d72cd75b99e7eb9cf86bb7e9).

### [L-04] Missing the is_withdrawable check in the withdraw_deposit(...) instruction

**Files:** [`withdraw_deposit.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/withdraw_deposit.rs)

**Description:**

In the Unlocker account, `is_withdrawable` is set to control whether the unlocker owner is allowed to withdraw tokens from the vault. However, the `withdraw_deposit()` instruction does not check the `is_withdrawable` field, causing the unlocker owner’s withdrawals to be always allowed.

**Impact:** The unlocker owner’s withdrawals will not be controlled by the `is_withdrawable` field.

**Recommendation:** Add the `is_withdrawable` check in the `withdraw_deposit()` instruction.

**Status:** Fixed

**Update from TokenTable:** Added `is_withdrawable` check to the `withdraw_deposit()` function in commit [e8521cef](https://github.com/EthSign/tokentable-unlocker-solana/tree/e8521cef9ff2aa88546d3ceba27c414178863c36).

### [L-05] Withdrawals from Fee Collector become impossible after renouncing ownership

**Files:** [`renounce_ownership.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/renounce_ownership.rs)

**Description:**

The `renounce_ownership` instruction in the Fee Collector program is available for the owner. The following instructions become impossible to call when ownership is renounced:

- `init_fee_token`;
- `set_custom_fee_bips`;
- `set_custom_fee_fixed`;
- `set_default_fee`;
- `transfer_ownership`;
- `withdraw`;

There are two major consequences to the protocol when ownership is renounced:

- The whole protocol will operate normally for existing unlockers, however, new ones could not have their own fee accounts, hence all fees will fallback to the default fee: `storage.default_fees_bips`;
- It will be impossible to withdraw fees;

**Impact:** Withdrawals shall be blocked if ownership is renounced.

**Recommendation:** Specify what the ownership renouncement should block from happening or remove the instruction for safety reasons.

**Status:** Fixed

**Update from TokenTable:** Removed `renounce_ownership()` in [e8c372dbee51e66a152f6683d002dbc2e297a07b](https://github.com/EthSign/tokentable-unlocker-solana/tree/e8c372dbee51e66a152f6683d002dbc2e297a07b).

## Informational

### [I-01] Miscalculated PresetAccount size

**Files:** [`preset.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/models/preset.rs#L22)

**Description:**

When creating a `PresetAccount`, the size of the `PresetAccount` is initialized based on the `Preset` struct passed in, using the `calculate_size()` function.

```rust
#[account(
  //...
  space = 8 + _preset.clone().calculate_size()
)]
pub preset: Account<'info, PresetAccount>,
```

The calculation seems to be implemented incorrectly, as the total size of non-vector items should be `linear_end_timestamp_relative + next_actual_id + stream + preset_id = 25` instead of 32.

```rust
pub fn calculate_size(self) -> usize {
  let mut size = 0;
  // @audit incorrect size
  size += 32; // Takes care of all non-vector items
  size += 4 + 8 * self.linear_start_timestamps_relative.len();
  size += 4 + 8 * self.linear_bips.len();
  size += 4 + 8 * self.num_of_unlocks_for_each_linear.len();
  size += 4 + self.project_id.len();

  size
}
```

**Impact:** This would result in unnecessary rent wastage.

**Recommendation:** Calculate account space using the correct size.

**Status:** Fixed

**Update from TokenTable:** Updated base size to 25 in [85e56b4993b46006e5cb36e08df56f49ac4a535e](https://github.com/EthSign/tokentable-unlocker-solana/tree/85e56b4993b46006e5cb36e08df56f49ac4a535e).

### [I-02] Unnecessary account ownership validation

**Files:** [`initialize.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/initialize.rs), [`deploy.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/deploy.rs)

**Description:**

Initialize and Deploy instructions are used the create new accounts: unlocker and config respectively. Those instruction’s context definitions contain the `init` attribute for the above accounts. Indicating that the accounts must not exist prior to calling those instructions. In the instruction handler, however exists an additional check that validates if those are new accounts; however, this check will always be true because the Anchor context definition ensures that they did not exist, hence the below require statement is unnecessary:

```rust
require!(ctx.accounts.config.admin == Pubkey::default(), TokenTableError::AlreadyDeployed);
```

**Impact:** No impact on code functionality.

**Recommendation:** Remove those require statements.

**Status:** Fixed

**Update from TokenTable:** Removed `require!()` statements in [ade803ab2628944df96a406546d5573692a13469](https://github.com/EthSign/tokentable-unlocker-solana/tree/ade803ab2628944df96a406546d5573692a13469).

### [I-03] In some cases the num_of_unlocks_for_each_linear vector may waste rent

**Original severity:** Best Practices

**Files:** [`create_preset.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/create_preset.rs#L64)

**Description:**

When the `PresetAccount` enables the stream configuration, the `num_of_unlocks_for_each_linear` field becomes ineffective and can be set to an empty vector.

```rust
if preset.stream {
  num_of_unlocks_for_incomplete_linear = latest_incomplete_linear_duration;
} else {
  num_of_unlocks_for_incomplete_linear =
    preset.num_of_unlocks_for_each_linear[latest_incomplete_linear_index as usize];
}
```

However, during the `create_preset()` function, the `num_of_unlocks_for_each_linear` field must always be set to the same length as the `linear_start_timestamps_relative` vector, regardless of the situation.

```rust
if
  !(total == BIPS_PRECISION) &&
  preset.linear_bips.len() == preset.linear_start_timestamps_relative.len() &&
  preset.linear_start_timestamps_relative[preset.linear_start_timestamps_relative.len() - 1] <
    preset.linear_end_timestamp_relative &&
  preset.num_of_unlocks_for_each_linear.len() == preset.linear_start_timestamps_relative.len()
{
```

**Impact:** This will cause some rent waste for certain fields in the `PresetAccount` when the stream is enabled.

**Recommendation:** It is recommended that when the stream is enabled, the `num_of_unlocks_for_each_linear` field can be left empty. Additionally, the `_preset_is_empty()` function should be modified to remove the `num_of_unlocks_for_each_linear` field from it, based on the stream value.

**Status:** Fixed

**Update from TokenTable:** If `preset.stream` is set to true, we no longer enforce that `preset.num_of_unlocks_for_each_linear` has the same length as `preset.linear_start_timestamps_relative` in [cd8e53d03b91c20af3153fb8feb6cbf1988d7bea](https://github.com/EthSign/tokentable-unlocker-solana/tree/cd8e53d03b91c20af3153fb8feb6cbf1988d7bea).

### [I-04] Lack of two step ownership transfers

**Original severity:** Best Practices

**Files:** [`transfer_ownership.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/transfer_ownership.rs), [`transfer_program_admin.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/transfer_program_admin.rs)

**Description:**

The program provides instructions for transferring ownership of the program admin and for the individual unlocker accounts (Projects). Those instructions allow transfer of the ownership to any arbitrary account. Currently, best practice dictates that there should be some form of control over who the ownership is transferred to. This, in a classical Solidity implementation, would involve two separate calls. On Solana, this can be done in a simplified way - there would still be a single instruction however there should be a requirement added to the instructions’ contexts that the new address should also be a Signer.

**Impact:** Accidental loss of control over the protocol or unlocker accounts.

**Recommendation:** Ensure Signer type of account for new owner accounts.

**Status:** Acknowledged

**Update from TokenTable:** Changed to two-step ownership transfers (using two transaction signers) in [4a176e0a](https://github.com/EthSign/tokentable-unlocker-solana/tree/4a176e0a4cfbaf9fa5aaa20b8b57e122e72d7cb3). This logic was amended in [89889a1fef4fe1f879818eb8e2b8a6d80e8b76de](https://github.com/EthSign/tokentable-unlocker-solana/tree/89889a1fef4fe1f879818eb8e2b8a6d80e8b76de), allowing the second signer to be null in calls to `transfer_ownership()` and `transfer_program_admin()`. In this case, the new owner/admin must call `receive_ownership()` or `receive_program_admin()`, respectively, to complete the permission transfer. After additional internal discussion, we have elected to roll back the two-step ownership transfer modifications for `transfer_ownership()` in [3dbd4b333432893f482acc7dc12b947c54ce324f](https://github.com/EthSign/tokentable-unlocker-solana/tree/3dbd4b333432893f482acc7dc12b947c54ce324f). The added complexity does not justify the risks in typical usage (limited frontend verification is performed).

### [I-05] Missing check for fee_collector in Unlocker account initialization

**Original severity:** Best Practices

**Files:** [`initialize.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/initialize.rs#L22)

**Description:**

The Unlocker account stores the `fee_collector` field, which is initialized in the `initialization()` instruction and cannot be modified afterward. If an incorrect `fee_collector` is provided during initialization, it will be permanently set, preventing any future changes.

**Impact:** For this Unlocker is will be impossible to claim tokens through the `claim()` instruction.

**Recommendation:** It is recommended to check whether `fee_collector` is the expected account address during the `initialization()` instruction.

**Status:** Fixed

**Update from TokenTable:** Added the ability to change the `fee_collector` for a project rather than adding verification in [a80d3c31d](https://github.com/EthSign/tokentable-unlocker-solana/tree/a80d3c31d16dc5e02c8995dac62d4bc0e7b0bf54). We may need to change this address at some point in the future. This function is only callable by an admin (read: one of our wallet accounts), so errors should not happen in setting these values, and we would be able to fix any errors if need be.

### [I-06] Missing check to verify if the fields stored in the Account are consistent with the seed

**Original severity:** Best Practices

**Files:** [`create_preset.rs`](https://github.com/EthSign/tokentable-unlocker-solana/blob/7516b8c86cb305f9d9eb3ac77e7fcd7c6b60cc2f/programs/unlocker-v2-solana/src/instructions/create_preset.rs#L40)

**Description:**

The `PresetAccount` stores the `project_id` field, and the `ActualAccount` stores both the `project_id` and `actual_id` fields. The protocol does not check whether these stored fields match the fields used to generate the seed during account initialization.

**Impact:** No on-chain impact, but off-chain parsing may result in mismatched data in the account.

**Recommendation:** It is recommended to check whether the relevant fields in the `Preset` struct and `Actual` struct match.

**Status:** Fixed

**Update from TokenTable:** Added checks to ensure relevant fields match in Preset and Actual structs in [2079107e](https://github.com/EthSign/tokentable-unlocker-solana/tree/2079107edcd5d2294ac969945b86b543a256283e).

**Update from CODESPECT:** An unnecessary duplicated `project_id` argument is added to the instruction while the same value is present in the actual struct argument. The same goes for Preset creation.
