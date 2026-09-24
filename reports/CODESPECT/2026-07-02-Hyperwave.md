**Auditors**

cholakovvv

0xSynthrax

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/068_Hyperwave_Sub-Account_Strategy.pdf)

# Findings

## Low Risk

### [L-01] Repayment path reverts if a blocklist-capable baseAsset blocks the strategy or the vault

**Files:** [SubAccountFundManager.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountFundManager.sol)

**Description:**

`transferFundBackToVault(...)` repays only via `baseAsset.safeTransferFrom(msg.sender, address(boringVault), amount)`, with `baseAsset` and `boringVault` immutable and no alternative route.

**Impact:** If `baseAsset` is blocklist-capable (USDC is used in the aprimeUSD deployment scripts), a blocklist of the calling strategy or the `boringVault` recipient makes this transfer revert, so that strategy cannot repay and its `allocated` cannot be reduced. It needs the token issuer to blocklist a protocol-controlled address, is not attacker-triggerable, and freezes one strategy’s repayment.

**Recommendation:** Provide a governance-controlled alternative repayment route, or accept and document the blocklist risk of the chosen `baseAsset`.

**Status:** Fixed

**Client response:** Added comment in [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`f4ff022`](https://github.com/SwellNetwork/boring-vault/commit/f4ff022e972bb88e6045cb10fe9db71433792837).

### [L-02] transferFundFromVault(...) uses unchecked transfer(...) call

**Files:** [SubAccountFundManager.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountFundManager.sol)

**Description:**

`transferFundFromVault(...)` in `SubAccountFundManager.sol` moves the vault’s `baseAsset` to the calling strategy by encoding a raw `transfer(...)` call:

```solidity
boringVault.manage(
    address(baseAsset),
    abi.encodeWithSignature(
        "transfer(address,uint256)",
        msg.sender,
        amount
    ),
    0
);
```

The transfer call’s result is never checked. A token that returns `false` instead of reverting on failure would cause this call to succeed at the EVM level while no tokens actually move. Meanwhile, `allocated[msg.sender]` has already been incremented before this call is made, so the strategy’s allocation accounting would be updated even though the transfer silently failed.

**Impact:** Silent `transfer(...)` fails will cause accounting desync.

**Recommendation:** Check the `transfer(...)` call’s successful execution.

**Status:** Fixed

**Client response:** Fixed in [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`972021b`](https://github.com/SwellNetwork/boring-vault/commit/972021b25361f0d8f0cbe4e6fa50dfabc05275c5).

### [L-03] withdraw(...) slippage floor misprices staker shares when unstake is true

**Files:** [YieldBasisDecoderAndSanitizer.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/DecodersAndSanitizers/hyperwave/aprimeusd/YieldBasisDecoderAndSanitizer.sol)

**Description:**

The `withdraw(uint256 poolId, uint256 shares, uint256 minAssets, bool unstake, address receiver, bool withdrawStablecoins)` decoder derives its slippage floor from [`preview_withdraw(shares)`](https://github.com/yield-basis/yb-core/blob/master/contracts/LT.vy), which interprets `shares` as LT shares. `unstake` is read but ignored and not bound in the merkle leaf, so a strategist can call `withdraw` with `unstake = true`, where [`shares` is denominated in staker (gauge) shares](https://github.com/yield-basis/yb-core/blob/master/contracts/HybridVault.vy#L482) instead of LT shares.

**Impact:** The configured staker (`0xd829456FD63Ada7DE0657714A3A7A26DE403E3D8`) is a non-1:1 ERC4626 vault over the LT (`convertToAssets(1e18) ≈ 0.985` LT, drifting as rewards accrue). With `unstake = true` the decoder prices staker shares as LT shares, overstating the withdrawal, so `minAllowed` is too high and the strategist must set `minAssets` above what the smaller actual LT withdrawal can deliver, reverting the call. No funds are at risk since the LT still enforces the strategist’s `minAssets`; this blocks the staked-withdrawal path through the decoder and requires the position to have been staked first.

**Recommendation:** Bind `stake` / `unstake` in the merkle leaf so the decoder only authorizes the unstaked domain its floor assumes; if staked withdrawals are required, convert staker shares to LT shares via the staker’s `convertToAssets(...)` before calling `preview_withdraw(...)`.

**Status:** Fixed

**Client response:** fixed in [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`bcfacae`](https://github.com/SwellNetwork/boring-vault/commit/bcfacae4894eebabcaee2bee076d0336418ea07b).

## Informational

### [I-01] transferFundBackToVault(...) floors allocated to zero while transferring the full amount

**Files:** [SubAccountFundManager.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountFundManager.sol)

**Description:**

`transferFundBackToVault(amount)` sets `allocated[msg.sender]` to `0` when `amount > allocated[msg.sender]`, but still calls `baseAsset.safeTransferFrom(msg.sender, address(boringVault), amount)` for the full amount. `allocated[msg.sender]` is only increased in `transferFundFromVault` when tokens leave the vault, so it never exceeds the strategy’s actual debt and cannot be driven below it.

**Impact:** A strategy that returns more than it owes transfers the excess (`amount - allocated[msg.sender]`) into the vault while `allocated` is set to `0`, leaving the excess untracked.

**Recommendation:** Cap the decrease and the transfer to `allocated[msg.sender]`, or revert when `amount > allocated[msg.sender]`.

**Status:** Acknowledged

**Client response:** It is intended. After pulling fund from the vault and running strategy, sub account gets yield and repay amount can be greater than pulling amount

**CODESPECT fix review:** Acknowledged.

### [I-02] Slippage floors collapse to zero on deposit(...) and withdraw(...)

**Files:** [YieldBasisDecoderAndSanitizer.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/DecodersAndSanitizers/hyperwave/aprimeusd/YieldBasisDecoderAndSanitizer.sol)

**Description:**

`deposit(...)` and `withdraw(...)` compute `minAllowed = (expected * (slippageBase - maxSlippage)) / slippageBase` (with `expected = preview_deposit(assets, debt, false) / preview_withdraw(shares)`) and revert only when `provided < minAllowed`, with no zero guard. When `expected` is small enough that `minAllowed` truncates to `0`, a `provided` of `0` passes, removing the only on-chain slippage bound, since these arguments are not in the merkle leaf.

**Impact:** When the floor is `0`, a `deposit(...)` / `withdraw(...)` passes with `minShares` / `minAssets = 0`, so the call runs with no on-chain slippage protection and accepts any output amount, exposing it to value loss from price movement or sandwiching. In practice the floor reaches `0` only for dust-level inputs (the previews return `0` or revert for sufficiently small assets / shares), so a meaningful deposit or withdraw is not affected on the current pool. The gap matters for other LTs or pool states where the previews can return `0` over a wider range, which is relevant since multiple pools are planned.

**Recommendation:** Revert when `expectedShares` / `expectedAssets` is `0`, and when the provided `minShares` / `minAssets` is `0`.

**Status:** Fixed

**Client response:** fixed in [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`7bdc0d9`](https://github.com/SwellNetwork/boring-vault/commit/7bdc0d97c6085e6e82162cf8552cfc5f71ae5b3b).

### [I-03] transferFundFromVault(...) reverts for a registered strategy whose limit is unset or non-numeric

**Files:** [SubAccountFundManager.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountFundManager.sol), [Configuration.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/Configuration.sol)

**Description:**

`transferFundFromVault(...)` reads the limit as a string with `configuration.getConfiguration(limitKey)` and parses it with `_parseUint`. `getConfiguration(...)` returns an empty string for an unset key, and `_parseUint` reverts with `ParsingFailed()` on an empty or non-numeric string.

**Impact:** A strategy registered through `setStrategyLimitKey(...)` whose `Configuration` value is never set (or set non-numeric) reverts on every `transferFundFromVault` until governance sets a valid number. It only blocks pulls, never allows an over-pull, and is fixed by setting a valid value.

**Recommendation:** Store the limit as a `uint256` per strategy, or validate the resolved value is a non-empty numeric string when the key is set.

**Status:** Acknowledged

**Client response:** The configuration contract is designed for storing all config values of the vault. It’s general, so the value’s type is string. The responsibility of setting the correct value in the configuration contract is of pdao (offchain). We have to set it via Safe, and signers have to verify the data

**CODESPECT fix review:** Acknowledged.

### [I-04] Merkle leaf binds decoder and target addresses but not their code

**Files:** [SubAccountWithMerkleVerification.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountWithMerkleVerification.sol)

**Description:**

`_verifyManageProof(...)` builds the leaf from the decoder and target addresses, the value flag, the selector, and the decoded address args, but not from their `codehash`, so the proof never covers the code deployed at those addresses.

**Impact:** If the code behind a whitelisted decoder or target address changes, a previously generated proof still verifies, so governance must re-issue `manageRoot` whenever a whitelisted integration changes. The path is pdao-only, the in-scope decoders are immutable, and any harm requires a third-party target to upgrade to hostile behaviour.

**Recommendation:** Bind `target.codehash` (and `decoder.codehash`) into the leaf, or whitelist immutable protocol-controlled adapters instead of raw upgradeable endpoints.

**Status:** Acknowledged

**Client response:** This is a deliberate design. Typically, the `argumentAddresses` arguments encoded in the decoder will not be changed unless the target contract function is changed. In case it changed, `manageRoot` will be updated via pdao.

**CODESPECT fix review:** Acknowledged.

### [I-05] Authorized wrap/unwrap leaves cannot execute against the non-payable sub-account

**Files:** [SubAccountWithMerkleVerification.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountWithMerkleVerification.sol), [CreateAprimeUSDYieldBasisMerkleRoot.s.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/script/CreateAprimeUSDYieldBasisMerkleRoot.s.sol)

**Description:**

The merkle root authorizes wrap (`WETH.deposit(...)`) and unwrap (`WETH.withdraw(...)`) leaves, but `SubAccountWithMerkleVerification` has no `receive(...)` / `fallback(...)` and a non-payable constructor, so it cannot hold or receive native ETH.

**Impact:** The unwrap reverts because `WETH.withdraw(...)` sends native ETH to the sub-account, and the wrap cannot be funded because the sub-account holds no native ETH. Both leaves are non-functional. No value is lost or frozen: WETH stays usable as an ERC20 through the `approve(...)` / `deposit(...)` / `withdraw(...)` leaves.

**Recommendation:** Remove the wrap/unwrap leaves from the root, or add `receive() external payable` if native-ETH flows are intended.

**Status:** Fixed

**Client response:** Fixed in commit [PR-86](https://github.com/SwellNetwork/boring-vault/pull/86/changes/86764e136ebb4708413f049a7d335154fe63b8d1).

**CODESPECT fix review:** Fixed in commit [`86764e1`](https://github.com/SwellNetwork/boring-vault/commit/86764e136ebb4708413f049a7d335154fe63b8d1).

### [I-06] Merkle authorization assumes the decoder and target share the same function for a given selector

**Files:** [SubAccountWithMerkleVerification.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountWithMerkleVerification.sol), [CreateAprimeUSDYieldBasisMerkleRoot.s.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/script/CreateAprimeUSDYieldBasisMerkleRoot.s.sol)

**Description:**

`_verifyCallData(...)` builds the leaf from `bytes4(targetData)` and the decoder-returned addresses, then `manage(...)` sends the same `targetData` to `target`, so both dispatch on the same selector. The binding is only complete when the decoder function and the target function for that selector share the same signature and the decoder returns all of that function’s address parameters. Nothing on-chain enforces this.

**Impact:** In the deployed roots the binding holds: the decoder functions are exact-signature mirrors of the bound targets, with spender / receiver / newOwner bound. A future leaf for a `(target, selector)` whose target function differs in signature from the decoder function (a cross-signature selector collision, `1/2^32`) could bind the wrong address or none. It is not reachable today and depends on a governance-authored leaf.

**Recommendation:** In the merkle tooling, assert each leaf’s signature resolves to the same selector on both decoder and target, and that the decoder returns every address parameter of the target function.

**Status:** Acknowledged

**Client response:** We implement the decoder and sanitizer and always have to ensure that the decoder and sanitizer is compatible with target contract.

**CODESPECT fix review:** Acknowledged.

### [I-07] maxSlippage and slippageBase are global, not configurable per poolId

**Files:** [YieldBasisDecoderAndSanitizer.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/DecodersAndSanitizers/hyperwave/aprimeusd/YieldBasisDecoderAndSanitizer.sol)

**Description:**

`isAllowedPoolId` and `poolIdToLT` are configured per `poolId`, but `maxSlippage` and `slippageBase` are single values shared by the whole decoder. The team confirmed support for multiple pools is planned, so one slippage band would govern every pool the decoder serves.

**Impact:** With multiple pools enabled, the same slippage band applies to every pool. This band is a secondary check; the strategist-provided `min_shares` and `min_assets`, enforced by the LT, are the primary protection.

**Recommendation:** Configure `maxSlippage` and `slippageBase` per `poolId`.

**Status:** Fixed

**Client response:** Fixed in [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`9e24f4d`](https://github.com/SwellNetwork/boring-vault/commit/9e24f4d09d8137b50e9c172f436624f4bf13055d).

### [I-08] withdraw(...) reuses the deposit pool allowlist, so disabling a pool freezes existing positions

**Files:** [YieldBasisDecoderAndSanitizer.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/DecodersAndSanitizers/hyperwave/aprimeusd/YieldBasisDecoderAndSanitizer.sol)

**Description:**

`deposit(...)` and `withdraw(...)` both check the same `isAllowedPoolId[poolId]` flag. `setAllowedPoolId(poolId, false)`, which governance uses to stop new deposits, also makes the `withdraw(...)` decoder revert for positions already in that pool.

**Impact:** Disabling a pool blocks the standard exit for positions already in it, and the revert aborts the whole `manageWithMerkleVerification(...)` batch. No funds are lost: re-enabling the pool restores the exit, and the pdao-only `manage` path can unwind the position without the decoder. It is reachable only through a `GOVERNANCE_ROLE` write.

**Recommendation:** Track entry and exit allowlists separately, or document that disabling a pool also blocks withdrawals from it.

**Status:** Fixed

**Client response:** Fixed in this PR [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`82337c5`](https://github.com/SwellNetwork/boring-vault/commit/82337c59c614ba5f21703d54d1604da787f3ad3f).

### [I-09] Interleaved verify/execute makes the YieldBasis slippage floor depend on a live preview

**Files:** [SubAccountWithMerkleVerification.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountWithMerkleVerification.sol), [YieldBasisDecoderAndSanitizer.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/DecodersAndSanitizers/hyperwave/aprimeusd/YieldBasisDecoderAndSanitizer.sol)

**Description:**

`manageWithMerkleVerification(...)` verifies and executes each call in the same loop iteration: `_verifyCallData(i)` is followed immediately by `manage(i)`, so call `i` is verified only after calls `0..i-1` have already changed state. For `deposit` and `withdraw`, the decoder derives the slippage floor from the live `ILT.preview_deposit(...)` / `preview_withdraw(...)` value (`deposit_crvusd` and `redeem_crvusd` include no floor).

**Impact:** An earlier call in the same batch can move the state a later call’s preview reads, so the decoder’s slippage floor for that later call is computed against already-mutated state. The strategist-provided `min_shares` / `min_assets` still cap the loss, but if `lt` is spot-priced (set via `setPoolLT(...)`), the decoder’s floor can be manipulated.

**Recommendation:** Verify all calls in the batch before executing any of them, or document that the decoder floor is a secondary check dependent on live preview state.

**Status:** Fixed

**Client response:** fixed in [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`91e80ac`](https://github.com/SwellNetwork/boring-vault/commit/91e80ac18ccd65416775f0148d8ebde1830f2bf1).

### [I-10] Direct manage(...) call will execute successfully even when the contract is paused

**Files:** [SubAccountWithMerkleVerification.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountWithMerkleVerification.sol)

**Description:**

`manage(...)` in `SubAccountWithMerkleVerification.sol` is declared `public requiresAuth`, meaning it is independently callable as an external entry point and not restricted to being invoked only internally by `manageWithMerkleVerification(...)`. The `isPaused` check is only enforced in the `manageWithMerkleVerification(...)`, and since `manage(...)` itself contains no `isPaused` check, it can be called while contract is paused.

**Impact:** The `pause()` mechanism can be silently bypassed.

**Recommendation:** Add the `isPaused` check directly inside `manage(...)` function.

**Status:** Fixed

**Client response:** fixed in [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`fee2ad4`](https://github.com/SwellNetwork/boring-vault/commit/fee2ad4572a3e8066f959d7314f073ca5f5f5122).

### [I-11] Missing zero-address validation in SubAccountFundManager constructor

**Files:** [SubAccountFundManager.sol](https://github.com/SwellNetwork/boring-vault/blob/6f6cd157b9aa4f63263e70339bbc37106cede493/src/base/Roles/SubAccountFundManager.sol)

**Description:**

The constructor in `SubAccountFundManager` assigns all three immutables directly from constructor arguments with no `address(0)` validation. In contrast, `YieldBasisDecoderAndSanitizer` does check the passed argument for `address(0)` before usage.

**Impact:** Since these variables are immutable, a deployment-time mistake is unrecoverable without redeploying the contract.

**Recommendation:** Add zero-address checks for constructor arguments.

**Status:** Fixed

**Client response:** fixed in [PR-87](https://github.com/SwellNetwork/boring-vault/pull/87).

**CODESPECT fix review:** Fixed in commit [`7830842`](https://github.com/SwellNetwork/boring-vault/commit/7830842d50e9e6effdf19341be277d825909b2c6).
