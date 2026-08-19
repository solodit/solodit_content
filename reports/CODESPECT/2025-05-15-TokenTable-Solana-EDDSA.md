**Auditors**

[JecikPo](https://x.com/jecikpo), [shaflow01](https://x.com/shaflow01)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/019_CODESPECT_TOKENTABLE_SOLANA_EDDSA.pdf)

# Findings

## High Risk

### [H-01] Missing fee_collector check in the Claim instruction

**Files:** [claim.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/87b79fe77b74d734fc5274da93300aa39444146c/programs/eddsa-token-distributor-solana/src/instructions/claim.rs#L123), [claim.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/87b79fe77b74d734fc5274da93300aa39444146c/programs/eddsa-token-with-fees-distributor-solana/src/instructions/claim.rs#L115)

**Description:**

In the `Claim` instruction, `fee_collector` is not verified in the ctx, and no check is performed in the subsequent handler function either.

```rust
#[derive(Accounts)]
#[instruction(_project_id: String, _recipient: Pubkey, _claim_id: [u8; 32])]
pub struct Claim<'info> {
    //...
    #[account(mut)]
    pub fee_collector_vault: Option<UncheckedAccount<'info>>,
    /// CHECK: Checked in the function call.
    pub fee_collector: UncheckedAccount<'info>,
    //...
}
```

**Impact:** A malicious actor can pass in a malicious program to bypass fee payment.

**Recommendation(s):** It is recommended to verify that the `fee_collector` passed in the instruction matches the one recorded in the `airdrop` account

**Status:** Fixed

**Update from TokenTable:** Added fee collector account constraint in [f934c5c727a9bf5cf6bc3dbd362b957a9d4f90ea](https://github.com/EthSign/tokentable-unlocker-solana/commit/f934c5c727a9bf5cf6bc3dbd362b957a9d4f90ea).

## Informational

### [I-01] Changing the projectToken might not be functional.

**Files:** [set_base_params.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/87b79fe77b74d734fc5274da93300aa39444146c/programs/eddsa-token-distributor-solana/src/instructions/set_base_params.rs#L16), [deposit.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/87b79fe77b74d734fc5274da93300aa39444146c/programs/eddsa-token-distributor-solana/src/instructions/deposit.rs#L50)

**Description:**

In the `set_base_params` instruction, the airdrop owner is allowed to modify the token account used for the airdrop.

```rust
pub fn set_base_params(
    ctx: Context<SetBaseParams>,
    _project_id: String,
    token: Pubkey,
    start_time: u64,
    end_time: u64,
    authorized_signer: Pubkey
) -> Result<()> {
    //...
    ctx.accounts.airdrop.token = token;
```

However, switching may not function properly because the vault associated with the airdrop is singular and may have already been initialized as a `TokenAccount` for the previous mint. Moreover, the vault initialization lacks permission control, and the initializer is not required to hold the corresponding token.

```rust
pub struct Deposit<'info> {
    //...
    pub airdrop: Account<'info, Airdrop>,
    #[account(
        init_if_needed,
        payer = authority,
        seeds = [b"vault".as_ref(), airdrop.key().as_ref()],
        bump,
        token::mint = token_mint,
        token::authority = airdrop,
        token::token_program = token_program
    )]
    pub vault: InterfaceAccount<'info, TokenAccount>,
```

**Impact:** Modifying the token associated with the airdrop account may not be feasible. For instance:

1. The project initializes the airdrop account, and the token is set to `mint1`;
2. Another user calls `deposit`, which initializes the vault as a `TokenAccount` for `mint1`;
3. The project then calls `set_base_params` to change the token to `mint2`;
4. When the project tries to deposit `mint2` tokens, it fails because the vault was already initialized as a `TokenAccount` for `mint1` by another user;

**Recommendation(s):** It is recommended to add permission control to the `deposit` instruction, allowing only the airdrop owner to call it.

**Status:** Fixed

**Update from TokenTable:** Removed the ability to update the project token using `set_base_params()` in all affected programs in [a30590bd697e7564447b3e0e48e64199f06fea7e](https://github.com/EthSign/tokentable-unlocker-solana/commit/a30590bd697e7564447b3e0e48e64199f06fea7e).

### [I-02] The fee_collector_storage account constraint makes switching the fee_collector failed.

**Files:** [claim.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/87b79fe77b74d734fc5274da93300aa39444146c/programs/eddsa-token-distributor-solana/src/instructions/claim.rs#L110), [initialize.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/87b79fe77b74d734fc5274da93300aa39444146c/programs/eddsa-token-distributor-solana/src/instructions/initialize.rs#L70), [set_fee_collector.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/87b79fe77b74d734fc5274da93300aa39444146c/programs/eddsa-token-distributor-solana/src/instructions/set_fee_collector.rs#L52)

**Description:**

The EDDSA program allows different `fee_collector` programs to be configured for airdrop accounts. However, since the CPI to `fee_collector` program in the ctx uses an `Account<'info, fee_collector::models::FeeCollectorStorage>` type, it checks whether the `fee_collector_storage` account’s owner matches `fee_collector::models::FeeCollectorStorage::owner`. This causes the instructions to fail if a different `fee_collector` program is configured, due to an owner mismatch on the `fee_collector_storage` account.

```rust
pub fee_collector_storage: Option<
    Box<Account<'info, fee_collector::models::FeeCollectorStorage>>
>,
```

**Impact:** The airdrop account cannot be updated or used with a different `fee_collector` program, which is not the expected behavior.

**Recommendation(s):** In the ctx, specify the type of `fee_collector_storage` as `UncheckedAccount<'info>` instead of `Account<'info, T>`. During the CPI call, the `fee_collector` program will verify whether the account is as expected.

**Status:** Fixed

**Update from TokenTable:** Switched to `UncheckedAccount<'info>` for all usages of `fee_collector_storage` in all programs in [c0d34b3e6128278a11bf9df06812ffb87df3d779](https://github.com/EthSign/tokentable-unlocker-solana/commit/c0d34b3e6128278a11bf9df06812ffb87df3d779).

### [I-03] Unused code

**Files:** [utils.rs](https://github.com/EthSign/tokentable-unlocker-solana/tree/87b79fe77b74d734fc5274da93300aa39444146c/programs/eddsa-token-distributor-solana/src/instructions/utils.rs#L208)

**Description:**

The `utils.rs` file containing signature verification capabilities of the program contains unused code:

```rust
pub fn merkle_verify(proof: Vec<[u8; 32]>, root: [u8; 32], leaf: [u8; 32]) -> bool {
    let mut computed_hash = leaf;
    for proof_element in proof.into_iter() {
        if computed_hash <= proof_element {
            computed_hash = keccak::hashv(&[&computed_hash, &proof_element]).0;
        } else {
            computed_hash = keccak::hashv(&[&proof_element, &computed_hash]).0;
        }
    }
    computed_hash == root
}
```

Which was likely copied unnecessarily from the Merkle Distributor program.

Another unreachable code section is related to the `verify_secp256k1_ix` which is not used anywhere in the code, there could be two options. As per design the Secp256k1 signatures are not supported by the protocol

**Impact:** Higher SOL fees for program deployment and upgrades

**Recommendation(s):** Remove the unnecessary code.

**Status:** Fixed

**Update from TokenTable:** Unused code removed in [f925d1f5aaf75371cc9fd99c5c34269362abba97](https://github.com/EthSign/tokentable-unlocker-solana/commit/f925d1f5aaf75371cc9fd99c5c34269362abba97) and [54c916fe785c9249584836fed6bad594492c2a80](https://github.com/EthSign/tokentable-unlocker-solana/commit/54c916fe785c9249584836fed6bad594492c2a80).
