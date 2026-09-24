**Auditors**

Kalogerone

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/025_CODESPECT_TOKENTABLE_MERKLE_WITH_FEES.pdf)

# Findings

## Medium Risk

### [M-01] Contract handles native tokens but the withdraw function doesn’t

**Files:** [TokenTableMerkleDistributorWithFees.sol](https://github.com/EthSign/tokentable-evm/blob/ebb6ef16f4e26faa153efabca9c789bb516af12d/src/merkle/core/extensions/TokenTableMerkleDistributorWithFees.sol)

**Description:**

The `TokenTableMerkleDistributorWithFees` contract is supposed to handle airdrops/distributions of native tokens, hence the `receive()` function. However, the `withdraw(...)` function can only retrieve ERC20 tokens:

```solidity
function withdraw(bytes memory) external virtual override onlyOwner {
    IERC20 token = IERC20(_getBaseMerkleDistributorStorage().token);
    token.safeTransfer(owner(), token.balanceOf(address(this)));
}
```

**Impact:** When the `TokenTableMerkleDistributorWithFees` contract is used for native tokens, the `withdraw(...)` function can’t be used and any excess or unclaimed tokens will be stuck in the contract.

**Recommendation(s):** Handle both cases appropriately, as the airdrop token can also be a native token.

**Status:** Fixed

**Client response:** Fixed in [f99596681d9d416beb2fa94ac08836ea21bcdd24](https://github.com/EthSign/tokentable-evm/pull/7/commits/f99596681d9d416beb2fa94ac08836ea21bcdd24)
