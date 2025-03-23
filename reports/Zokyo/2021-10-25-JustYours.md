**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Burn functions should be removed

**Description**

Since the white paper states that, there would never be any burning of tokens, we advise the
burn functions be removed as it’s not a required function of the core ERC20 token standard.

### Change accessibility specifier of mint function

**Description**

As stated in the white paper that there would only be a fixed amount of $JUST token, the
minting function should have a private accessibility modifier, so that after using it in the
constructor, no other contract or person can call it outside the contract.


### Total supply allocation upon deployment address should not be an EOA

**Description**

The address used for minting tokens of JUST token, we could not verify whether it's a smart
contract address, an EOA, or a multisig wallet.

**Recommendation**:

We recommend you use a smart contract address that is being controlled by a DAO or a
multisig contract.

## Informational

### Change public accessibility modifiers to external

All functions with public accessibility modifiers can be improved with external accessibility
modifiers.

### Make state variables that do not change constants

State variables such as name, decimals, and symbols, can be declared with the constant
accessibility modifiers since they do not change throughout the lifetime of the smart contract.
