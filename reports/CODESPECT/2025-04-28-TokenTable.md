**Auditors**

[JecikPo](https://x.com/jecikpo)

[Bloqarl](https://x.com/TheBlockChainer)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/016_CODESPECT_TOKENTABLE_MERKLE_DISTRIBUTOR.pdf)

# Findings

## Medium Risk

### [M-01] Incorrect Withdrawal Implementation May Lead to Lock of Unclaimed NFTs

**Files:** [SimpleERC721MerkleDistributor.sol](https://github.com/EthSign/merkle-token-distributor/tree/96fedd0d945693149e0903c84502004bf819996c/src/core/extensions/SimpleERC721MerkleDistributor.sol#L12)

**Description:**

The `SimpleERC721MerkleDistributor` contract allows claiming of ERC721 tokens instead of ERC20. It inherits most of its functions from `TokenTableMerkleDistributor`. The main difference is that `_send()` and `withdraw()` functions handle ERC721 token minting and transfers instead of ERC20 transfers.

The claiming process in this case relies on minting the NFTs directly from the token contract. In the ERC20 versions the project owner is equipped with `withdraw()`, which allows to recover all non-claimed tokens.

In this `SimpleERC721MerkleDistributor` contract, however the the overloaded `withdraw()` attempts to transfer existing NFTs from the contract. This will not work because the unclaimed NFTs are not actually minted to that contract.

```solidity
function withdraw(bytes memory extraData) external virtual override onlyOwner {
    uint256[] memory tokenIds = abi.decode(extraData, (uint256[]));
    for (uint256 i = 0; i < tokenIds.length; i++) {
        IERC721(_getBaseMerkleDistributorStorage().token).safeTransferFrom(address(this), owner(), tokenIds[i]);
    }
}

function _send(address recipient, address token, uint256 amount) internal virtual override {
    for (uint256 i = 0; i < amount; i++) {
        IERC721SafeMintable(token).safeMint(recipient);
    }
}
```

**Impact:** In the case where the minting permissions are granted to the distributor contract, but not to the project owner, the owner cannot mint the unclaimed tokens for himself and they might end up locked (or rather never minted).

**Recommendation(s):** Change the `withdraw()` function in `SimpleERC721MerkleDistributor` so that it mints the NFTs instead of transferring them. Warning! If that change is implemented the `withdraw()` function must also be placed into the `SimpleNoMintERC721MerkleDistributor` contract as it inherits from `SimpleERC721MerkleDistributor`. If that is not done, then withdrawals will not work for `SimpleNoMintERC721MerkleDistributor`.

**Status:** Fixed

**Update from TokenTable:** Revised withdraw logic in `0e4cd1d1c27dfbb98080728da8955a10d1143a9c`.

## Low Risk

### [L-01] Upgrade Permission for the Protocol Assigned to the Project Owner

**Files:** [BaseMerkleDistributor.sol](https://github.com/EthSign/merkle-token-distributor/tree/96fedd0d945693149e0903c84502004bf819996c/src/core/BaseMerkleDistributor.sol#L293)

**Description:**

In the protocol, there are two roles:

- The `MDCreate2` contract controlled by TokenTable, which is responsible for initialising the contracts inheriting from `BaseMerkleDistributor` and includes the fee parameters required for token distribution;
- The contract owner, which is controlled by the project team responsible for the token distribution;

However, the upgrade privilege is assigned to the contract owner, which can lead to potential issues.

```solidity
// solhint-disable-next-line no-empty-blocks
function _authorizeUpgrade(address newImplementation) internal virtual override onlyOwner { }
```

It gives the project owner control to upgrade the distribution contracts.

**Impact:** The project team can upgrade the contract and set the deployer address to a malicious implementation they control. This allows them to bypass paying fees to TokenTable or even take the fees for themselves.

**Recommendation(s):** Removal of the upgradability option.

**Status:** Fixed

**Update from TokenTable:** Fixed in `c991b09f8da9eba24b0a789e6c7cb332d0394f40`.

## Informational

### [I-01] The NFT Fee Handling is Incompatible with BIPS Type of Fees

**Files:** [SimpleERC721MerkleDistributor.sol](https://github.com/EthSign/merkle-token-distributor/tree/96fedd0d945693149e0903c84502004bf819996c/src/core/extensions/SimpleERC721MerkleDistributor.sol#L16)

**Description:**

In case of ERC721 token distribution, the `claimedAmount` of the individual claim is representing the amount of tokens to be minted or transferred. As is the case with NFTs that variable can be small, i.e. 1 or 2. The `claimedAmount` is then passed into the `ITTUFeeCollector::getFee()` to calculate the exact fee amount charged to the claimer. While it works fine in case fixed fees are configured, it may not work as expected in case the protocol owner would like to charge fees based on bips, as the following calculation from the `ITTUFeeCollector::getFee()` may return zero in case of a low value of `claimedAmount`:

```solidity
tokensCollected = (tokenTransferred * feeBips) / BIPS_PRECISION;
```

The exact number when zero is returned depends on the value of `feeBips`.

**Impact:** Project needs to stick to fixed fees in case of ERC721 distributions, hence, fee policy flexibility is lost.

**Recommendation(s):** multiply the `claimedAmount` by `BIPS_PRECISION` before sending it to `ITTUFeeCollector::getFee()`.

**Status:** Acknowledged

**Update from TokenTable:** Acknowledged.

### [I-02] getClaimDelegate Function Not Blocked When Delegated Claiming is Disabled

**Files:** [NFTGatedMerkleDistributor.sol](https://github.com/EthSign/merkle-token-distributor/tree/96fedd0d945693149e0903c84502004bf819996c/src/core/extensions/custom/NFTGatedMerkleDistributor.sol#L32)

**Description:**

The `NFTGatedMerkleDistributor` contract has delegated claims disabled. The following functions:

- `setClaimDelegate();`
- `batchDelegateClaim();`
- `delegateClaim();`

are blocked using revert. However the `getClaimDelegate()` is not.

**Impact:** No impact to the protocol functionality, only inconsistent implementation.

**Status:** Acknowledged

**Update from TokenTable:** Acknowledged.
