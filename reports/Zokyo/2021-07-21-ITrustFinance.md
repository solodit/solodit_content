**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Solidity version update

**Description**

The solidity version should be updated. Issue is classified as Medium, because it is included to
the list of standard smart contracts’ vulnerabilities. Currently used version (0.7.5) is considered
obsolete, which contradicts the standard checklist.

**Recommendation**:

You need to update the solidity version to the latest one in the branch - consider 0.7.6 or 0.8.6.

## Low Risk

### Public functions should be declared external to save gas.

**Description**

iTrustInsecureV2.sol, onERC721Received(), line 269.

**Recommendation**:

Function should be declared as external.

### Compare to a boolean constant

**Description**

Boolean constants can be used directly and do not need to be compare to true or false.
-require(bool,string)(_paused == false,Contract Frozen) (iTrustInsureV2.sol#122)
-require(bool,string)(purchaseExchange.treasuryAddress != address(0) &&
purchaseExchange.active == true,iTrust: Inactive exchange) (iTrustInsureV2.sol#211-214)
-require(bool)(cover.claimsAllowed == true && _userOwnsCover(userGUID,coverId) == TRUE
&& msg.sender == _userPolicies[userGUID].walletAddress) (iTrustInsureV2.sol#399-401)
-require(bool,string)(cover.iTrustOwned == true,submit NFT) (iTrustInsureV2.sol#398)
-cover.claimed == false (iTrustInsureV2.sol#484)
-cover.claimed == true (iTrustInsureV2.sol#691)

**Recommendation**:
Compare to a boolean constant.

## Informational

### Usage of standard libraries is preferrable

**Description**

To transfer admin rights, it is recommended to use the OZ AccessControl library.
To protect against reentry it is recommended to use the OZ ReentrancyGuard library.
For assigning roles and adding admins, it is recommended to use the OZ AccessControl
library.
For pausable functionality it is recommended to use the OZ Pausable library.
This approach will help in elimination of possible bugs, simplify the codebase and increase the
overall code quality.

**Recommendation**:

Consider usage of standard openzeppelin libraries.

### Gas optimization by omitting extra variable

**Description**

It is possible to call a method immediately without writing to a variable.
![image](https://github.com/user-attachments/assets/1ae826d4-9a58-497f-8033-edb855a2d9af)

**Recommendation**:

Call the method immediately without writing to a variable.

### Usage of “magic” numbers

**Description**

buyCover(), line 205, line 237
Avoid usage of “magic” numbers for the accuracy of calculations. Consider usage of the
constant.

**Recommendation**:

Consider usage of the constant.
