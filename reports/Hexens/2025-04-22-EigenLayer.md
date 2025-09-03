**Auditors**

[Hexens](https://hexens.io/)

---

# Findings
## Medium Risk

### [EIGEN2-14] Stake deviations due to adding/removing strategy may allow for ejector manipulation

**Severity:** Medium

**Path:** src/StakeRegistry.sol#L128-L174
src/StakeRegistry.sol#L727-L731
src/EjectionManager.sol#L135-L163

**Description:** In the StakeRegistry contract, the `updateOperatorsStake()` function is triggered by the `updateOperators()` function of the RegistryCoordinator contract to update the stakes of some specific operators. It will calculate the new stake of each operator based on the current strategies of a quorum and update the new total stake of the StakeRegistry.
```
(uint96[] memory stakeWeights, bool[] memory hasMinimumStakes) =
    _weightOfOperatorsForQuorum(quorumNumber, operators);

int256 totalStakeDelta = 0;
// If the operator no longer meets the minimum stake, set their stake to zero and mark them for removal
/// also handle setting the operator's stake to 0 and remove them from the quorum
for (uint256 i = 0; i < operators.length; i++) {
    if (!hasMinimumStakes[i]) {
        stakeWeights[i] = 0;
        shouldBeDeregistered[i] = true;
    }

    // Update the operator's stake and retrieve the delta
    // If we're deregistering them, their weight is set to 0
    int256 stakeDelta = _recordOperatorStakeUpdate({
        operatorId: operatorIds[i],
        quorumNumber: quorumNumber,
        newStake: stakeWeights[i]
    });

    totalStakeDelta += stakeDelta;
}

// Apply the delta to the quorum's total stake
_recordTotalStakeUpdate(quorumNumber, totalStakeDelta);
```
However, when adding or removing a strategy from a quorum, not all operators of that quorum will have their stakes updated immediately. Some operators may be updated first, leading to unfair competition when large stake deviations occur. The ejector role can take advantage of this to manipulate ejections as they wish.

For example, right after a strategy is added to a quorum, an ejector might post-run to update specific operators, increasing their stakes and the total stake of that quorum. After that, the `amountEjectableForQuorum` of that quorum will increase, allowing the ejector to eject other operators who still have outdated stakes. The operator selection for that ejection depends on the ejector.

**Remediation:**  After adding or removing a strategy in a quorum, it should be ensured that all operators of that quorum update their stakes right away, or a validation should be created to check that all operators in the quorum have been updated before executing an ejection.

**Status:**  Acknowledged


- - -

### [EIGEN2-5] Max operator limit can be exceeded

**Severity:** Medium

**Path:** SlashingRegistryCoordinator.sol:_kickOperator (L395-409)

**Description:** The function `_kickOperator` is used to forcefully remove an operator from a set of quorum numbers. It is called from `ejectOperator`, `_updateOperatorsStakes` and `_registerOperatorWithChurn`. The latter has an invalid assumption on the function in that it does not guarantee the removal of an operator and the decrease of the total operator count.

In `_registerOperatorWithChurn`, it will only execute the churn if the max operator count was reached using the provided kick parameters on lines 520-533:
```
if (results.numOperatorsPerQuorum[i] > operatorSetParams.maxOperatorCount) {
    _validateChurn({
        quorumNumber: uint8(quorumNumbers[i]),
        totalQuorumStake: results.totalStakes[i],
        newOperator: operator,
        newOperatorStake: results.operatorStakes[i],
        kickParams: operatorKickParams[i],
        setParams: operatorSetParams
    });

    bytes memory singleQuorumNumber = new bytes(1);
    singleQuorumNumber[0] = quorumNumbers[i];
    _kickOperator(operatorKickParams[i].operator, singleQuorumNumber);
}
```
However, in the function `_kickOperator` on lines 406-408 you can see that it does not remove the operator if the quorum numbers to be removed are not a subset of the operator’s quorum bitmap, in other words if it is not registered fully to those quorums:
```
if (quorumsToRemove.isSubsetOf(currentBitmap)) {
    _forceDeregisterOperator(operator, quorumNumbers);
}
```
This edge case can be triggered if the operator to be churned has left at least one of the to be removed quorums and a new operator has taken its place using a normal registration, bringing the total operator count again to the maximum and brining code execution into `_kickOperator`, while skipping the removal.

As a result, the total operator count can be increased for every signed churn parameters, as each one can be used to increase the count by one using the edge case. It can be abused by the caller if they own both the churn operator and the to be churned operator.
```
function _kickOperator(address operator, bytes memory quorumNumbers) internal virtual {
    OperatorInfo storage operatorInfo = _operatorInfo[operator];
    // Only proceed if operator is currently registered
    require(operatorInfo.status == OperatorStatus.REGISTERED, OperatorNotRegistered());

    bytes32 operatorId = operatorInfo.operatorId;
    uint192 quorumsToRemove =
        uint192(BitmapUtils.orderedBytesArrayToBitmap(quorumNumbers, quorumCount));
    uint192 currentBitmap = _currentOperatorBitmap(operatorId);

    // Check if operator is registered for all quorums we're trying to remove them from
    if (quorumsToRemove.isSubsetOf(currentBitmap)) {
        _forceDeregisterOperator(operator, quorumNumbers);
    }
}
```


**Remediation:**  While the soft-failure inside of `_kickOperator` is not necessarily a bug by itself, the misassumption of `_registerOperatorWithChurn` on this function is the problem. The function `_registerOperatorWithChurn` should validate the invariant that the total operator count was not exceeded at the end of the function regardless of what happens in `_kickOperator`.

**Status:**  Fixed

- - -

### [EIGEN2-4] Missing constraint check for modification of lookahead time of slashable quorum

**Severity:** Medium

**Path:** StakeRegistry.sol:_setLookAheadPeriod#L794-L802

**Description:** The StakeRegistry exposes the external function `setLookAheadPeriod`, which calls the internal variant and allows the CoordinatorOwner to set the lookahead time for a slashable quorum directly.

However, this modifying function is missing the constraint check that exists in `SlashingRegistryCoordinator.sol:_createQuorum`:
```
require(
    AllocationManager(address(allocationManager)).DEALLOCATION_DELAY() > lookAheadPeriod,
    LookAheadPeriodTooLong()
);
```
As a result, the CoordinatorOwner can set a look ahead period time that is greater than the deallocation delay of the AllocationManager, which would allow for voting power to be used that cannot be slashed, breaking the governance invariant. 
```
    function _setLookAheadPeriod(uint8 quorumNumber, uint32 _lookAheadBlocks) internal {
        require(
            stakeTypePerQuorum[quorumNumber] == IStakeRegistryTypes.StakeType.TOTAL_SLASHABLE,
            QuorumNotSlashable()
        );
        uint32 oldLookAheadDays = slashableStakeLookAheadPerQuorum[quorumNumber];
        slashableStakeLookAheadPerQuorum[quorumNumber] = _lookAheadBlocks;
        emit LookAheadPeriodChanged(oldLookAheadDays, _lookAheadBlocks);
    }
```

**Remediation:**  The function `_setLookAheadPeriod` should include the same constraint check as when creating the quorum.

**Status:**  Fixed


- - -

### [EIGEN2-2] Current rate limit logic allows ejection beyond allowed limit

**Severity:** Medium

**Path:** EjectionManager.sol:ejectOperators

**Description:** The owner or an ejector can eject operators. Only the ejector has a rate limit, which restricts how much can be totally rejected during a quorum, that is based on a limit set by the owner.

When an ejector calls `ejectOperators` to eject an operator, it checks the `amountEjectable` for the  quorum. This is used as the total amount ejectors are allowed to remove.

The issue occurs in the following condition:
```
function ejectOperators(
    bytes32[][] memory operatorIds
) external {  

-- SNIP -- 

  if (
      isEjector[msg.sender]
          && quorumEjectionParams[quorumNumber].rateLimitWindow > 0
          && stakeForEjection + operatorStake > amountEjectable
  ) {

      ratelimitHit = true;

      stakeForEjection += operatorStake;
      ++ejectedOperators;

      slashingRegistryCoordinator.ejectOperator(
          slashingRegistryCoordinator.getOperatorFromId(operatorIds[i][j]),
          abi.encodePacked(quorumNumber)
      );

      emit OperatorEjected(operatorIds[i][j], quorumNumber);

      break;
  }
```
If the rate limit is hit, it will still eject the operator and only then break the loop.

Consider the following scenario:

- `amountEjectable` = 1e16 (the allowed amount for this quorum to be removed by the ejector) 

- `OperatorXAmount` =  10e18.
```
stakeForEjection + operatorStake > amountEjectable
0 + 10e18 > 1e16
```
It will eject the operator that had a value of `10e18`, even though the ejector is only allowed to eject up to `1e16`.

**Remediation:**  Considering removing the ejecting logic in the case where the rate limit would have been hit, so it will only make `ratelimitHit` true and break the loop.
```
function ejectOperators(
    bytes32[][] memory operatorIds
) external {  

-- SNIP -- 

  if (
      isEjector[msg.sender]
          && quorumEjectionParams[quorumNumber].rateLimitWindow > 0
          && stakeForEjection + operatorStake > amountEjectable
  ) {

      ratelimitHit = true;

--    stakeForEjection += operatorStake;
--    ++ejectedOperators;

--    slashingRegistryCoordinator.ejectOperator(
--         slashingRegistryCoordinator.getOperatorFromId(operatorIds[i][j]),
--         abi.encodePacked(quorumNumber)
--     );

--     emit OperatorEjected(operatorIds[i][j], quorumNumber);

      break;
  }
```

**Status:**   Fixed


- - -

## Low Risk

### [EIGEN2-15] Ejection timestamp is not updated when operator is kicked out in update loop

**Severity:** Low

**Description:** An operator could be kicked by ejection, which invokes the kick functions in `SlashingRegistryCoordinator.sol` It sets a `lastEjectionTimestamp` to the current block.timestamp, which disallows the same operator to register immediately again.
```
function ejectOperator(
    address operator,
    bytes memory quorumNumbers
) public virtual onlyEjector {
    lastEjectionTimestamp[operator] = block.timestamp;
    _kickOperator(operator, quorumNumbers);
}
```
However `_kickOperator` is also called when a operator gets kicked by someone registering with churn and during `updateOperators` that kicks operators that don't meet the minimum threshold.
```
function _updateOperatorsStakes(... ) internal virtual {
      -- snip
      if (doesNotMeetStakeThreshold[j]) {
          _kickOperator(operators[j], singleQuorumNumber);
      }
  }
}
```
It doesn’t update the `lastEjectionTimestamp`, although they are kicked out by not having enough threshold. 
```
function ejectOperator(
    address operator,
    bytes memory quorumNumbers
) public virtual onlyEjector {
    lastEjectionTimestamp[operator] = block.timestamp;
    _kickOperator(operator, quorumNumbers);
}
```


**Remediation:**  Consider updating the operator’s `lastEjectionTimestamp` when they are kicked out for not having a minimum threshold, so they can’t register immediately again.

**Status:** Acknowledged


- - -

### [EIGEN2-10] A new operator could replace another operator with less staking amount

**Severity:** Low

**Path:** SlashingRegistryCoordinator.sol:_setOperatorSetParams

**Description:** The function `_setOperatorSetParams` sets the configuration for a quorumNumber for the `SlashingRegistryCoordinator`. 

Currently, there are no checks for the `kickBIPsOfOperatorStake` and  `kickBIPsOfTotalStake`. However it would be better to implement some limitations on these values.

For example. in the function `_validateChurn`, these values are used to determine if the registrering user’s stake is greater than the `operatorToKickStake` times `kickBIPsOfOperatorStake` its value:
```
_validateChurn
    require(
          newOperatorStake > _individualKickThreshold(operatorToKickStake, setParams),
          InsufficientStakeForChurn()
      );
```
```
function _individualKickThreshold(
    uint96 operatorStake,
    OperatorSetParam memory setParams
) internal pure returns (uint96) {
    return operatorStake * setParams.kickBIPsOfOperatorStake / BIPS_DENOMINATOR;
}
```
To prevent a case where a lower stake could kick a higher stake, the value should always be higher than 100% expressed in BIPS (10_000). It would be an unfair situation to kick out another operator while the newOperatorStake is lower than the operatorToKickStake.
```
    function _setOperatorSetParams(
        uint8 quorumNumber,
        OperatorSetParam memory operatorSetParams
    ) internal {
        _quorumParams[quorumNumber] = operatorSetParams;
        emit OperatorSetParamsUpdated(quorumNumber, operatorSetParams);
    }
```


**Remediation:**  We recommend to add a check to ensure `kickBIPsOfOperatorStake` is higher than 100% (`10_000`).

**Status:** Acknowledged


- - -

### [EIGEN2-8] Redundant double check of max operator count in register operator

**Severity:** Low

**Path:** SlashingRegistryCoordinator.sol:registerOperator#L151-L225

**Description:** The `registerOperator` function is called from the AllocationManager. The operator can specify a few parameters, such as the registration type. 

In the case of the `NORMAL` registration type, the function simply calls the internal `_registerOperator` with `checkMaxOperatorCount: true`. This will trigger a check at the end of `_registerOperator` where the new total operator count is checked against the quorum’s `maxOperatorCount`.

However, after calling `_registerOperator`, the function will again perform the same check by looping over the quorum numbers and checking the result against `maxOperatorCount`:
```
for (uint256 i = 0; i < quorumNumbers.length; i++) {
    uint8 quorumNumber = uint8(quorumNumbers[i]);

    require(
        numOperatorsPerQuorum[i] <= _quorumParams[quorumNumber].maxOperatorCount,
        MaxOperatorCountReached()
    );
}
```
This second check is exactly the same and performed right afterwards, making it redundant and a waste of gas.

**Remediation:**  Remove the redundant check in `registerOperator`, as the parameter `checkMaxOperatorCount: true` already covers this.

**Status:**  Fixed

- - -

