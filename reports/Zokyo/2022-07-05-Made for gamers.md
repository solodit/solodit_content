**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Critical Risk

### Anyone is able to upgrade implementation of the contract.
**Description**

EXPO.sol and EXPOVO.sol: function_authorizeUpgrade(). Function_authorizeUpgrade() is necessary in order to authorize that upgrading implementation is valid, thus this function should always be implemented and check that msg.sender of upgradeTo() function is valid(Either owner, admin or authorized user).

**Recommendation**

Validate msg.sender in_authorizeUpgrade(). For example, onlyOwner modifier from OZ OwnableUpgradeable can be used.

**Re-audit comment**

Resolved.

Post-audit:

OnlyOwner modifier is used and scenario was also covered with additional unit-tests.

### Lock period can be avoided with other ERC777 functions.
**Description**

Expo.sol. Contract overrides transfer functions, such as transfer(), send(), burn() in order to check lock period. However, there are few ERC777 standard functions which are not overridden and thus, can be used to avoid lock period on transfer. The following ERC777 functions perform transfer and burn and don't validate lock period: operatorSend(), operatorBurn(), transferFrom().

**Recommendation**

Override these functions as well to validate lock period in them.

**Re-audit comment**

Resolved.

Post-audit:

Internal functions_send() ans_burn() were overridden to check lock period.

## Medium Risk

### Iteration through all locks, including expired locks.
**Description**

Expo.sol: functions transfer(), send(), burn(), transferWithLock(). Functions iterate through all user's locks to verify that he doesn't transfer more than locked until some period of time. Iteration is performed through expired locks as well, performing unnecessary actions and increasing gas spendings. Also, in case there are a lot of locks for the user, iterating through all of them might consume more gas than allowed per transaction, preventing user from transferring his tokens. Issue is marked as medium, since only the owner can create locks for user.

**Recommendation**

Remove locks, which have expired after successful transfer.

**Re-audit comment**

Resolved

## Informational

### Owner is able to destroy the contract.
**Description**

Expo.sol: function destroySmartContract(). Owner has the ability to destroy the contract with the following function, destroying all users' balances as well. In case the owner key is compromised or stolen, the contract can be destroyed forever with all users' balances. Verify the necessity of this function. This issue is connected to crucial smart-contract logic, thus need to be mentioned in the report, and it needs to be verified by the team.

**Recommendation**

Verify the necessity of this function.

**Re-audit comment**

Resolved.

Post-audit:

The team has removed the selfdestruct functionality

### Use Solidity literal.
**Description**

EXPO.sol and EXPOVO.sol: function initialize(), line 31. Currently, "maxSupply" is assigned to a value 2700000000000000000000000000000. Such notation is difficult to be read and may lead to accuracy mistakes, when some zeros might be missed.

**Recommendation**

Since both contracts inherit OZ ERC777 standard token, which has 18 decimals, ether literal can be used to increase code readability. For example: 2_700_000_000_000 ether.

**Re-audit comment**

Resolved

### Burnable token.
**Description**

EXPO.sol and EXPOVO.sol inherit the OZ ERC777 standard token which has a default burn function. Though, the default burnable functionality allows every user to burn their own tokens. Thus in case of big distributions or any hack for a big amount, there is an ability for the malicious user to burn enough tokens to affect the economy of the protocol. Thus verify the usage of the default burn functionality. Burn functionality itself is not a security issue, but it crucial for the protocol, thus it needs to be verified and reflected in the report.

**Recommendation**

Verify the usage of the default burn functionality.

**Re-audit comment**

Resolved.

Post-audit:

The Customer team decided to leave the burn functionality, but set a minimum total supply that it cannot go below
