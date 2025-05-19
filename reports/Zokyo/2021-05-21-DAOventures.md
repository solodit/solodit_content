**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Unlimit mint in DVGToken.sol

**Description**

The contract owner can mint unconditionally tokens to any address.

**Recommendation**:

Add restrictions, if possible. If not, add a comment that briefly explains how users are
protected from unexpected minting.

## Medium Risk

### Math could fail due to negative number in DVGToken.sol

**Description**

At function _moveDelegates, line 220 srcRepOld can be 0 or less than amount:
![image](https://github.com/user-attachments/assets/31b3c020-78e3-4794-a2bb-2d7dcd1b2831)


While line 221 expects srcRepOld to be >= amount:
![image](https://github.com/user-attachments/assets/015f1caf-eb93-420e-bbcb-bd27d8cb2dd1)

**Recommendation**:
Depending on requirements, add ‘revert’ or ‘if’ to correctly handle possible negative value.

### Potential gas overflow due to unlimited array length

**Description**

In function massUpdatePools() there is a loop that depends on pool.length. In the case
significant amount of pools, this function potentially could not be completed due to gas limits.

**Recommendation**:

Add a comment that confirms that no significant pool amount expected. Otherwise, add offset
and limit parameters, to update pools portionally.

## Low Risk

### SPDX license identifier not provided in DVGToken.sol

**Description**

Trust in smart contracts can be better established if their source code is available. Since
making source code available always touches on legal problems with regards to copyright, the
Solidity compiler encourages the use of machine-readable SPDX license identifiers.

**Recommendation**:

Before publishing, consider adding a comment containing "SPDX-License-Identifier:
<SPDX-License>" to each source file. Use "SPDX-License-Identifier: UNLICENSED" for
non-open-source code. Please see https://spdx.org for more information.

### Unnecessary ‘public’ keyword for constructor of DAOstake contract

**Description**

Visibility (public / external) is not needed for constructors anymore: To prevent a contract
from being created, it can be marked abstract. This makes the visibility concept for
constructors obsolete.

**Recommendation**:

Remove ‘public’ keyword from constructor.

## Informational

### Wrong comment in DVGToken.sol

**Description**

At line 73 there is a comment “Delegate votes from `msg.sender` to `delegatee`”, but actually
delegates function doesn’t delegate anything and doesn’t work with msg.sender. (delegates
does).

**Recommendation**:

Change comment to match actual function behaviour.

### Same code doubled twice DAOstake.sol

**Description**

![image](https://github.com/user-attachments/assets/1ce0954b-15ec-468f-8115-ca365d655850)

At function updatePool(uint256 _pid), lines 265 and 271 contains the same code:

**Recommendation**:

Move this code before or below if condition.
