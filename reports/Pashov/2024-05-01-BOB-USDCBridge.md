**Auditor**

[Pashov Audit Group](https://twitter.com/PashovAuditGrp)

# Findings

## Low Risk

### [L-01] Bridged USDC Standard not fully complied

[docs link](https://github.com/circlefin/stablecoin-evm/blob/master/doc/bridged_USDC_standard.md#2-ability-to-burn-locked-usdc)

The Bridged USDC standard specifies that the bridge should possess the capability to burn locked tokens.

> Burn the amount of USDC held by the bridge that corresponds precisely to the circulating total supply of bridged USDC established by the supply lock.

However, the current implementation burns all tokens held by the L1 bridge, even those that were not bridged (e.g. sent by mistake). This discrepancy may lead to differences in the supplies on L1 and L2.

```solidity
    function burnLockedUSDC() external {
        require(msg.sender == burner, "Not whitelisted");

        IPartialUsdc token = IPartialUsdc(l1Usdc);
>>      uint256 balance = token.balanceOf(address(this));
        token.burn(balance);
    }
```

Consider burning only locked tokens:

```diff
+       uint256 balance = deposits[l1Usdc][l2Usdc];
        token.burn(balance);
```
