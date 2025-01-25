**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### Addresses Should Not Be Hardcoded

**Severity** - Low

**Status** - Acknowledged

**Description**

In the Fias2.sol addresses such as FOREVERR_MINTER and LITCRAFT_MINTER have been hardcoded , it is possible that on different chains the addresses corresponding to these roles are different and hence instead of hardcoding the addresses they should be assigned in the initializer.

**Recommendation**:

It is recommended that addresses should be assigned in the initializer instead of being hardcoded.

**Client comment**: 

no impact because we are only using this contract on ETH and not other chains with other Forevver/LitCraft/Admin addresses.

## Informational

### Unused Constants in Fias Token Contract

**Severity** - Informational

**Status** - Acknowledged

**Description**:

The Fias token contract defines two constants, GLOBAL_LIMIT and CONTRACT_ADMIN, which are not utilized within the contract's implementation. The GLOBAL_LIMIT is set to 350,000,000, presumably intended to represent the maximum total supply of tokens. The CONTRACT_ADMIN is defined as an address, likely meant for administrative functions. However, neither of these constants is referenced in any of the contract's functions or logic.


**Recommendation**:


If these constants are intended for future use:
Document their intended purpose in comments above each constant.
Implement the functionality that utilizes these constants, such as enforcing the global token limit or adding admin-only functions.
If these constants are no longer needed:
Remove the unused constants to improve code clarity and slightly reduce deployment gas costs.
For the GLOBAL_LIMIT:
If it's meant to represent the maximum token supply, consider implementing a check in the initialize function to ensure the total minted amount doesn't exceed this limit.
For the CONTRACT_ADMIN:
If administrative functions are planned, implement them with appropriate access control using this address.
Consider using OpenZeppelin's Ownable or AccessControl contracts for more robust admin functionality.

**Client comment**: no impact, the admin functions that previously used those constants were removed and they only remain for clarity/readability.


### Lack of Event Emissions for Critical State Changes in Fias Token Contract

**Severity** - Informational

**Status** - Acknowledged

**Description**:

The Fias token contract, including its V2 upgrade, fails to emit events for significant state changes, particularly during token minting in the initialize function and token burning in the burn function. 


**Recommendation**:


For the initialize function:
Define and emit a custom event for the initial token minting. For example:
event InitialMint(address indexed to, uint256 amount);

```solidity
function initialize() public initializer {
   __ERC20_init("Fias", "FIAS");
   _mint(LITCRAFT_MINTER, LITCRAFT_LIMIT*(10**18));
   emit InitialMint(LITCRAFT_MINTER, LITCRAFT_LIMIT*(10**18));
   _mint(FOREVVER_MINTER, FOREVVER_LIMIT*(10**18));
   emit InitialMint(FOREVVER_MINTER, FOREVVER_LIMIT*(10**18));
}
```
For the burn function:
Utilize OpenZeppelin's built-in _burn function, which already emits a Transfer event. If additional information is needed, consider adding a custom event:
event TokensBurned(address indexed burner, uint256 amount);

```solidity
function burn(uint256 amount) public {
   _burn(_msgSender(), amount);
   emit TokensBurned(_msgSender(), amount);
}
```

**Client comment**: 

this would be nice to add if we change anything and it’s odd that it isn’t part of the default OpenZeppelin functions, but there is no security risk to this.
