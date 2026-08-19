**Auditors**

[Talfao](https://x.com/talfao1), [Kalogerone](https://x.com/kalogerone)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/008_CODESPECT_TEMPEST_DEPOSITIDLE_FEATURE.pdf)

# Findings

## Medium Risk

### [M-01] Tempest vault not fully compatible with Royco IAM Vault

**Files:** [`AmbientInterface.sol`](https://github.com/Tempest-Finance/tempest_smart_contract/blob/5d5dd30c5b33e533f95c42aa421a136f0e6e320f/src/ambient-lp/AmbientInterface.sol)

**Description:**

The Tempest team has introduced a new feature, `depositIdle`, designed to incentivize users to deposit into Tempest via Royco on the Plume network. However, deposits must be processed through Royco’s Vault IAM, which requires partial ERC-4626 compatibility. The Tempest Vault, however, does not fully comply with this standard.

To successfully integrate the Tempest Vault with Royco’s IAM Vault, the following minimum requirements must be met [[ref](https://docs.royco.org/for-incentive-providers/create-an-iam)]:

- `asset()` function must not revert → *Not implemented in Tempest.*;
- `deposit()` function must not revert → *Tempest intends to use `depositIdle()` for Royco instead.*;
- `convertToAssets()` function must not revert → *Implemented.*;
- `previewDeposit()` function must return an accurate value → *Implemented.*;
- ERC-20 functionality must remain intact → *Implemented.*;

**Additional compatibility issues:**

There are also differences in the parameter structure of deposit and withdrawal functions between Tempest and Royco Vaults:

- Tempest’s `withdraw(...)` function takes **five** parameters, whereas Royco’s `withdraw(...)` uses **three**:

```solidity
VAULT.withdraw(uint256, address, address)
```

- Tempest’s `depositIdle(uint8, uint256, address)` (even if renamed to `deposit(...)`) differs from Royco’s deposit function:

```solidity
VAULT.deposit(uint256, address)
```

- Royco’s `mint(...)` function will not work as Tempest does not implement any `mint(...)` function;

**Withdrawal reversion risk:**

Additionally, Royco’s `withdraw(...)` function is likely to revert due to its built-in slippage check. Royco verifies that:

```solidity
expectedShares == shares
```

Where:

- **`expectedShares`** is derived from `previewWithdraw(...)`;
- **`shares`** is returned from `withdraw(...)`;

However, Tempest collects fees from Ambient during the execution of `withdraw(...)`, which may cause the returned `shares` to be lower than `expectedShares`, triggering a revert.

A potential workaround is to exclusively use Royco’s `redeem(...)` function, which does not include the slippage check.

**Impact:** The Tempest Vault is currently incompatible with Royco’s Vault IAM, which could prevent successful integration and disrupt the planned incentive mechanism.

**Recommendations:**

- Rename `depositIdle(...)` to `deposit(...)`, ensuring an equivalent function remains for the existing `deposit(...)` implementation;
- Implement the `asset(...)` getter function to meet compatibility requirements;
- Modify deposit and withdrawal function definitions to align with Royco Vault’s structure;
- Consider using only the `redeem(...)` function, as it bypasses Royco’s slippage check;

**Status:** Fixed

**Update from Tempest:**
