**Auditors**

[JecikPo](https://x.com/jecikpo)

[shaflow01](https://x.com/shaflow01)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/015_CODESPECT_TOKENTABLE_ECDSA_DISTRIBUTOR.pdf)

# Findings

## Medium Risk

### [M-01] The upgrade permission for the protocol was assigned to the wrong role.

**Files:** [BaseECDSADistributor.sol](https://github.com/EthSign/ecdsa-token-distributor/tree/6d5db7f144d7468644313c98f9f310dbaadd1b01/src/core/BaseECDSADistributor.sol#L194)

**Description:**

In the protocol, there are two roles: one is the deployer contract controlled by the TokenTable, which is responsible for initializing the ECDSADistributor contract and includes the fee parameters required for token distribution. The second role is the contract owner, which is controlled by the project team responsible for the token distribution. However, the upgrade privilege is assigned to the contract owner, which can lead to potential issues.

```solidity
// solhint-disable-next-line no-empty-blocks
function _authorizeUpgrade(address newImplementation) internal virtual override onlyOwner { }
```

**Impact:** The project team can upgrade the ECDSADistributor contract and set the deployer address to a malicious implementation they control. This allows them to bypass paying fees to the TokenTable or even steal the fees.

**Recommendation:** It is recommended to transfer the upgrade authority to the deployer contract controlled by the TokenTable.

**Status:** Fixed

**Client response:** Fixed at [9c2cc47a5ed2dd745a3396f9c51733f9f25a69b6](https://github.com/EthSign/ecdsa-token-distributor/pull/8/commits/9c2cc47a5ed2dd745a3396f9c51733f9f25a69b6)

## Informational

### [I-01] Hook call contains untrusted data

**Files:** [BaseECDSADistributor.sol](https://github.com/EthSign/ecdsa-token-distributor/tree/6d5db7f144d7468644313c98f9f310dbaadd1b01/src/core/BaseECDSADistributor.sol), [FungibleTokenWithFeesECDSADistributor.sol](https://github.com/EthSign/ecdsa-token-distributor/tree/6d5db7f144d7468644313c98f9f310dbaadd1b01/src/core/extensions/FungibleTokenWithFeesECDSADistributor.sol)

**Description:**

The `userClaimData` in the hook call is not part of the signed data and is instead allowed to be arbitrarily provided by the caller.

```solidity
function claim(
    ...
    bytes[] calldata extraDatas
)
   ...
{
    //...
}
```

```solidity
function __tryCallClaimHook(...)
    private
{
    address claimHook = _getBaseECDSADistributorStorage().claimHook;
    if (claimHook != address(0)) {
        isBeforeClaim
            ? IClaimHook(claimHook).beforeClaim(delegate, recipient, group, data, extraData)
            : IClaimHook(claimHook).afterClaim(delegate, recipient, group, data, claimedAmount, extraData);
    }
}
```

**Impact:** If the hook incorrectly assumes that this field is trustworthy data, it could impact the system. Moreover, since anyone can claim the receipt on behalf of the user, it’s possible that the input for this field is not what the receipt expected.

**Recommendation:** It is recommended to include this data within the signed data to ensure its trustworthiness.

**Status:** Acknowledged

**Client response:** Acknowledged

### [I-02] The version information is not included in the signed data.

**Files:** [BaseECDSADistributor.sol](https://github.com/EthSign/ecdsa-token-distributor/tree/6d5db7f144d7468644313c98f9f310dbaadd1b01/src/core/BaseECDSADistributor.sol#L169)

**Description:**

The ECDSADistributor contract is an upgradeable contract and contains version information. After the contract is upgraded, the version information may be changed.

```solidity
function version() public pure virtual override returns (string memory) {
    return "0.4.1";
}
```

However, when constructing the signed data for user claims, the version information is not included in the signature, which may introduce potential risks.

```solidity
function encodeHashToBeSigned(...)
    public
    view
    virtual
    returns (bytes32)
{
    return keccak256(abi.encode(block.chainid, address(this), recipient, userClaimId, userClaimData));
}
```

**Impact:** After the contract version information is changed, previously generated signatures can still be used. This may pose some potential risks.

**Recommendation:** It is recommended to include the version information in the signed data.

**Status:** Acknowledged

**Client response:** Acknowledged
