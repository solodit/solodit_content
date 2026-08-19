**Auditors**

[JecikPo](https://x.com/jecikpo), [shaflow01](https://x.com/shaflow01)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/011_CODESPECT_TOKENTABLE_SOLANA_MERKLE_AIRDROP.pdf)

# Findings

## Medium Risk

### [M-01] The set_default_fee_collector instruction cannot be executed

**Files:** [set_fee_collector.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/2025f68a4d699cc4997c133775f26f2768aba7e6/programs/merkle-token-distributor-solana/src/instructions/set_default_fee_collector.rs)

**Description:**

The `set_default_fee_collector` instruction is used to modify the `default_fee_collector`. Since it requires `config.admin` for permission validation, the config account should have already been initialized when calling the instruction. However, due to the incorrect assignment of the `init` attribute to the config account in the ctx, the `set_default_fee_collector` instruction fails to execute successfully.

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

**Recommendation(s):** It is recommended to remove the `init` attribute from the config account in the ctx.

**Status:** Fixed

**Update from TokenTable:** Removed `init` attribute from the config account in the Anchor context in [e2cf5fbc8802845c56d0e0ab48c874c0000ce015](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/e2cf5fbc8802845c56d0e0ab48c874c0000ce015) and added `mut` attribute in [8edb2ab7e2a63c37258b78f365bce2d43db3403f](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/8edb2ab7e2a63c37258b78f365bce2d43db3403f).

## Low Risk

### [L-01] Fee configuration conflict

**Files:** [merkle-token-distributor-solana::initialize.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/merkle-token-distributor-solana/src/instructions/initialize.rs#L55), [unlocker-v2-solana::initialize.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/unlocker-v2-solana/src/instructions/initialize.rs#L67)

**Description:**

If the airdrop account in the `merkle-token-distributor-solana` program and the unlocker account in the `unlocker-v2-solana` program share the same `project_id`, they will use the same fee configuration. Since both accounts can be created permissionlessly, there is no guarantee that they are created and controlled by the same owner. This may lead to a fee configuration conflict.

**Impact:** If unlock and airdrop accounts with the same `project_id` are controlled by different owners and each wants to use a different `fee_token`, there may be some complications in fee configuration.

**Recommendation(s):** It is recommended to distinguish between the fee configuration accounts for the two accounts.

**Status:** Fixed

**Update from TokenTable:** Add distributor pubkey field as additional seeds parameter when deriving a fee account in [40ebaffac8ecbf186e0625568fb10de967340d6c](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/40ebaffac8ecbf186e0625568fb10de967340d6c).

## Informational

### [I-01] Allow the fee_collector to be set arbitrarily during the initialization of the airdrop account

**Files:** [initialize.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/merkle-token-distributor-solana/src/instructions/initialize.rs)

**Description:**

In the `initialize` instruction, if `init_fee_account` is set to `false`, then the check `ctx.accounts.fee_collector.as_ref().unwrap().key() == fee_collector.key()` is skipped. This means that it allows the `airdrop.owner` to initialize any `fee_collector`.

```rust
pub fn initialize(...) -> Result<()> {
    ctx.accounts.airdrop.owner = owner;
    ctx.accounts.airdrop.fee_collector = fee_collector;
    ctx.accounts.airdrop.project_token = project_token;

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
        //...
    }
}
```

**Impact:** In the current system, allowing the `fee_collector` account to be set arbitrarily during initialization does not cause any loss, because the no-fee claim, as designed by the protocol, fails due to a constraint in the ctx. However, the `airdrop.owner` may have the motivation to initialize the `fee_collector` as `pubkey::default` during initialization. This would enable claims related to that account to be processed without any fees.

**Recommendation(s):** It is recommended not to allow the `airdrop.owner` to arbitrarily initialize the `fee_collector`.

**Status:** Fixed

**Update from TokenTable:** In [aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461), `fee_collector` program account verification is handled manually. Anchor now expects an `UncheckedAccount<>`, and in all instructions where `fee_collector` can be set, we verify that the provided account matches the instruction parameter value and that the provided account is executable.

### [I-02] Changing the fee_collector to a different program will cause instructions to fail

**Files:** [set_fee_collector.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/merkle-token-distributor-solana/src/instructions/set_fee_collector.rs)

**Description:**

The `set_fee_collector` instruction allows setting a different `fee_collector` program for Airdrop’s fee processing capabilities, as per TokenTable’s feedback from the previous audit:

The problem arises when the `FeeCollector` `program_id` is changed and certain instructions which take the `fee_collector` program account are called. The Anchor implementation under the hood will validate the program account against the `program_id` of the `FeeCollector` which was placed there at compile time:

```rust
pub fee_collector: Option<Program<'info, FeeCollector>>,
```

**Impact:** It will not be possible to update the `fee_collector` program account without also updating the entire merkle token distributor program.

**Recommendation(s):** Remove the `fee_collector` account from Anchor’s context structs and handle it manually within the instruction code.

**Status:** Fixed

**Update from TokenTable:** In [aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/aa48e8ab3c30b65f0e90a3be35cdb81a7f7f9461), `fee_collector` program account verification is handled manually. Anchor now expects an `UncheckedAccount<>`, and in all instructions where `fee_collector` is used, we verify that the provided account matches the expected unlocker’s/airdrop’s `fee_collector` but skip the account executable check, since this would have already been checked when the account was set.

### [I-03] Lack of Option wrapper on fee account

**Files:** [claim.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/merkle-token-distributor-solana/src/instructions/claim.rs#L133)

**Description:**

The claim instructions are invoked with a few accounts related to fee collection:

- `authority_fee_ata`;
- `fee_collector_storage`;
- `fee_collector_vault`;
- `fee_collector`;
- `fee_token_mint`;
- `fee`;
- `fee_token_program`;

The fee collection mechanism is optional, hence the design allows skipping them if they are unnecessary through the `Option` wrapper on the account type in the instructions contexts.

The fee account however is not:

```rust
/// CHECK: The account is checked in the FeeCollector, not here.
#[account(mut)]
pub fee: UncheckedAccount<'info>,
```

**Impact:** Expected difficulties in building the fee-less transactions as the fee account still needs to be provided to the instruction call.

**Recommendation(s):** Wrap the fee account type in `Option`.

**Status:** Fixed

**Update from TokenTable:** As of [78051afb53579a4e6558519000d6c35f510a5533](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/78051afb53579a4e6558519000d6c35f510a5533), the fee collection mechanism is no longer optional. `fee` is a required account and the documented structure here is needed to support the updated fee collection mechanism.

### [I-04] Miscalculated MerkleAirdrop size

**Files:** [merkle_airdrop](https://github.com/EthSign/tokentable-unlocker-solana/tree/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/merkle-token-distributor-solana/src/state/merkle_airdrop.rs#L20)

**Description:**

When creating the `MerkleAirdrop` account, use `calculate_size` to determine the allocated space.

```rust
pub fn calculate_size(uri_length: usize) -> usize {
    let mut size: usize = 0;
    size += 248; // Takes care of all non-vector items
    size += 4 + uri_length; // Add required data size of data vector

    size
}
```

The calculation seems to be implemented incorrectly as the total size of non-vector items should be `32 + 32 * 1 + 32 + 32 + 8 + 8 + 32 + 32 + 1 = 209` instead of `248`.

**Impact:** This would result in unnecessary rent wastage.

**Recommendation(s):** Calculate account space using the correct size.

**Status:** Fixed

**Update from TokenTable:** Updated account size calculation from a base of `248` to `209` in [08d6356a40601e5e5b0cf8cb6dfac9102da23583](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/08d6356a40601e5e5b0cf8cb6dfac9102da23583).

### [I-05] Redundant code

**Files:** [utils.rs](https://github.com/EthSign/tokentable-unlocker-solana/blob/67a39faff7b848ae05c5e3ab45e36b60efcc622e/programs/merkle-token-distributor-solana/src/instructions/utils.rs#L156)

**Description:**

The protocol contains multiple pieces of redundant code.

1. Both `if` and `require` statements are used when checking the result of the `merkle_verify` call; however, a single `require` statement would suffice;

```rust
pub fn _verify_and_claim<'info>(...) -> Result<u64> {
    //...
    if !merkle_verify(proof, root, leaf) {
        require!(false, TokenTableError::InvalidProof);
    }
}
```

**Impact:** Redundant code hinders readability and increases deployment costs.

**Recommendation(s):** It is recommended to optimize the redundant code.

**Status:** Fixed

**Update from TokenTable:** `merkle_verify()` require statement simplified in [2025f68a4d699cc4997c133775f26f2768aba7e6](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/2025f68a4d699cc4997c133775f26f2768aba7e6).

### [I-06] Redundant check

**Files:** [claim.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/2025f68a4d699cc4997c133775f26f2768aba7e6/programs/merkle-token-distributor-solana/src/instructions/claim.rs#L126)

**Description:**

The claim instruction contain redundant checks for `fee_collector`. The `fee_collector` is checked in the ctx and then checked again in the execution logic.

```rust
/// CHECK: Checked in the function call.
#[account(constraint = fee_collector.key() == airdrop.fee_collector.key())]
pub fee_collector: UncheckedAccount<'info>,

// ...
pub fn claim(...) -> Result<()> {
    // Fee collector
    require!(
        ctx.accounts.unlocker.fee_collector == ctx.accounts.fee_collector.key(),
        TokenTableError::InvalidFeeCollector
    );
}
```

**Impact:** Redundant checks increase the execution overhead of the transaction call.

**Recommendation(s):** It is recommended to remove the redundant checks.

**Status:** Fixed

**Update from TokenTable:** Redundant checks removed in [1aed8dad5fca73dd7e7b3d2a666c939a39a37be6](https://github.com/EthSign/tokentable-unlocker-solana/pull/8/commits/1aed8dad5fca73dd7e7b3d2a666c939a39a37be6).
