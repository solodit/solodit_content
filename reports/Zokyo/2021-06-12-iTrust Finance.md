**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Missing result assignment

**Description**

Burn.sol, line 130. “_burnData[vaultAddress].totalBurned.add(toBurn);”
Total burned value is not updated - addition result is not placed to storage.

**Recommendation**:
fix the condition.

### No check for allowance in transferFrom

**Description**

Vault.sol, line 351
Missing checking sender's allowance in transferFrom method. Thus, anyone can spend other
users' tokens

**Recommendation**:

check spender's allowance like in ERC20

### Incorrect condition

**Description**

StakingData.sol, line 416: Condition “i < 0” is always false for uint256. Review the functionality

**Recommendation**: 

fix the condition.

### Incorrect contracts used

**Description**

Some contracts inherit upgradeable contracts from OpenZeppelin contracts-upgradeable. So
it should use all utility contracts from the upgradeable set. For now incorrect contracts from
the “vanilla” set are used in a mixed case with upgradeable ones. Such approach can create
collisions, affect the development and create unpredictable issues in the runtime.
Vault.sol: IERC20, SafeMath and ECDSA
StakingData.sol: SafeMath
Burn.sol: SafeMath
BaseContract.sol: SafeMath
RoundData.sol: SafeMath
StakeData.sol: SafeMath

**Recommendation**:

Fix the contracts

## Medium Risk

### Unused storage variable

**Description**

BaseContract.sol, _ReentrantCheck
Vault.sol, _ReentrantCheck
Variables are not used in the code.

**Recommendation**: 

Remove unused variables.

### Solidity version update

**Description**

The solidity version should be updated. Throughout the project (including interfaces).
Issue is classified as Medium, because it is included to the list of standard smart contracts’
vulnerabilities. Currently used version (0.7.5) is not the last in the line, which contradicts the
standard checklist.

**Recommendation**: 

You need to update the solidity version to the latest one in the branch -
0.7.6.

## Low Risk

### Unused local variable

**Description**

StakingData.sol, line 256.
Variable maxIteration is unused in the contract. Review the functionality or remove the
variable.

**Recommendation**: 

Review the functionality or remove the variable.

### Unused internal constants

**Description**

Constants STATUS_DEFAULT and STATUS_CANCELED declared in Burn.sol are never used in
Burn.sol

**Recommendation**: 

review the functionality or remove constants.

### Unused method for paused status

**Description**

Vault.sol, 437 _ifNotPaused() is never used.
So it makes function isPaused() from iTrustVaultFactory.sol unused as well.
Also these method are actual duplicates for isActiveVault() method, so isPaused() can be safely
removed.

**Recommendation**:

Remove unused method.

### Re-use local variable

**Description**

Vault.sol, line 188, “require(msg.value == _AdminFee)”
Variable adminFee was added for gas savings and can be re-used in the expression instead of
_AdminFee.

**Recommendation**:

Re-use local variable

### Ignored return value

**Description**

ITrustVaultFactory.sol, createVault() ignores return value by stakingDataContract.addVault()

**Recommendation**:

consider adding require statement.

### Unused functions

**Description**

ITrustVaultFactory.sol: isPaused, _onlyAdmin
Burn.sol: _vaultAddress, _getStartOfDayTimeStamp, validateFactory, _valueCheck
GovernanceDistribution.sol: _getStartOfDayTimeStamp
StakingData.sol: getTotalSupplyForBlock, getHoldingsForIndexAndBlock,
getNumberOfStakingAddresses

**Recommendation**:

Remove unused functions

### Useless boolean return

**Description**

StakingData.sol, endRound(), addVault() - always return true, which makes the return value
useless.

**Recommendation**:

remove return value.

### Boolean equality comparison

**Description**

-require(bool)(_AdminList[newAddress] == false) (iTrustVaultFactory.sol#70)
-require(bool)(_TrustedSigners[newAddress] == false) (iTrustVaultFactory.sol#83)
-_TrustedSigners[account] == true (iTrustVaultFactory.sol#88)
-_AdminList[account] == true (iTrustVaultFactory.sol#144)
-_VaultStatus[msg.sender] == false (iTrustVaultFactory.sol#148)
-_VaultStatus[vaultAddress] == true (iTrustVaultFactory.sol#152)
-require(bool,string)(_AdminList[msg.sender] == true) (iTrustVaultFactory.sol#156-159)
-require(bool,string)(_AdminList[msg.sender] == true) (iTrustVaultFactory.sol#33)
-require(bool)(vaultFactory.isPaused() == false) (vaults\Vault.sol#439)

**Recommendation**:

Use boolean values directly without equality comparison

### Use storage pointer

**Description**

StakingData.sol, line 308, _getAllAcountUnstakesForAddress()
StakingData.sol, line 287, _getAccountStakesForAddress()
StakingData.sol, line 225, _getRoundRewardsForAddress()
RoundData.sol, line 15, endRound()
Burn.sol, line 64, getCurrentBurnData()
Burn.sol, line 77, startBurn()
Burn.sol, line 140, endBurn()
Consider usage of storage pointer to mapping member in order to get gas savings since the
function has several calls to this data.

**Recommendation**:

Use storage pointer.

## Informational

### Consider usage of exponential notation

**Description**

There are several places with literals with too many digits. Consider usage of constants for
them with exponential notation. It will increase the readability of the code and decrease the
chance of the typo error in the number of digits.
div(1000000000000000000) (vaults\Burn.sol#115)
div(1000000000000000000) (vaults\StakingData.sol#367)

**Recommendation**:

Use “snake” literals form or use exponential notation.

### Use standard ReentrancyGuard

**Description**

Throughout the project the variable _Locked together with _nonReentrant() function are used
for the reentrancy prevention, though, for the safety of further development it is
recommended to use standard ReentrancyGuard with modifier. It will increase the overall
code quality.

**Recommendation**:

Use standard ReentrancyGuard.
