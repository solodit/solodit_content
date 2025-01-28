**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Solidity files are not pointing to specific Solidity compiler version.

**Recommendation**:

Use specific Solidity compiler version.

### Method withdraw of Vault contract is not triggering recalculation of SharePriceCheckpoint.

### Calling twice safeApprove or approve methods and passing first time 0 and second time amount of tokens does not solve potential double spending problem as they should be executed as separate transactions and after first transaction mined to block, user has to verify that spender did not spend any tokens approved before.

**Recommendation**:

Reimplement mitigation of double spend problem or remove approval to 0 as it does not
handle double spending problem.

### VaultDAI, VaultUSDC, VaultUSDT, AaveStrategy, CompoundStrategy contracts are missing predefined list of addresses that can be used to verify correct initialization of them as they specifically have to be used with specific contracts.

**Recommendation**:

Define allowed list of contracts addresses with reference to chain id.

### Governance contract is not audited as repository is not including it.

**Recommendation**:

Include reference to existing governance contract or add it to repository.

## Medium Risk

### Solidity files has no license declaration.

**Recommendation**:

Specify license in every Solidity file.

### package.json, development dependency added to dependencies.

**Recommendation**:

It is expected that development dependencies should be located in devDependency.

### Vault contract has incorrect event names (dFeeDeposited, totalFeeCollected), it is expected that event names are declared in PascalCase.

### Method finalizeUpgrade of Vault contract can be called multiple times, that can cause unexpected side effects.

### ComptrollerInterface is not reflecting updated state property `markets` output arguments.

**Recommendation**:

Add extra bool output variable.

### DRY, slot manipulation methods (setBoolean, getBoolean, setAddress, getAddress, setUint256, getUint256) are duplicated across two contracts.

**Recommendation**:

Move that methods to separate contract or library.

### AAveStrategy & CompoundStategy methods (startSaving, investAllUnderlying, withdrawAllToVault, withdrawToVault) can be called by Controller or Governance bypassing Vault without recalculation of sharePriceCheckpoint.
## Low Risk

### ClaimRewardProxy.removeRewardDistributor, FundDivisionStrategy.unwhitelistStrategy methods are using nonoptimal removal approach.

**Recommendation**:

Reassign last element to position of element to be deleted and call method pop on that array.

### Uniswap interface contracts should be added as NPM dependency instead of copying sources.

