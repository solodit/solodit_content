**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Unchecked possible zero address for `_to` parameter in EthBridge.
**Description**

In contract EthBridge.sol, at line 69 in function depositERC20To the_to function parameter can be zero and is not checked. The following function calls will not revert in case of a zero _to address, leading to undefined behavior, such as depositing tokens but not being able to receive them.
ftrace | funcSig
function depositERC20To(
69
address_l1Token
70
address _12Token,
71
address_tot,
72
uint256 _amount,
73
address_zroPaymentAddress
74
bytes memory _ adapterParams
75
bytes calldata_data
76
) external virtual payable {
77
require(_to != address(0), "EthBridge: zero_to address");
78
_initiateERC20Deposit(_l1Token
79
,
80
_12Token, msg.sender, _to

**Recommendation**

Add a sanity check for the_to address to not be zero and revert otherwise.

**Re-audit comment**

Resolved

## Low Risk

### Pragma version lock
**Description**

It's recommended to have the same compiler version that the contracts were tested with the most. This way it reduces the risk of introducing unknown bugs. There are also different versions used throughout the project, for example ^0.8.0 in LzApp.sol and ^0.8.9 in EthBridge.sol.

**Recommendation**

Lock pragma versions.

**Re-audit comment**

Acknowledged

### Config param sanity check for `_srcAddress` in LzApp setTrustedRemote.
**Description**

In contract LzApp.sol, in the function set Trusted Remote, there's no sanity check for the _srcAddress parameter. This can generate a wrong assignment even if the route can only be accessed by the owner of the contract.

**Recommendation**

Add a sanity check to prevent_srcAddress from being an empty slice of bytes.

**Re-audit comment**

Acknowledged

### Config param sanity check for `_srcAddress` in LzApp forceResumeReceive.
**Description**

In contract LzApp.sol, in the function force Resume Receive, there's no sanity check for the _srcAddress parameter. This can generate a wrong assignment, even if the route can only be accessed by the owner of the contract.

**Recommendation**

Add a sanity check to prevent_srcAddress from being an empty slice of bytes.

**Re-audit comment**

Acknowledged

## Informational

### Unused function parameters in EthBridge.
**Description**

In contract EthBridge.sol, at lines 126 and 127, variables_srcAddress and_nonce are not being used throughout the function.

**Recommendation**

Make sure the variables are not actually needed and remove them.

**Re-audit comment**

Acknowledged

### Variable declaration order in LzApp.
**Description**

In contract LzApp.sol, at line 17 the variable failed Messages is declared after the internal function_NonblockingLzApp_init. Maintain a consistent order of variable and function declarations throughout the project.

**Recommendation**

Move the variable declaration to the top of the file, before any function declaration. Do this in all contract files.

**Re-audit comment**

Resolved

### No upgradeability pattern used despite indicators.
**Description**

In contracts LzApp.sol and NonnlockingLzApp.sol you are leaving an empty reserved space, for possible future upgrades and also using the OwnableUpgradeable and Initializer contracts. However, there's no upgradability mechanism used, such as Openzeppelin's Upgradable Proxy.

**Recommendation**

If you're not intending to use this pattern, please refactor the contracts to reflect this, otherwise complete the implementation in this direction.

**Re-audit comment**

Acknowledged
