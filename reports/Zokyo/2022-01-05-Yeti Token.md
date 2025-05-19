**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Function add() is not restricted.

**Description**


Function add() has public access and no restrictions. Since the function pushes everything to
the array - so anyone can “spam” to the storage and make the contract unusable.

**Recommendation**:

Finish logic or remove commented code.

...

Storage.sol

Post-audit: File Storage.sol was deleted.

### Minting tokens is not restricted.

**Description**

Line 30, function mint(). Currently, anyone can call this function, thus minting as many tokens
as one wants.

**Recommendation**:

Consider adding restrictions to the mint() function(such as onlyOwner).


### Contract duplicates logic, which is already implemented in ERC20.

**Description**


Contract YetiToken inherits ERC20 logic, thus implementing all the logic of ERC20(including
transfers, allowances and storing of balances).
Contract should not duplicate this logic( such as mappings ‘balances’, ‘allowed’, and functions
‘transfer’, ‘approve’. This might also cause unexpected behavior, since contract overrides
function ‘transfer’, which writes changes to mapping ‘balances’, but doesn’t override function
‘transferFrom’, which will write changes to mapping ‘_balances’ from ERC20.
Function ‘approve’, which is supposed to operate in combination with ‘transferFrom’ won’t give
a proper result, since ‘approve’ writes allowances to a new mapping ‘allowed’, but
‘transferFrom’ uses a mapping ‘_allowances’ from ERC20.

**Recommendation**:

Remove duplicate logic such as mappings ‘balances’ and ‘allowed’ and functions ‘approve’,
‘transfer’, ‘balanceOf’, ‘allowance’, ‘totalSupply’, ‘balanceOf’.

### Variable ‘TOTAL_SUPPLY’ has no decimals

**Description**


Standard ERC20 token has 18 decimals, however decimals of ‘TOTAL_SUPPLY’ are missing.

**Recommendation**:

Add decimals for variable(e.g. TOTAL_SUPPLY = 100000000e18).

## Medium Risk

### Use the function ‘_mint’ in the constructor

**Description**

Line 18. All changes to balances should be performed through ERC20 functions.

**Recommendation**:

Use function ‘_mint’ in order to mint sender tokens properly. Function ‘_mint’ will write a given
value to mapping ‘_balances’ from ERC20, as well as update variable ‘_totalSupply’ and emit an
event that tokens were minted.

### Update Solidity version.

**Description**

Contracts utilize “^0.8.0” version and the hardhat config uses 0.8.0 version which is already
outdated, since the OpeZeppelin team has fixed several issues up to the version 0.8.10

**Recommendation**:

Use the latest stable Solidity version 0.8.10.

## Low Risk

### Contract contains commented code.

**Description**

lines 14, 17-19, 22-24, 26-28. Contract should not contain commented or unfinished logic.

**Recommendation**:

Finish logic or remove commented code.

## Informational

### Use the Address library.

**Description**

Lines 51-52, function withdraw(). Sending ether should be performed with the function
Address.sendValue, which already performs all necessary validations.

**Recommendation**:

Use Address.sendValue instead.

### Nft and royalties are minted to the owner.

**Description**


Lines 75, 84. Nft is always minted to the owner. Royalties are always set to the owner, since
only he can msg.sender.

**Recommendation**:

If royalties and nft should be minted to another address, consider adding a parameter with
this address in function mint()

### Use constant instead of numbers directly.

**Description**


Lines 85, 111. Numbers 100 and 10000 should be replaced with constants in order to improve
code readability.

**Recommendation**:

Replace numbers with constants.

### Variable can be marked as constant.

**Description**


Line 12. Variable ‘TOTAL_SUPPLY’ is never changed

**Recommendation**:

Make variable a constant.
