**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Possible Inflationary Attack In ynETH May Allow Attackers To Get An Unfair Amount of Shares

**Severity**: High

**Status**: Resolved

**Description**

The ynETH.sol contract relies on the execution layer receiver and the consensus layer receivers to determine the amount of shares to distribute to the user in the form of ynETH tokens. Malicious users could forcibly transfer a certain amount of Eth into the execution layer receiver, trigger processRewards (which anybody can call) in the rewards distributor then deposit 1 wei of Ether into ynETH to cause the next user to receive an unfair amount of tokens. In addition to this, the initial user may be minted a disproportionate amount of ynETH tokens as the first depositor will be minted at a rate of 1:1 indefinitely and subsequent users are minted less (depending on the balance of incoming rewards in which case, significantly less). 

**Recommendation**: 

It’s recommended that deposits in the ynETH contract are bootstrapped similarly to the ynLSD contract to make the bug considerably more expensive to trigger.


## Medium Risk

### Deprecated ETH Transfer Method May Prevent Validators From Being Registered 

**Severity**: Medium

**Status**: Resolved

**Description**

The `ynETH.withdrawETH()` method is used by the `StakingNodesManager` contract in order to withdraw Ether and register a new validator which uses `payable(*).transfer(toAmount)`. The original transfer method uses a fixed stipend of 2,300 gas units which may not be sufficient for some contracts to process the transfer resulting in a revert, preventing the registration of a validator.

**Recommendation**: 

It’s recommended that a low level .call() is used to transfer Ether between contracts and EOAs.

### No Storage Gaps Present In Upgradeable ynBase.sol Contract May Cause Storage Collisions On Upgrade

**Severity**: Medium

**Status**: Invalid

**Description**

For upgradeable contracts it’s recommended that storage gaps are introduced to allow for developers to freely add new state variables in the next iteration of contracts without disrupting the storage compatibility with existing deployed contracts. Insufficient storage space in the base may cause storage collisions when attempting to upgrade the contracts to its next version.

**Recommendation**: 

It’s recommended that storage gaps are introduced in the ynBase.sol contract to remove the likelihood of storage collisions. 


### Lack Of Checks To Prevent Adding Duplicate Assets in the Initialize() Functions 

**Severity**: Medium

**Status**: Acknowledged

**Description**

There exists no checks to prevent adding duplicate assets when attempting to initialize the ynLSD contract. The impact of duplicate assets lies in the totalAssets() function which determines the shares distributed to the users via the deposit() function. This may result in a larger value than expected.

**Recommendation**: 

It’s recommended that the ynLSD contract checks for its corresponding EigenLayer Strategy contract in order to prove existence when reinitializing or check if the asset exists in the assets array although the latter could be quite gas intensive.
Client comment: The decision is to acknowledge and leave this as is. The initialization is performed by the YieldNest DAO at launch time and presence of duplicates would be assumed to be avoided. If the ynLSD contract has duplicate assets it's immediately obvious at initialization time and is considered forfeit.

## Informational

### Unused Custom Errors

**Severity**: Informational

**Status**: Resolved

**Description**

The following custom errors are not used throughout the codebase:
StakingNode.sol, L52-L55
StakingNodesManager.sol, L42, 45
LSDStakingNode.sol, L35-L36

**Recommendation**: 

It’s recommended that these custom errors are removed.

### Multiple Read Operations Against Storage Variables

**Severity**: Informational/Gas

**Status**: Resolved

**Description**

The `withdrawETH()` function of the ynETH contract, the `totalDepositedInPool` variable is read multiple times directly from storage. The SLOAD opcode (cold read) will cost 2,100 gas units and 100 units of gas every read after that (warm read) which can add up to be a considerable amount.

**Recommendation**: 

It’s recommended that storage variables across the code base are first cached to memory which leverages the MLOAD instruction and will only code a minimum of 3 gas units for each read operation.
