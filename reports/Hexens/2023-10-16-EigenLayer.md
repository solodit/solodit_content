**Auditors**

[Hexens](https://hexens.io/)

---

# Findings
## High Risk

### [EIG-7] Direct loss of user principal funds when processing a full withdrawal of penalised validator

**Severity:** Critical

**Path:** EigenPod.sol:_processFullWithdrawal#L652-L706

**Description:**  

The EigenPod allows users to proof withdrawals from the beacon chain for connected validators, so the ETH can then be withdrawn from the EigenPod. The function `_processFullWithdrawal` is called whenever a withdrawal has happened after the withdrawable epoch of a validator.

In this function and in the branch where it processes a full withdrawal for a penalised validator (i.e. withdrawal amount is less than `MAX_VALIDATOR_BALANCE_GWEI`) on lines 680-686, it uses `_calculateRestakedBalanceGwei` to convert the withdrawal amount.

This function is normally used to calculate a validator’s balance and the amount of shares to mint in the EigenPodManager. This function also greatly rounds down the amount. 

Here it is used directly on the withdrawal amount and only this rounded down amount is added to `withdrawableRestakedExecutionLayerGwei` and `withdrawalAmountWei`. The rounded down amount will be strictly less than the actual withdrawn amount, but the leftover is basically discarded and becomes stuck in the contract.

For example, if the `MAX_VALIDATOR_BALANCE_GWEI` is 32 ETH and `RESTAKED_BALANCE_OFFSET_GWEI` is 0.75 ETH, then for a withdrawal amount of 30.74 ETH it would round down to 29 ETH, causing almost 1.5 ETH of principal funds to become lost.

```
function _processFullWithdrawal(
    uint40 validatorIndex,
    bytes32 validatorPubkeyHash,
    uint64 withdrawalHappenedTimestamp,
    address recipient,
    uint64 withdrawalAmountGwei,
    ValidatorInfo memory validatorInfo
) internal returns (VerifiedWithdrawal memory) {
    VerifiedWithdrawal memory verifiedWithdrawal;
    uint256 withdrawalAmountWei;

    uint256 currentValidatorRestakedBalanceWei = validatorInfo.restakedBalanceGwei * GWEI_TO_WEI;

    /**
      * If the validator is already withdrawn and additional deposits are made, they will be automatically withdrawn
      * in the beacon chain as a full withdrawal.  Thus such a validator can prove another full withdrawal, and
      * withdraw that ETH via the queuedWithdrawal flow in the strategy manager.
      */
        // if the withdrawal amount is greater than the MAX_VALIDATOR_BALANCE_GWEI (i.e. the max amount restaked on EigenLayer, per ETH validator)
    uint64 maxRestakedBalanceGwei = _calculateRestakedBalanceGwei(MAX_VALIDATOR_BALANCE_GWEI);
    if (withdrawalAmountGwei > maxRestakedBalanceGwei) {
        // then the excess is immediately withdrawable
        verifiedWithdrawal.amountToSend =
            uint256(withdrawalAmountGwei - maxRestakedBalanceGwei) *
            uint256(GWEI_TO_WEI);
        // and the extra execution layer ETH in the contract is MAX_VALIDATOR_BALANCE_GWEI, which must be withdrawn through EigenLayer's normal withdrawal process
        withdrawableRestakedExecutionLayerGwei += maxRestakedBalanceGwei;
        withdrawalAmountWei = maxRestakedBalanceGwei * GWEI_TO_WEI;
    } else {
        // otherwise, just use the full withdrawal amount to continue to "back" the podOwner's remaining shares in EigenLayer
        // (i.e. none is instantly withdrawable)
        withdrawalAmountGwei = _calculateRestakedBalanceGwei(withdrawalAmountGwei);
        withdrawableRestakedExecutionLayerGwei += withdrawalAmountGwei;
        withdrawalAmountWei = withdrawalAmountGwei * GWEI_TO_WEI;
    }
    // if the amount being withdrawn is not equal to the current accounted for validator balance, an update must be made
    if (currentValidatorRestakedBalanceWei != withdrawalAmountWei) {
        verifiedWithdrawal.sharesDelta = _calculateSharesDelta({
            newAmountWei: withdrawalAmountWei,
            currentAmountWei: currentValidatorRestakedBalanceWei
        });
    }


    // now that the validator has been proven to be withdrawn, we can set their restaked balance to 0
    validatorInfo.restakedBalanceGwei = 0;
    validatorInfo.status = VALIDATOR_STATUS.WITHDRAWN;
    validatorInfo.mostRecentBalanceUpdateTimestamp = withdrawalHappenedTimestamp;

    _validatorPubkeyHashToInfo[validatorPubkeyHash] = validatorInfo;

    emit FullWithdrawalRedeemed(validatorIndex, withdrawalHappenedTimestamp, recipient, withdrawalAmountGwei);

    return verifiedWithdrawal;
}

function _calculateRestakedBalanceGwei(uint64 amountGwei) internal view returns (uint64) {
    if (amountGwei <= RESTAKED_BALANCE_OFFSET_GWEI) {
        return 0;
    }
    uint64 effectiveBalanceGwei = uint64(((amountGwei - RESTAKED_BALANCE_OFFSET_GWEI) / GWEI_TO_WEI) * GWEI_TO_WEI);
    return uint64(MathUpgradeable.min(MAX_VALIDATOR_BALANCE_GWEI, effectiveBalanceGwei));
}
```

**Remediation:**  The full withdrawal for a penalised validator should not only use `_calculateRestakedBalanceGwei` on the withdrawal amount or the difference should be calculated and returned to the user immediately by setting `verifiedWithdrawal.amountToSend`.

**Status:**  Fixed



- - -

### [EIG-10] Withdrawal proofs can be forged due to missing index bit size check

**Severity:** Critical

**Path:** BeaconChainProofs.sol:verifyWithdrawal#L269-L399

**Description:**  

In order for the EigenPod to verify and consequently process a withdrawal from the beacon chain, it uses the BeaconChainsProofs' `verifyWithdrawal` function. This function takes various parameters to prove the existence of a supplied Withdrawal struct in the beacon chain state Merkle root.

To do so, it proves 5 different leaves: the block root against the beacon state root, the slot number against the block root, the execution root against the block root, the timestamp against the execution root and finally the withdrawal against the execution root. 

The first proof is not fully checked and it is vulnerable to tampering. Because the other proofs all depend on this first proof, it also influences the others and allows for tampering there as well.

The block root is proven against the beacon state root by first traversing to the historical summaries root in the beacon state. This is done using a constant `HISTORICAL_SUMMARIES_INDEX` which is then concatenated with the `historicalSummaryIndex` that is supplied by the user to choose the right historical summary root from the Merkle tree. This is again concatenated with `blockRootIndex`, also supplied by the user to choose the right block from the historical summary tree.

To make sure that the user does not control the flow of traversal through the Merkle tree, it is important to make sure that the proof lengths are of correct lengths and that the bit size of indexes are not greater than the tree height. This is done correctly for each proof, except for the `historySummaryIndex`, which is missing a size check.

For example, `blockRootIndex` is check on lines 279-282:

```
require(
    withdrawalProof.blockRootIndex < 2 ** BLOCK_ROOTS_TREE_HEIGHT,
    "BeaconChainProofs.verifyWithdrawal: blockRootIndex is too large"
);
```
But `historySummaryIndex` is missing such checks.

This allows a malicious user to provide an index greater than the tree height. If this were a simple, single Merkle tree with one index, then it would not be problem. But in this case we are traversing combinations of multiple trees from the beacon state root to the block root and so it allows for other indexes to be overwritten.

For example, in the first proof the combined index for the proof is calculated using the concatenations as described above:

```
uint256 historicalBlockHeaderIndex = (HISTORICAL_SUMMARIES_INDEX <<
    ((HISTORICAL_SUMMARIES_TREE_HEIGHT + 1) + 1 + (BLOCK_ROOTS_TREE_HEIGHT))) |
    (uint256(withdrawalProof.historicalSummaryIndex) << (1 + (BLOCK_ROOTS_TREE_HEIGHT))) |
    (BLOCK_SUMMARY_ROOT_INDEX << (BLOCK_ROOTS_TREE_HEIGHT)) |
    uint256(withdrawalProof.blockRootIndex);
```
As can be seen, the `historySummaryIndex` is appended on top of the constant `HISTORICAL_SUMMARIES_INDEX` (which should ensure that we first traverse from the beacon state root to the history summaries root). Now that `historySummaryIndex` is unbounded, it becomes possible to overwrite the `HISTORY_SUMMARIES_INDEX` to any value and make the traversal go into any other field of the beacon state instead: https://github.com/ethereum/consensus-specs/blob/dev/specs/capella/beacon-chain.md#beaconstate

In order to exploit this bug and forge a withdrawal proof, it becomes important to plan a path where the proofs will go and result in valid values for the withdrawal struct.

For example, the first proof provides a lot of freedom in traversing from the `historicalSummaryIndex` and `blockRootIndex`. 

The timestamp, slot and execution root proofs can be ignored as the proof lengths are short and they can simply pass with a hash value as leaf. The hash value would be interpreted as timestamp and slot, which will not make any checks fails, rather it gives unique timestamps and slots which could give either full or partial withdrawals depending on the validator’s withdrawable epoch.

Only the withdrawal proof will require some planning, as the withdrawal fields will either have to be some other leaf value or brute-forced hashes (intermediate leaves) in some part of the entire beacon state Merkle tree. Brute-forced hashes would still work, as the only used fields from the withdrawal struct are the validator index (which is parsed into 5 bytes) and the withdrawal amount (which is parsed into 8 bytes, expressed in Gwei and should be not too large).

A working exploit would allow a malicious user to proof withdrawals for themselves or victim users. If the timestamp could be controlled, then it can also be used to proof 0 amount withdrawals for victim users that have a real withdrawal at some timestamp. The timestamp would be set to `true` and they cannot prove the actual withdrawal anymore, locking their ETH.

```function verifyWithdrawal(
        bytes32 beaconStateRoot,
        bytes32[] calldata withdrawalFields,
        WithdrawalProof calldata withdrawalProof
    ) internal view {
        require(
            withdrawalFields.length == 2 ** WITHDRAWAL_FIELD_TREE_HEIGHT,
            "BeaconChainProofs.verifyWithdrawal: withdrawalFields has incorrect length"
        );

        require(
            withdrawalProof.blockRootIndex < 2 ** BLOCK_ROOTS_TREE_HEIGHT,
            "BeaconChainProofs.verifyWithdrawal: blockRootIndex is too large"
        );
        require(
            withdrawalProof.withdrawalIndex < 2 ** WITHDRAWALS_TREE_HEIGHT,
            "BeaconChainProofs.verifyWithdrawal: withdrawalIndex is too large"
        );

        require(
            withdrawalProof.withdrawalProof.length ==
                32 * (EXECUTION_PAYLOAD_HEADER_FIELD_TREE_HEIGHT + WITHDRAWALS_TREE_HEIGHT + 1),
            "BeaconChainProofs.verifyWithdrawal: withdrawalProof has incorrect length"
        );
        require(
            withdrawalProof.executionPayloadProof.length ==
                32 * (BEACON_BLOCK_HEADER_FIELD_TREE_HEIGHT + BEACON_BLOCK_BODY_FIELD_TREE_HEIGHT),
            "BeaconChainProofs.verifyWithdrawal: executionPayloadProof has incorrect length"
        );
        require(
            withdrawalProof.slotProof.length == 32 * (BEACON_BLOCK_HEADER_FIELD_TREE_HEIGHT),
            "BeaconChainProofs.verifyWithdrawal: slotProof has incorrect length"
        );
        require(
            withdrawalProof.timestampProof.length == 32 * (EXECUTION_PAYLOAD_HEADER_FIELD_TREE_HEIGHT),
            "BeaconChainProofs.verifyWithdrawal: timestampProof has incorrect length"
        );

        require(
            withdrawalProof.historicalSummaryBlockRootProof.length ==
                32 *
                    (BEACON_STATE_FIELD_TREE_HEIGHT +
                        (HISTORICAL_SUMMARIES_TREE_HEIGHT + 1) +
                        1 +
                        (BLOCK_ROOTS_TREE_HEIGHT)),
            "BeaconChainProofs.verifyWithdrawal: historicalSummaryBlockRootProof has incorrect length"
        );
        /**
         * Note: Here, the "1" in "1 + (BLOCK_ROOTS_TREE_HEIGHT)" signifies that extra step of choosing the "block_root_summary" within the individual
         * "historical_summary". Everywhere else it signifies merkelize_with_mixin, where the length of an array is hashed with the root of the array,
         * but not here.
         */
        uint256 historicalBlockHeaderIndex = (HISTORICAL_SUMMARIES_INDEX <<
            ((HISTORICAL_SUMMARIES_TREE_HEIGHT + 1) + 1 + (BLOCK_ROOTS_TREE_HEIGHT))) |
            (uint256(withdrawalProof.historicalSummaryIndex) << (1 + (BLOCK_ROOTS_TREE_HEIGHT))) |
            (BLOCK_SUMMARY_ROOT_INDEX << (BLOCK_ROOTS_TREE_HEIGHT)) |
            uint256(withdrawalProof.blockRootIndex);

        require(
            Merkle.verifyInclusionSha256({
                proof: withdrawalProof.historicalSummaryBlockRootProof,
                root: beaconStateRoot,
                leaf: withdrawalProof.blockRoot,
                index: historicalBlockHeaderIndex
            }),
            "BeaconChainProofs.verifyWithdrawal: Invalid historicalsummary merkle proof"
        );

        //Next we verify the slot against the blockRoot
        require(
            Merkle.verifyInclusionSha256({
                proof: withdrawalProof.slotProof,
                root: withdrawalProof.blockRoot,
                leaf: withdrawalProof.slotRoot,
                index: SLOT_INDEX
            }),
            "BeaconChainProofs.verifyWithdrawal: Invalid slot merkle proof"
        );

        {
            // Next we verify the executionPayloadRoot against the blockRoot
            uint256 executionPayloadIndex = (BODY_ROOT_INDEX << (BEACON_BLOCK_BODY_FIELD_TREE_HEIGHT)) |
                EXECUTION_PAYLOAD_INDEX;
            require(
                Merkle.verifyInclusionSha256({
                    proof: withdrawalProof.executionPayloadProof,
                    root: withdrawalProof.blockRoot,
                    leaf: withdrawalProof.executionPayloadRoot,
                    index: executionPayloadIndex
                }),
                "BeaconChainProofs.verifyWithdrawal: Invalid executionPayload merkle proof"
            );
        }

        // Next we verify the timestampRoot against the executionPayload root
        require(
            Merkle.verifyInclusionSha256({
                proof: withdrawalProof.timestampProof,
                root: withdrawalProof.executionPayloadRoot,
                leaf: withdrawalProof.timestampRoot,
                index: TIMESTAMP_INDEX
            }),
            "BeaconChainProofs.verifyWithdrawal: Invalid blockNumber merkle proof"
        );

        {
            /**
             * Next we verify the withdrawal fields against the blockRoot:
             * First we compute the withdrawal_index relative to the blockRoot by concatenating the indexes of all the
             * intermediate root indexes from the bottom of the sub trees (the withdrawal container) to the top, the blockRoot.
             * Then we calculate merkleize the withdrawalFields container to calculate the the withdrawalRoot.
             * Finally we verify the withdrawalRoot against the executionPayloadRoot.
             *
             *
             * Note: Merkleization of the withdrawals root tree uses MerkleizeWithMixin, i.e., the length of the array is hashed with the root of
             * the array.  Thus we shift the WITHDRAWALS_INDEX over by WITHDRAWALS_TREE_HEIGHT + 1 and not just WITHDRAWALS_TREE_HEIGHT.
             */
            uint256 withdrawalIndex = (WITHDRAWALS_INDEX << (WITHDRAWALS_TREE_HEIGHT + 1)) |
                uint256(withdrawalProof.withdrawalIndex);
            bytes32 withdrawalRoot = Merkle.merkleizeSha256(withdrawalFields);
            require(
                Merkle.verifyInclusionSha256({
                    proof: withdrawalProof.withdrawalProof,
                    root: withdrawalProof.executionPayloadRoot,
                    leaf: withdrawalRoot,
                    index: withdrawalIndex
                }),
                "BeaconChainProofs.verifyWithdrawal: Invalid withdrawal merkle proof"
            );
        }
    }
```

**Remediation:**  Check the length of `historicalSummaryIndex`, for example:
```
require(
    withdrawalProof.historicalSummaryIndex < 2 ** HISTORICAL_SUMMARIES_TREE_HEIGHT,
    "BeaconChainProofs.verifyWithdrawal: historicalSummaryIndex is too large"
);
```

**Status:**   Fixed

- - -

### [EIG-14] M1 EigenPods can restake and withdraw without proving and burning shares

**Severity:** Critical

**Path:** EigenPod.sol:withdrawNonBeaconChainETHBalanceWei#L393-L404

**Description:**  

The EigenPod offers 2 functions to withdraw ETH directly without proving a withdrawal. The first one is `withdrawBeforeRestaking`, which requires `hasRestaked` to be `false`. The second one is  `withdrawNonBeaconChainETHBalanceWei`, which is introduced in M2 and takes from a balance counter that is increased upon execution of `receive`. 

The former is to allow for depositing and withdrawing before restaking (and so without proofs) and the latter is to withdraw any ETH that was mistakenly sent to the EigenPod. However, they do not work well together. 

Most M1 EigenPods will still have `hasRestaked` be set to `false`, as proving is not enabled for M1 EigenPods. Once they are upgraded to the M2 implementation, they will have access to both functions.

This can then be exploited to restake and withdraw the stake without proving the withdrawal and consequently without burning the shares, effectively allowing for free minting of shares.

Consider the following scenario:

1. An M1 EigenPod with `hasRestaked` is `false` is upgraded to the M2 implementation.

2. The owner sends 32 ETH to the EigenPod, the `nonBeaconChainETHBalanceWei` increases with 32 ETH.

3. The owner calls `withdrawBeforeRestaking`, which will simply send the entire ETH balance (32 ETH) to the owner.

4. The owner activates restaking, creates a validator and verifies the withdrawal credentials, receiving 32 ETH in shares.

5. The owner exits the validator and the EigenPod receives the 32 ETH principal.

6. The owner can now call `withdrawNonBeaconChainETHBalanceWei` to withdraw the 32 ETH, because `nonBeaconChainETHBalanceWei` is still equal to 32 ETH, bypassing the withdrawal proof and keeping the 32 ETH shares.

7. Repeat (or use multiple validators) for more free shares.

```
receive() external payable {
    nonBeaconChainETHBalanceWei += msg.value;
    emit NonBeaconChainETHReceived(msg.value);
}

function withdrawNonBeaconChainETHBalanceWei(
    address recipient,
    uint256 amountToWithdraw
) external onlyEigenPodOwner {
    require(
        amountToWithdraw <= nonBeaconChainETHBalanceWei,
        "EigenPod.withdrawnonBeaconChainETHBalanceWei: amountToWithdraw is greater than nonBeaconChainETHBalanceWei"
    );
    nonBeaconChainETHBalanceWei -= amountToWithdraw;
    emit NonBeaconChainETHWithdrawn(recipient, amountToWithdraw);
    _sendETH_AsDelayedWithdrawal(recipient, amountToWithdraw);
}

function withdrawBeforeRestaking() external onlyEigenPodOwner hasNeverRestaked {
    _processWithdrawalBeforeRestaking(podOwner);
}

function _processWithdrawalBeforeRestaking(address _podOwner) internal {
    mostRecentWithdrawalTimestamp = uint32(block.timestamp);
    _sendETH_AsDelayedWithdrawal(_podOwner, address(this).balance);
}
```

**Remediation:**  The function `_processWithdrawalBeforeRestaking` should also zero out `nonBeaconChainETHBalanceWei`, as the entire balance will be withdrawn anyway.

**Status:**   Fixed


- - -
## Medium Risk

### [EIG-17] Operator or delegation approver have the power to censor delegated stakers

**Severity:** Medium

**Path:** DelegationManager.sol:undelegate#L213-L257

**Description:**  

The operator or delegation approver that stakers have delegated to, have the power to selectively censor those stakers.

As soon as anyone can register an operator on EigenLayer, an attacker can create operators massively and grief stakers by undelegating them, consequently damaging protocol stability and making users averse to use the protocol.

The operator or delegation approver can call the `undelegate()` function for a particular staker. This will move the staker to the undelegation limbo (call to `forceIntoUndelegationLimbo()`) , and forcefully removing staker’s shares from `StartegyManager` (call to `forceTotalWithdrawal()`).

```    function undelegate(
        address staker
    ) external onlyWhenNotPaused(PAUSED_UNDELEGATION) returns (bytes32 withdrawalRoot) {
        require(isDelegated(staker), "DelegationManager.undelegate: staker must be delegated to undelegate");
        address operator = delegatedTo[staker];
        require(!isOperator(staker), "DelegationManager.undelegate: operators cannot be undelegated");
        require(staker != address(0), "DelegationManager.undelegate: cannot undelegate zero address");
        require(
            msg.sender == staker ||
                msg.sender == operator ||                                       // @audit
                msg.sender == _operatorDetails[operator].delegationApprover,    // @audit
            "DelegationManager.undelegate: caller cannot undelegate staker"
        );
        
        // remove any shares from the delegation system that the staker currently has delegated, if necessary
        // force the staker into "undelegation limbo" in the EigenPodManager if necessary
       if (eigenPodManager.podOwnerHasActiveShares(staker)) {
            uint256 podShares = eigenPodManager.forceIntoUndelegationLimbo(staker, operator); // @audit
            ...
        }
        // force-queue a withdrawal of all of the staker's shares from the StrategyManager, if necessary
        if (strategyManager.stakerStrategyListLength(staker) != 0) {
            IStrategy[] memory strategies;
            uint256[] memory strategyShares;
            (strategies, strategyShares, withdrawalRoot) = strategyManager.forceTotalWithdrawal(staker); // @audit
            ...
    }
```

The staker is able to delegate his shares to another operator only after `withdrawalDelayBlocks` period. Which is 1 week according to the documentation https://github.com/Layr-Labs/eigenlayer-contracts/blob/master/docs/core/StrategyManager.md#strategymanager.

```
    function exitUndelegationLimbo(
        uint256 middlewareTimesIndex,
        bool withdrawFundsFromEigenLayer
    ) external onlyWhenNotPaused(PAUSED_WITHDRAW_RESTAKED_ETH) onlyNotFrozen(msg.sender) nonReentrant {
        ...

        // enforce minimum delay lag
        require(
            limboStartBlock + strategyManager.withdrawalDelayBlocks() <= block.number,  // @audit
            "EigenPodManager.exitUndelegationLimbo: withdrawalDelayBlocks period has not yet passed"
        );
```
```
   function _completeQueuedWithdrawal(
        QueuedWithdrawal calldata queuedWithdrawal,
        IERC20[] calldata tokens,
        uint256 middlewareTimesIndex,
        bool receiveAsTokens
    ) internal onlyNotFrozen(queuedWithdrawal.delegatedAddress) {
       ...
        // enforce minimum delay lag
        require(
            queuedWithdrawal.withdrawalStartBlock + withdrawalDelayBlocks <= block.number,   // @audit
            "StrategyManager.completeQueuedWithdrawal: withdrawalDelayBlocks period has not yet passed"
        );
```
```
* uint withdrawalDelayBlocks:
    As of M2, this is 50400 (roughly 1 week) // @audit
    Stakers must wait this amount of time before a withdrawal can be completed
```
During 1 week period staker’s funds, including beacon chain staked ETH, and LSTs (cbETH, rETH, stETH), aren’t usable on EigenLayer.

**Remediation:**  The staker should claim back his shares immediately without waiting for the `withdrawalDelayBlocks`. Delay `withdrawalDelayBlocks` should be applied only if the staker withdraws their tokens (ETH or LSTs) back.

**Status:**   Acknowledged

- - -

### [EIG-19] The EigenPod balance update function has to be permissioned

**Severity:** Medium

**Path:** EigenPod.sol:verifyBalanceUpdate#L193-L274

**Description:**  

The `verifyBalanceUpdate()` function is permissionless and, therefore, could be called by anyone with valid proof of a validator's current balance on the beacon chain. Resulting in the adjustment of a pod owner's shares.

Whilst withdrawals are processed by `verifyAndProcessWithdrawals()` against historical summaries https://github.com/Layr-Labs/eigenlayer-contracts/blob/master/docs/core/proofs/BeaconChainProofs.md#beaconchainproofsverifywithdrawal. They could be generated with a time lag of `8192 slots` or ~ `27 hours`.

Consider the following scenario when a Pod owner withdrew a fraction of their validators. In an unlucky case,  withdrawal slots of validators are close, and are at the beginning of the `8192 slot` window. So the pod owner have to wait one day before they can call the `verifyAndProcessWithdrawals()`.

In the meantime, someone with malicious intent could call `verifyBalanceUpdate` before the pod owner processes the withdrawal. And so the balances of those validators will be set to `0`, and the pod owner's shares will be massively decreased.

```
    function verifyBalanceUpdate(
        uint64 oracleTimestamp,
        uint40 validatorIndex,
        BeaconChainProofs.StateRootProof calldata stateRootProof,
        BeaconChainProofs.BalanceUpdateProof calldata balanceUpdateProof,
        bytes32[] calldata validatorFields
    ) external onlyWhenNotPaused(PAUSED_EIGENPODS_VERIFY_BALANCE_UPDATE) {  // @audit permissionless
    ...
    }
```

**Remediation:**  The `verifyBalanceUpdate()` function should be permissioned.  

**Status:**   Fixed

- - -

## Low Risk

### [EIG-1] Custom errors

**Severity:** Low

**Description:**  

In each contract the validation checks are performed using the `require` function with a reason string.

```  modifier onlyEigenPodManager() {
      require(msg.sender == address(eigenPodManager), "EigenPod.onlyEigenPodManager: not eigenPodManager");
      _;
  }
```

**Remediation:**  We would recommend to replace these with custom errors. This should be done by flipping the check.

For example:

```
require(X == Y, "X is not Y");
```
becomes

```
error XnotY(uint, uint);

if (X != Y)
  revert XnotY(X, Y);
```
The usage of custom errors will save a lot of gas during deployment as well as save on code bytesize of the contract. Furthermore, custom errors are much clearer as they allow for parameter values, making debugging much easier.

**Status:**   Acknowledged 


- - -

### [EIG-13] Creation of pods can suffer denial of service

**Severity:** Low

**Path:** EigenPodManager.sol:_deployPod#L378-L396

**Description:**  

The function to deploy a pod uses a counter `numPods` to check the total amount of pods against the `maxPods` configuration variable.

An attacker can create as many pods as needed in order to reach `maxPods` and prevent anyone else to create more pods or stake.

`maxPods` can be increased, however, the attacker can always quickly increase `numPods` with the creation of more pods for as long as maxPods low enough. Setting `maxPods` to a very high value would also negate the reason for its existence in such context.

```
    function _deployPod() internal onlyWhenNotPaused(PAUSED_NEW_EIGENPODS) returns (IEigenPod) {
        // check that the limit of EigenPods has not been hit, and increment the EigenPod count
        require(numPods + 1 <= maxPods, "EigenPodManager._deployPod: pod limit reached");
        ++numPods;
        // create the pod
        IEigenPod pod = IEigenPod(
            Create2.deploy( 
                0,
                bytes32(uint256(uint160(msg.sender))),
                // set the beacon address to the eigenPodBeacon and initialize it
                abi.encodePacked(beaconProxyBytecode, abi.encode(eigenPodBeacon, ""))
            )
        );
        pod.initialize(msg.sender);
        // store the pod in the mapping
        ownerToPod[msg.sender] = pod;
        emit PodDeployed(address(pod), msg.sender);
        return pod;
    }
```


**Remediation:**  We recommend to remove the cap on the number of pods as it does not seem to have any actual purpose. 

**Status:**   Acknowledged


- - -