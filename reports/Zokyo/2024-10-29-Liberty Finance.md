**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### _decimalOffset can be set to 6 

**Severity**: Low

**Status**: Resolved

**Description**

Contract AmanaVaultV1 inherits ERC4626 from OpenZeppelin library which uses virtual shares to mitigate inflation attack by setting decimalOffset as 0 but mentions the following as well:

While not fully preventing the attack, analysis shows that the default offset
(0) makes it non-profitable, as a result of the value being captured by the virtual shares (out of the attacker's
donation) matching the attacker's expected gains.

This means although the attacker will not make a profit, it still cannot be in the magnitude to prevent the attack completely.

Openzeppelin doc further mentions:
With a larger offset, the attack becomes orders of magnitude more
expensive than it is profitable. More details about the underlying math can be found xref:erc4626.adoc#inflation-attack[here]

Hence it is advised to set decimalOffset as 6 by overriding the method in the AmanaVaultV1 contract.

**Recommendation**: 

Set _decimalOffset as 6.

### Method ownerDepositForDistribution should check if enough tokens present in the rewards contract

Severity: Low

Status: Resolved

**Description**

In Contract Rewards.sol, the method ownerDepsoitForDistribution(..) sets a distribution amount but does not check if enough tokens are present in the contract to distribute such an amount. In the case there are not enough tokens, distributeRewards() will fail unless tokens are transferred.

Also, this method does not need nonReentrant modifier as it can be called only by owner and there are no transfer of tokens here.

**Recommendation**: 

Add a check to ensure enough tokens are present in the contract when the distribution amount is being set.

## Informational

### Upgradeable ERC20 token

**Severity**: Informational

**Status**: Acknowledged

**Description**

Contract LibertyFinanceToken is an ERC20 token that is upgradeable as well. It is generally not advised to make an ERC20 token upgradeable as it becomes unpredictable for token holders regarding their token shares and poses a centralization risk.

**Recommendation**: 

Consider not upgrading the ERC20 contract unless required by DAO/Governance and use multi-sig wallet for upgrades.

### Disable initializer

**Severity**: Informational

**Status**: Resolved

**Description**

Contract AmanaVaultV1 and Contract Rewards do not disable initializer as recommended by OpenZeppelin  by adding the code as  follows:
```solidity
constructor() {
       _disableInitializers();
   }
```
**Recommendation**: 

Disable intializers by adding the above code.
