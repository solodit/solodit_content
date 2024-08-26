**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Vulnerability in Reward Distribution Mechanism Allowing Fund Theft in SubnetActorManagerFacet.sol

**Severity**: Critical 

**Status**: Resolved

**Location**: SubnetActorManagerFacet.sol contract, distributeRewardToRelayers function

**Description**

A critical vulnerability exists in the IPC protocol's distributeRewardToRelayers function of the SubnetActorManagerFacet.sol contract. This vulnerability allows an attacker to drain funds from a subnet by exploiting a sequence of operations involving crafted messages and checkpointing.
**Attack Process**:
- Initial Requirement: The attacker must have been a rewarded relayer for a past checkpoint.
- Message Crafting: The attacker creates a custom message targeting the distributeRewardToRelayers function. This message is crafted to specify a height for reward eligibility and a large reward amount.
- Checkpoint Commitment: The commitBottomUpCheckpoint function in GatewayRouterFacet.sol is invoked, executing the attacker’s crafted message.
- Malicious Reward Execution: The distributeRewardToRelayers function is executed with the parameters set by the attacker, bypassing security checks due to its origin from the gateway (satisfying the onlyGateway modifier).
- Fund Drainage: The attacker sets a high reward amount in the crafted message, leading to an unauthorized distribution of subnet funds.
- 
**Vulnerability Details**:
  
In the GatewayRouterFacet, the commitBottomUpCheckpoint function triggers _applyMessages, which is also part of the applyCrossMessages function.
When a cross message (crossMsg) is executed, if crossMsg.message.to.rawAddress is set to SubnetActorManagerFacet and crossMsg.message.method to distributeRewardToRelayers, the attack unfolds.
The attacker manipulates crossMsg.message.params to set desired height and value parameters by encoding function parameter data, similar to an encoded call.
**Entry Point for Attack**: The attacker utilizes the IPC protocol to craft custom call data on functions like sendUserXnetMessage in GatewayMessengerFacet. This crafted message is then applied during checkpoint commitment, triggering the vulnerability.

**Recommendation**:

To enhance the security and integrity of the protocol, a crucial recommendation is to implement a restriction within the system that disallows the use of protocol contract addresses as target addresses when crafting messages. This measure aims to prevent potential exploits or manipulations that might arise from interactions with protocol contract addresses.

### Arbitrary amount can be withdrawn (stolen)

**Severity**: Critical

**Status**: Resolved

GatewayManagerFacet.sol - A vulnerability is identified in the `releaseRewardForRelayer` function. The issue allows for arbitrary amount inputs without proper validation, potentially leading to unauthorized fund drainage from the Gateway.
```solidity
   function releaseRewardForRelayer(uint256 amount) external nonReentrant {
        if (amount == 0) {
            revert CannotReleaseZero();
        }

        (bool registered, Subnet storage subnet) = LibGateway.getSubnet(msg.sender);
        if (!registered) {
            revert NotRegisteredSubnet();
        }

        payable(subnet.id.getActor()).sendValue(amount);
    }
```
The `releaseRewardForRelayer` function does not adequately validate the input amount, allowing arbitrary values to be specified. Given that custom subnets are allowed in the project, this presents an opportunity for malicious actors to drain funds from the Gateway by providing arbitrary and unauthorized amount values.

**Recommendation** - 

Validate Amount Input: Implement robust validation checks on the amount parameter to ensure that it is within acceptable and authorized limits.

### Handling Incompatibility with Fee-on-Transfer, Deflationary, and Rebasing Tokens in `GatewayManagerFacet`

**Severity** - High

**Status** - Acknowledged

**Overview**

The GatewayManagerFacet contract facilitates token funding operations across subnets through its `fundWithToken` function. This function is designed to lock a specified amount of ERC20 tokens from the user's address and mint a corresponding amount in the destination address within a subnet. However, an oversight has been identified when interacting with fee-on-transfer (FoT), deflationary, or rebasing tokens. These token types inherently modify the transfer amount through fees, deflation, or rebasing mechanisms, leading to discrepancies between the specified and actually transferred amounts.

Function Analysis Function: fundWithToken(SubnetID calldata subnetId, FvmAddress calldata to, uint256 amount)

**Purpose:**

To fund a specified address within a subnet with a specified amount of ERC20 tokens. Process: Validates the token's compliance with the ERC20 standard. Locks the specified token amount into custody. Creates and commits a top-down message to mint the equivalent token supply on the destination subnet. Core Issue The primary issue arises from the contract's failure to account for the actual token balance transfer discrepancies in the case of FoT, deflationary, or rebasing tokens. The lock function utilizes safeTransferFrom to move tokens from the user to the contract without accounting for the potential reduction in the transferred amount due to fees or token supply adjustments. 

**Impact**: 

Accounting Errors: The protocol assumes the locked amount matches the specified amount, disregarding the actual amount received. This discrepancy leads to the minting of more tokens on the destination subnet than what is locked in the contract. Integrity of Token Supply: Such inconsistencies can inflate the token supply on the subnet, undermining the token's economic model and potentially affecting its value and user trust. 

**Recommendation**:

l A robust solution involves adjusting the fundWithToken workflow to accurately reflect the transferred token amount post-interaction with FoT, deflationary, or rebasing tokens. The proposal includes: Pre- and Post-Transfer Balance Checks: Implement balance checks before and after the execution of safeTransferFrom within the lock function to determine the actual transferred amount. Adjustment of Minted Amount: Adjust the amount to be minted on the destination subnet based on the actual received amount rather than the initially specified amount.







### Validators with dust collateral can join the network

**Severity**: High

**Status**: Acknowledged

SubnetActorManagerFacet.sol - In join function , validators are allowed to join the network to be added to `genesisValidators` in exchange for dust values as low as 1 native coin. This in turn can flood the `s.genesisValidators` array. There is a minimum stake mechanism (i.e. `s.minActivationCollateral` ) but it is meant for bootstrapping the subnet. Validators though can be added arbitrarily while the subnet is not bootstrapped yet. The only cost hindering this spamming of validators is the gas cost.
It is worth noting that a large number of `genesisValidators` adds up to the size of the for loop in `LibStaking.depositWithConfirm` function as shown in the following:

```solidity
   function depositWithConfirm(address validator, uint256 amount) internal {
        ...
        if (!s.bootstrapped) {
            // add to initial validators avoiding duplicates if it
            // is a genesis validator.
            bool alreadyValidator;
            uint256 length = s.genesisValidators.length;
            for (uint256 i; i < length; ) {
                if (s.genesisValidators[i].addr == validator) {
                    alreadyValidator = true;
                    break;
                }
                unchecked {
                    ++i;
                }
            }
            if (!alreadyValidator) {
                uint256 collateral = s.validatorSet.validators[validator].confirmedCollateral;
                Validator memory val = Validator({
                    addr: validator,
                    weight: collateral,
                    metadata: s.validatorSet.validators[validator].metadata
                });
                s.genesisValidators.push(val);
            }
        }
    }
```
This increases the gas cost of the operation as more spammed `genesisValidators` are added up. Also there is a potential risk of denial-of-service if the for loop is exceedingly big (ps. not a likely scenario due to the gas cost that will be incurred on attacker to add up many validators).
Similarly this issue can potentially be caused by SubnetActorManagerFacet.stake function as shown in the following:
```solidity
   function stake() external payable whenNotPaused notKilled {
        ...
        // AUDIT:  dust stakers are allowed
        if (msg.value == 0) {
            revert CollateralIsZero();
        }
        ...
        // AUDIT: genesisValidators are added up here
        if (!s.bootstrapped) {
            LibStaking.depositWithConfirm(msg.sender, msg.value);
            return;
        }
        LibStaking.deposit(msg.sender, msg.value);
    }
```
**Recommendation** - 

This issue can be avoided joining the network demands a certain amount of stake to be paid upfront.

**Fix**:  Client acknowledged that the risk is low at the moment, in addition, this is to be addressed in the future.

### Reward Distribution Mechanism in `submitCheckpoint` Function can be gamed 

**Severity**: High

**Status**: Acknowledged

**Location**: SubnetActorManagerFacet.sol

**Description**

The current implementation of the `submitCheckpoint` function in the contract allows for potential gaming of the reward system by validators. A significant loophole exists where a relayer can receive rewards even when submitting a checkpoint late. This opens up a scenario where a relayer can listen to other relayers' submissions and submit within the same epoch, potentially even front-running these transactions. This behavior leads to the dilution of fee distribution, as more relayers share the fees, thereby reducing the intended rewards for timely and legitimate submissions.
**Implications**:
Dilution of Rewards: Genuine relayers who submit checkpoints promptly might receive lower rewards due to the increased number of participants in the reward pool, including those who submit late.
Incentive Misalignment: This issue could lead to a scenario where relayers are incentivized to wait and copy other submissions rather than participating in a timely and honest manner.
Potential for Manipulation: Malicious actors could exploit this vulnerability to consistently gain rewards without contributing meaningful work.

**Recommendation** :

Implement Commit-Reveal Scheme: Redesign the reward mechanism using a commit-reveal scheme. In this approach, relayers would first commit to a checkpoint without revealing their identity or submission details. After a designated commit period, a separate reveal phase would allow relayers to disclose their submissions. This method can prevent front-running and copying of submissions. This is a suggestion, there may be other approaches more appropriate for this protocol.

### Vulnerability in `kill()` Function Allowing Premature Disruption in `SubnetActorManagerFacet`

**Severity**: High

**Status**: Resolved 

**Location**: SubnetActorManagerFacet.sol

**Description**

The `kill()` function in the `SubnetActorManagerFacet` contract, which is designed to deactivate the subnet when all validators have left, contains a significant vulnerability. The function can be called by any external actor, and when executed, it essentially disables most of the subnet's functionality. This is due to the widespread use of the `notKilled` modifier in various functions within the contract. A critical issue arises at the initial deployment phase of the `SubnetActorManagerFacet`, as it starts without any validators. This allows a potential attacker (griefer) to monitor new contract deployments and call the `kill()` function before any validators are added. This action renders the newly deployed contract useless. If executed on a large scale, this vulnerability can severely disrupt the deployment and operation of subnets within the system.
This flaw can be used to launch a denial-of-service attack against the network, preventing the effective use and deployment of new subnets 
**Recommendation**:  

Allow `kill()` to be callable after a prespecified time has passed after deployment.


### Deletion does not delete all values

**Severity**: High

**Status**: Resolved

**Description**

In LibQuorum.sol, the function `pruneQuorums()` can be problematic on line: 143 
```solidity
delete self.quorumSignatureSenders[h];
```
That's because `quorumSignatureSenders` is a mapping containing a struct which is containing a mapping. And deleting the outer mapping will not delete inner mapping contents. This could result in dangerous issues as it could lead to reuse of signatures or other undiscovered issues.

See: Openzeppelin EnumerableSet warning: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L33

Crytic's Deletion bug: https://github.com/crytic/slither/wiki/Detector-Documentation#deletion-on-mapping-containing-a-structure

**Recommendation**: It is advised to clear the inner mappings properly to avoid any issues.

**Comments**: This issue was fixed in https://github.com/consensus-shipyard/ipc/pull/604

## Medium Risk

### Potential Gas Limitation Issue with `.transfer()` Method in LibStaking.sol

**Severity**: Medium

**Status**: Resolved 

**Location**: LibStaking.sol contract, Lines 90 and 443

**Description**

The `.transfer()` method is used in two instances within the LibStaking.sol contract (specifically on lines 90 and 443) to send Ether. However, using `.transfer()` poses a risk due to its 2300 gas stipend limitation, which may not be sufficient for all recipient contracts and could lead to failed transactions. T

**Issue Details**:

Gas Stipend Limitation: The `.transfer()` method forwards exactly 2300 gas to the recipient, which is only enough for basic operations like logging an event. This can be insufficient if the recipient contract performs more complex operations in its fallback function.
Risk of Transaction Failure: If the recipient contract's fallback function requires more than 2300 gas (e.g., performing state changes, emitting events, or calling other functions), the transaction will fail.

**Recommendation**: 

It’s better to use the `sendValue` method

### External call is fed the wrong `QuorumObjKind` Parameter

**Severity**: Medium

**Status**: Resolved

**Description**

GatewayRouterFacet.sol - In `execBottomUpMsgBatch` function, the kind argument for `distributeRewardToRelayers` (i.e. in the encodeCall) should be `QuoromObjKind.BottomUpMsgBatch` instead of `QuoromObjKind.Checkpoint`.
```solidity
Address.functionCallWithValue({
    target: msg.sender,
    data: abi.encodeCall(
        ISubnetActor.distributeRewardToRelayers,
        (block.number, totalFee, QuorumObjKind.Checkpoint)
    ),
    value: totalFee
});
```
The issue is limited though by the fact that this feature is not yet implemented in SubnetActorManagerFacet as shown here:
```solidity
       } else if (kind == QuorumObjKind.BottomUpMsgBatch) {
            // FIXME: The distribution of rewards for batches can't be done
            // as for checkpoints (due to how they are submitted). As
            // we are running out of time, we'll defer this for the future.
            revert MethodNotAllowed("rewards not defined for batches");
        }
```
But for customly implemented subnets that can pose a serious issue if the implemented subnet is not aware about the lack of this feature.
**Recommendation** 

Apply the correct kind in encodeCall or reject that transaction the GatewayRouterFacet as well since the feature is not enabled yet.
**Fix**: That part is removed as of commit 4a2acb6  hence making the issue no longer relevant.

### Marked active despite not having enough collateral

**Severity**: Medium

**Status**: Resolved

**Description**

GatewayManagerFacet.sol - In `register()` function, collateral is not being validated to be greater than `s.minStake`. Consequently we end up having a subnet status marked as Status.Active as shown in the following:
```solidity
        subnet.id = subnetId;
        subnet.stake = collateral;
        subnet.status = Status.Active;
        subnet.genesisEpoch = block.number;
        subnet.circSupply = genesisCircSupply;
```
Noting that this contradicts the implementation of addStake function which asserts that the stake becomes greater than a given threshold (i.e. s.minStake) in order to have an active status as shown in the following:
```solidity
       if (subnet.status == Status.Inactive) {
            if (subnet.stake >= s.minStake) {
                subnet.status = Status.Active;
            }
        }
```
**Recommendation**

Apply the required validation method to ensure that collateral is the intended value to have an active subnet. 
**Fix**: Issue is fixed in commit 2440ac2 by removing   `subnet.status = Status.Active;`. Therefore the subnet is no longer marked active on calling register to avoid that mislabelling if collateral is not enough.
**Client comment**: Invalid. There is no such vulnerability in the latest codebase after implementing changes in the protocol - https://www.google.com/url?q=https://github.com/consensus-shipyard/ipc-monorepo/blob/main/contracts/src/subnet/SubnetActorManagerFacet.sol%23L107-L254&sa=D&source=docs&ust=1704927823490184&usg=AOvVaw3-GVrDm8nvR39T5f8sH0Xv

### Use < Instead of <=

**Severity** - Medium

**Status** - Resolved

**Description**

Case 1:

In the function createBottomUpMsgBatch (L353 in GatewayRouterFacet.sol) we check that if we are trying to create a batch from the future and the check for that is ->
```solidity
if (batch.blockHeight % s.bottomUpMsgBatchPeriod != 0 || block.number <= batch.blockHeight) {
            revert InvalidBatchEpoch();        }
```
This check would make the code revert even if `block.number == batch.blockHeight` which is not a block from the future but the current block.

Case 2:

In the `register()` function (GatewayManagerFacet.sol L32) ,it is checked that route length after the addition of the subnet i.e. `route.length + 1` must not exceed `maxTreeDepth`.
```solidity
if (s.networkName.route.length + 1 >= s.maxTreeDepth) {
            revert MethodNotAllowed(ERR_CHILD_SUBNET_NOT_ALLOWED);
        }
```
But this condition would revert even when new length after addition is equal to `maxTreeDepth` while the value `maxTreeDepth` should be allowed . 


**Recommendation**:

Change the statement to 
```solidity
if (batch.blockHeight % s.bottomUpMsgBatchPeriod != 0 || block.number < batch.blockHeight) {
            revert InvalidBatchEpoch();
        }
```
And  , 
```solidity
if (s.networkName.route.length + 1 > s.maxTreeDepth) {
            revert MethodNotAllowed(ERR_CHILD_SUBNET_NOT_ALLOWED);
        }
```




### `commitCheckpoint` Returns Execution Without Rewarding The Relayer

**Severity** - Medium

**Status** - Resolved

**Description**

Function `commitCheckpoint()` (L41 GatewayRouterFacet.sol) commits a verified checkpoint and rewards the relayer if `checkpointRelayerRewards` is set to true . But a value of 0 is sent to the function `distributeRewardsToRelayer` at L64  , this would just return execution and not reward the relayer.

**Recommendation**: 

Reward the relayer appropriately



### `SubnetKeys[]` Should Be Removed 

**Severity** - Medium

**Status** - Resolved

**Description**


Inside the `kill()` function in the contract `GatewayManagerFacet.sol` , we decrease the `totalSubnets` at L135 , then delete `subnets[subnet.id.toHash()]` .  
Since `subnetKeys` array hold the keys of the registered subnets , `subnetKeys[subnet.id]` should also be removed when a subnet is killed.

**Recommendation**:

Along with `totalSubnets`  and `subnets[]` also update the `subnetKeys[]` when a subnet is killed



### Deletion Should Be Done After Reading Validator Addresses

**Severity** - Medium

**Status** - Resolved

In the function `pruneQuorums()` (L133 LibQuorum.sol) `self.quorumSignatureSenders[h];` is deleted at L143 , then at L145 `values()` are read from the same `self.quorumSignatureSenders[h]` which has been deleted previously , therefore , reading empty values.

**Recommendation**: 

Make the deletion after reading values from `self.quorumSignatureSenders[h];`

## Low Risk


### Fee For Funding A Subnet Is Hardcoded To 0

**Severity** - Low

**Status** - Acknowledged

**Description**

It is currently decided that funding a subnet is free , but according to the comment at L144 GatewayManagerFacet.sol “There may be an associated fee that gets distributed to validators”

But the fee is hardcoded to 0 at L170

**Recommendation**: Instead of hardcoding the fee have a setter which sets the fee used depending on the decision.

**Client comment**: Accepted. We are going to remove the Funding mechanism. That will fix this and several critical issues. It will be redesigned and implemented later. The issues with low/info severity are tracked here - https://github.com/consensus-shipyard/ipc/issues/536


### Input lack validation to be non-zero

**Severity**: Low

**Status**: Acknowledged

**Description**

LibGateway.sol - In `storeBottomUpCheckpoint` function, the input checkpoint is not being validated. It accepts a zero value of checkpoint.blockHeight. Note that if blockHeight is zero it implies the non-existence of a checkpoint to begin with.
```solidity
   function storeBottomUpCheckpoint(
        BottomUpCheckpoint memory checkpoint
    ) internal {
        GatewayActorStorage storage s = LibGatewayActorStorage.appStorage();
        s.bottomUpCheckpoints[checkpoint.blockHeight] = checkpoint;
    }
```
That library method is being used in GatewayRouterFacet as follow:
```solidity
   function createBottomUpCheckpoint(
        BottomUpCheckpoint calldata checkpoint,
        bytes32 membershipRootHash,
        uint256 membershipWeight
    ) external systemActorOnly {
        if (checkpoint.blockHeight % s.bottomUpCheckPeriod != 0) {
            revert InvalidCheckpointEpoch();
        }
        if (LibGateway.bottomUpCheckpointExists(checkpoint.blockHeight)) {
            revert CheckpointAlreadyExists();
        }

        LibQuorum.createQuorumInfo({
            self: s.checkpointQuorumMap,
            objHeight: checkpoint.blockHeight,
            objHash: keccak256(abi.encode(checkpoint)),
            membershipRootHash: membershipRootHash,
            membershipWeight: membershipWeight,
            majorityPercentage: s.majorityPercentage
        });
        LibGateway.storeBottomUpCheckpoint(checkpoint);
    }
```
This shows that `checkpoint.blockHeight` can pass through with a zero value: `if (checkpoint.blockHeight % s.bottomUpCheckPeriod != 0)`. But the issue is limited because of `systemActorOnly` modifier which limits the access of this function.

**Recommendation**

Apply the necessary input validation.

### Potential Inconsistency in Handling `powerScale` in `SubnetActorDiamond.sol` Constructor

**Severity**: Low 

**Status**: Acknowledged 

**Location**: SubnetActorDiamond.sol contract, Constructor

**Description**

The constructor of the `SubnetActorDiamond.sol` contract allows for the setting of a `powerScale` parameter of type int8. There is a check in place to ensure that `powerScale` does not exceed 18. However, there is no corresponding check for the lower bound, particularly for negative values. Given that `powerScale` is of type int8, it can hold negative values, which might lead to unintended behavior or inconsistencies in the contract's logic if the contract allows a large negative value.
**Issue Details**:
Type of powerScale: The variable `powerScale` is of type int8, which allows for negative values.
Lack of Lower Bound Check: The current implementation only checks if powerScale is greater than 18 and does not account for negative values.
Potential Unintended Behavior: Without a lower bound check, large negative values of powerScale might lead to unexpected behavior, depending on how powerScale is used in the contract.

**Recommendation**: 

Implement Lower Bound Check: Add a validation check to ensure powerScale is within a reasonable range for negative values as well

## Informational

### Exposure to Reentrance

**Severity**: Informational

**Status**: Resolved

**Description**

SubnetActorManagerFacet.sol - Function `unstake()` is exposed to reentrancy as well as cross-function reentrancy. The reentrancy is can be triggered when the execution reaches `LibStaking.withdrawWithConfirm(msg.sender, amount);`. The severity of this finding is not serious because the function adheres to checks-effects-interactions pattern, hence no scenario found in which the attacker would be incentivised to exploit a reentancy to acquire any sort of unjust gain.
```solidity
   function unstake(uint256 amount) external whenNotPaused notKilled {
        // disbling validator changes for federated validation subnets (at least for now
        // until a more complex mechanism is implemented).
        enforceCollateralValidation();

        if (amount == 0) {
            revert CannotReleaseZero();
        }

        uint256 collateral = LibStaking.totalValidatorCollateral(msg.sender);

        if (collateral == 0) {
            revert NotValidator(msg.sender);
        }
        if (collateral <= amount) {
            revert NotEnoughCollateral();
        }
        if (!s.bootstrapped) {
            LibStaking.withdrawWithConfirm(msg.sender, amount);
            return;
        }

        LibStaking.withdraw(msg.sender, amount);
    }
```

**Recommendation** 

Apply reentrancy guard.

**Fix**: Issue addressed and resolved according to recommendation in commit 9a663e5 .

### Redundant return

**Severity**: Informational

**Status**: Resolved

**Description**

SupplySourceHelper.sol - extra line for return ret; is unneeded because ret is already assigned and returning the values it is assigned to by default.
```solidity
   /// @notice Gets the balance in our treasury.
    function balance(SupplySource memory supplySource) internal view returns (uint256 ret) {
        if (supplySource.kind == SupplyKind.Native) {
            ret = address(this).balance;
        } else if (supplySource.kind == SupplyKind.ERC20) {
            ret = IERC20(supplySource.tokenAddress).balanceOf(address(this));
        }
        return ret;
    }
```
**Recommendation**

Omit the return statement.

**Fix**: Issue addressed and resolved according to recommendation in commit 9a663e5 .


### Unnecessary Default Safe Math Usage

**Severity**: Informational

**Status**: Acknowledged

**Description**

GatewayRouterFacet.sol - In `execBottomUpMsgBatch` function, the code snippet performs a subtraction operation on `subnet.circSupply` and `totalAmount` without the possibility of an underflow, as the preceding condition ensures subnet.circSupply is greater than or equal to `totalAmount`. The default safe math operation introduces unnecessary gas overhead, and the operation can be more gas-efficient by using the unchecked statement.
```solidity
       if (subnet.circSupply < totalAmount) {
            revert NotEnoughSubnetCircSupply();
        }

        subnet.circSupply -= totalAmount;
```
While the code snippet correctly ensures that the subtraction operation will not result in an underflow, the unnecessary default safe math usage can be optimized for gas efficiency. Wrapping the operation in an unchecked statement is a recommended practice for situations where the developer can guarantee that underflows will not occur. This enhancement contributes to more efficient gas utilization without compromising safety.

**Recommendation** 

Wrap Operation in unchecked Statement, given that the condition preceding the operation ensures there is no risk of underflow.

### Unnecessary Default Safe Math Usage - 2

**Severity**: Informational

**Status**: Acknowledged

**Description**

GatewayRouterFacet.sol - In `recordWithdraw` function, the code snippet performs a subtraction operation on total and amount without the possibility of an underflow, as the preceding condition ensures total is greater than or equal to amount. The default safe math operation introduces unnecessary gas overhead, and the operation can be more gas-efficient by using the unchecked statement.
```solidity
   function recordWithdraw(ValidatorSet storage validators, address validator, uint256 amount) internal {
        uint256 total = validators.validators[validator].totalCollateral;
        if (total < amount) {
            revert WithdrawExceedingCollateral();
        }

        total -= amount;
        validators.validators[validator].totalCollateral = total;
    }
```
While the code snippet correctly ensures that the subtraction operation will not result in an underflow, the unnecessary default safe math usage can be optimized for gas efficiency. Wrapping the operation in an unchecked statement is a recommended practice for situations where the developer can guarantee that underflows will not occur. This enhancement contributes to more efficient gas utilization without compromising safety.

**Recommendation**

Wrap Operation in unchecked Statement, given that the condition preceding the operation ensures there is no risk of underflow.

### Unnecessary Default Safe Math Usage - 3

**Severity**: Informational

**Status**: Acknowledged

**Description**

SubnetActionManagerFacet.sol - In `preRelease` function, the code snippet performs a subtraction operation on `s.genesisbalance[msg.sender]` and amount without the possibility of an underflow, as the preceding condition ensures `s.genesisbalance[msg.sender]` is greater than or equal to amount. The default safe math operation introduces unnecessary gas overhead, and the operation can be more gas-efficient by using the unchecked statement.

```solidity
   function preRelease(uint256 amount) external nonReentrant {
        ...
        if (s.genesisBalance[msg.sender] < amount) {
            revert NotEnoughBalance();
        }

        s.genesisbalance[msg.sender] -= amount;
        ...
    }
```
While the code snippet correctly ensures that the subtraction operation will not result in an underflow, the unnecessary default safe math usage can be optimized for gas efficiency. Wrapping the operation in an unchecked statement is a recommended practice for situations where the developer can guarantee that underflows will not occur. This enhancement contributes to more efficient gas utilization without compromising safety.

**Recommendation**

Wrap Operation in unchecked Statement, given that the condition preceding the operation ensures there is no risk of underflow.

### Performance Concerns with Nested For Loop

**Severity**: Informational

**Status**: Acknowledged

**Description**

LibQuorum.sol - In pruneQuorums function:
The smart contract includes a nested for loop structure, which, depending on the size of the data being processed, may result in increased gas costs. Nested loops can lead to a denial of service if the loops are large enough to hit the limit.
```solidity
   function pruneQuorums(QuorumMap storage self, uint256 newRetentionHeight) internal {
        ...
        for (uint256 h = oldRetentionHeight; h < newRetentionHeight; ) {
            ...
            for (uint256 i; i < n; ) {
                delete self.quorumSignatures[h][validators[i]];
                unchecked {
                    ++i;
                }
            }
            unchecked {
                ++h;
            }
        }
        self.retentionHeight = newRetentionHeight;
    }
```
**Recommendation**

To optimize performance and reduce gas consumption, it is recommended to review and refactor Loop Logic: Evaluate the necessity of the nested for loop structure and consider refactoring the logic to minimize computational complexity.


### Incorrect Description in pop Function Comments of `LibMaxPQ`  

**Severity**: Informational

**Status**: Resolved 

**Location**: LibMaxPQ

In the LibMaxPQ library, the In the LibMaxPQ library, the comments above the pop function inaccurately describe its functionality. The comment states that the function pops the "minimal value" from the priority queue, but the function's logic suggests that it actually pops the "maximum value." This discrepancy can lead to confusion about the function's behavior.
**Location of Issue**:
**File**: LibMaxPQ
**Function**: pop
Functionality Check:
Based on the provided code snippet and typical implementations of Max Priority Queues, it seems that the largest value in a Max Priority Queue is stored at the root (slot 1). The pop function appears to remove the top element, which would be the largest value in the queue, not the smallest.
**Issue Details**:
Misleading Information: The comment incorrectly indicates that the function pops the minimal value.
Potential for Misinterpretation: Developers or maintainers reading the code could be misled by the comment, potentially leading to incorrect assumptions or use of the function. 
**Fix**: Issue addressed and resolved in commit 632e710 .
**Recommendation for Resolution**:
Correct the Comment: Update the comment to accurately reflect the function's behavior of removing the maximum value from the priority queue.
 above the pop function inaccurately describe its functionality. The comment states that the function pops the "minimal value" from the priority queue, but the function's logic suggests that it actually pops the "maximum value." This discrepancy can lead to confusion about the function's behavior.
**Location of Issue:**
**File**: LibMaxPQ
**Function**: pop
**Functionality Check**:
Based on the provided code snippet and typical implementations of Max Priority Queues, it seems that the largest value in a Max Priority Queue is stored at the root (slot 1). The pop function appears to remove the top element, which would be the largest value in the queue, not the smallest.
**Issue Details**:
**Misleading Information**: The comment incorrectly indicates that the function pops the minimal value.
Potential for Misinterpretation: Developers or maintainers reading the code could be misled by the comment, potentially leading to incorrect assumptions or use of the function.
**Recommendation for Resolution**:
Correct the Comment: Update the comment to accurately reflect the function's behavior of removing the maximum value from the priority queue.

### Potential for Gas Optimization through Tighter Packing of `SubnetActorStorage` Struct

**Severity**: Informational 

**Status**: Resolved 

**Instance 1**: 

Potential for Enhanced Gas Efficiency in ConstructorParams Struct through Variable Reordering
Overview: The ConstructorParams struct currently includes a mixture of data types that may not be optimally ordered for gas efficiency. Solidity's storage optimization mechanisms can be better leveraged by reordering the struct's variables, thus potentially reducing the gas cost associated with deploying and interacting with contracts using this struct.
Current Structure: The struct includes:
SubnetID parentId;
address ipcGatewayAddr;
ConsensusType consensus;
uint256 minActivationCollateral;
uint64 minValidators;
uint64 bottomUpCheckPeriod;
uint8 majorityPercentage;
uint16 activeValidatorsLimit;
uint256 minCrossMsgFee;
int8 powerScale;
Proposed Optimization: The struct can be reorganized to group variables of smaller sizes together, allowing Solidity to pack these into the same 32-byte storage slot.
Instance 2:
The SubnetActorStorage struct in the given contract can benefit from optimization through tighter packing of its variables. Solidity storage variables are laid out in 32-byte slots, and proper organization of these variables can lead to more efficient use of these slots, reducing the gas cost associated with storage operations.
Current Structure: The SubnetActorStorage struct contains a mix of variable types of different sizes (e.g., uint256, uint64, address, bool, int8, and mappings). The current arrangement does not fully utilize the potential for tight packing.
Proposed Optimization: Reorder the variables in SubnetActorStorage to group smaller-sized variables together. This reordering allows Solidity to pack these variables into the same 32-byte storage slots, thereby optimizing storage usage.
Rationale for Suggested Change:
Efficient Storage Slot Usage: Solidity can pack multiple smaller-sized variables (like uint64, uint16, uint8, int8) into a single 32-byte storage slot. Grouping these variables together can minimize the number of storage slots needed.
Gas Cost Reduction: Efficiently using storage slots can lead to reduced gas costs, especially in operations involving contract deployment and state updates.
Maintaining Logical Grouping: It's important to balance optimization with logical structuring of data. The reordering should not compromise the readability and logical grouping of related variables.

**Recommendation**:

Group smaller integer variables together (uint64, uint16, uint8, int8).
Place larger data types like uint256 and address next.
SubnetID and ConsensusType may be placed according to their respective sizes or at the end if they are complex types.

### Enhancing Gas Efficiency in `createSubnetId` Through Caching `subnet.route.length`

**Severity**: Informational 

**Status**: Resolved

**Location**: subnetIdHelper.sol

**Description**

In the `subnetIdHelper` smart contract, specifically within the `createSubnetId` function, there is an opportunity to optimize gas usage by caching the value of subnet.route.length. The current implementation retrieves this value multiple times, which can be more gas-intensive than necessary.
Rationale for Suggested Change:
Gas Efficiency Through Caching:
Accessing the length of a dynamic array (subnet.route.length) multiple times in a function incurs a small but repeated gas cost.

**Recommendation**:

By caching this length in a local variable (`routeLength`), and using it throughout the function, we reduce the number of times the contract has to fetch this property, saving gas.Each access to subnet.route.length can be considered a read operation from the contract's state. Reducing state access where possible is a good practice in smart contract optimization.

### Missing Getter Function for `appliedBottomUpNonce` in GatewayGetterFacet.sol

**Severity**: Informational 

**Status**: Resolved 

**Location**: GatewayGetterFacet.sol

**Description**

The Subnet struct in the GatewayGetterFacet.sol contract includes an `appliedBottomUpNonce` field, but there is no corresponding getter function to retrieve this value. The absence of a getter for appliedBottomUpNonce presents an inconsistency in the contract's interface, potentially hindering the ability to access important contract state information.

**Issue Details**:

**Missing Functionality**: The lack of a getter function for appliedBottomUpNonce limits external visibility into this particular state variable.
Inconsistent Access Patterns: While other fields in the Subnet struct are accessible through getter functions, appliedBottomUpNonce is not, leading to an inconsistent interface.

**Recommendation for Resolution**:

Implement Getter Function: Add a new function to the GatewayGetterFacet.sol contract that allows users to retrieve the appliedBottomUpNonce value for a given subnet. This enhances transparency and usability of the contract.

### Inconsistent Naming in `getAppliedTopDownNonce` Function of GatewayGetterFacet.sol

**Severity**: Informational 

**Status**: Resolved 

**Location**: GatewayGetterFacet.soo

**Description**

In the GatewayGetterFacet.sol contract, the function `getAppliedTopDownNonce` is misleadingly named as it returns the topDownNonce instead of the appliedTopDownNonce. This discrepancy between the function name and its actual behavior can lead to confusion and potential misuse in the context where the function is invoked.
**Issue Details**:
Naming Mismatch: The function name suggests that it returns an appliedTopDownNonce, but the implementation shows that it actually returns the topDownNonce.
Potential Misinterpretation: Users or developers interacting with this function might expect to receive the appliedTopDownNonce, leading to erroneous implementations or interpretations based on the function's output.

**Recommendation for Resolution**:

Rename Function: Consider renaming the function to accurately reflect its functionality, such as getTopDownNonce, to prevent confusion and ensure clarity in the contract's interface.

### Gas Optimization in LibMaxPQ Contract Using Bitwise Shift Operations

**Severity**: informational

**Status**: Resolved 

**Description**

The LibMaxPQ contract currently employs multiplication and division by 2 in its logic (specifically at lines 158, 133, and 119). This informational finding suggests the replacement of these operations with bitwise shift operations as a means to enhance gas efficiency in the contract's execution.
Details of Current Implementation:
Multiplication by 2:
Line 158: x * 2
Line 133: y * 2
Division by 2:
Line 119: z / 2
**Recommendation and Rationale for Suggested Change**:
Gas Efficiency with Bitwise Operations:
Multiplication and Division by 2: These are relatively more expensive in terms of gas usage because they are interpreted as arithmetic operations.
Bitwise Shifts: Replacing these with bitwise left shift (<<) for multiplication and right shift (>>) for division is more efficient. This is because bitwise shifts are simpler operations at the binary level and typically consume less gas.
Left Shift (<<): A single left bitwise shift (x << 1) is equivalent to multiplying by 2 (x * 2).
Right Shift (>>): A single right bitwise shift (z >> 1) is equivalent to dividing by 2 (z / 2).
Solidity and EVM Optimization:
In Solidity, optimizing for gas usage is crucial. Bitwise operations are generally more efficient in the Ethereum Virtual Machine (EVM), leading to lower transaction costs.
No Functional Impact:
Switching to bitwise operations in the mentioned scenarios should not impact the contract's logic. The primary purpose of these operations is to manipulate numerical values for algorithmic logic, and bitwise shifts accomplish the same with better efficiency.

**Recommendation**:

These are the following recommendations:
Replace the multiplication by 2 operations (x * 2, y * 2) with left bitwise shifts (x << 1, y << 1) on lines 158 and 133.
Replace the division by 2 operation (z / 2) with a right bitwise shift (z >> 1) on line 119.
Expected Outcome:
The change is anticipated to yield gas savings per transaction involving these operations. The savings, while potentially modest per transaction, can accumulate to significant amounts over many transactions, contributing to overall contract efficiency.

### Improper naming of the contracts

**Severity**: Informational

**Status**: Acknowledged 

**Description**

The file LibSubnetRegistryStorage is named LibSubnetRegistryStorage but it actually contains just a struct. 

**Recommendation**: 

Consider moving it to structs folder or rename the file to match its actual behavior.


### Incorrect Comment In LibMaxPQ

**Severity** - Informational

**Status** - Resolved

**Description**


The comment at L122 in LibMaxPQ says 
// parent power is not larger than that of the current child, heap condition met.

This is not true , it should be 

// parent power is not smaller than that of the current child, heap condition met.

This is because this is a max heap where parent value is larger than its child nodes.

 **Recommendation**: Correct the statements.


### `minValidator` Check Can Be Done Before To Save Gas

**Severity** - Informational

**Status** - Resolved

**Description**

The check at L560 in SubnetActorManagerFacet.sol (to check if the length is less than minValidators) can be done on the beginning of the function preBootstrapSetFederatedPower , that way if the length is less than the minValidators then it does not need to go through the for loop.

 **Recommendation**:

Have the check at the beginning of the function.


### Code Behaves Differently From The Comment

**Severity** - Informational

**Status** - Resolved

**Description**

The comment at L248 in `SubnetActorManagerFacet` says “adding this check to prevent new validators from joining after the subnet has been bootstrapped”  ,  but the check ,
```solidity
if (s.bootstrapped) {
            enforceCollateralValidation();
        }
```
Does not enforce this, validators still can join with the condition that collateral validation is enforced.

Recommendation: Update the comment or the code
