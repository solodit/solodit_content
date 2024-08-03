**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### The `ModuleCanceled` event emits the wrong value for the timer parameter.


**Severity**: Low

**Status**: Resolved

**Description**

`ModuleCanceled` event emits the timer value of the module to be canceled for the third
parameter.
`densityModule_timer[_densityModule][_moduleId]` is emitted in the event but it is set to 0
before emitting.
So the `ModuleCanceled` event will always emit 0 for the timer parameter.

**Recommendation**:

Add a local variable for `densityModule_timer[_densityModule][_moduleId]` and emit the local
variable instead of `densityModule_timer[_densityModule][_moduleId]`.

## Informational

### could be set immutable.

**Severity**: Informational

**Status**: Resolved

**Description**

`densityModule0`, `densityModule1`, `densityModule2`, `densityModule3`, `densityModule4`
variables are only set in the constructor.

**Recommendation**:

Set the variables immutable.

### Unnecessary if condition for `densityModule_timer[_densityModule][_module]`.

**Severity**: Informational

**Status**: Resolved

**Description**

https://github.com/SentienceQuest/heirloom/blob/main/src/Heirloom.sol#L55
In the `moduleSupportedRequirements()` modifier, if `ownerOfWill` is not zero, there is no need
to check if `densityModule_timer[_densityModule][_module]` is zero because
`densityModule_owner` and `densityModule_timer` variables are set to non-zero or zero at the
same time.

**Recommendation**:

Remove if condition for `densityModule_timer`.

### The wrong spell in the `ModuleBenefiaryReplaced` event name.

**Severity**: Informational

**Status**: Resolved

**Description**

https://github.com/SentienceQuest/heirloom/blob/main/src/Heirloom.sol#L25
`ModuleBenefiaryReplaced` should be replaced with `ModuleBeneficiaryReplaced`.

**Recommendation**:

Change the event name to `ModuleBeneficiaryReplaced`.
