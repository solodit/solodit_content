
**Auditor**

[Alex The Entreprenerd](https://x.com/gallodasballo?lang=en)



# Findings

## Low Risk

### [L-01] Executive Summary

The module allows a registered chainlink upkeep to automatically re-lock aura locks

Funds cannot be stolen in any way

Due to the lax timing and lack of MEV, I cannot expect the upkeep to cause any particular issue

It's worth noting that in case you want to deprecate the module, for example to stop re-locking, it will be sufficient to remove it from the safe modules

There is no particular risk tied to adding this module as in the worst case it will consume a bit of `LINK` token to perform the upkeep

### [L-02] Operative Gotcha - Safe.getModules is limited to the first 10 modules

**Impact**

`_isModuleEnabled` is written as follows:

https://github.com/onchainification/aura_locker_v2/blob/07294ae3638909ecd768a6a0f831fa513abe91a0/src/AuraLockerModule.sol#L123

```solidity
    /// @dev The Gnosis Safe v1.1.1 does not yet have the `isModuleEnabled` method, so we need a workaround
    function _isModuleEnabled() internal view returns (bool) {
        address[] memory modules = SAFE.getModules();
        for (uint256 i = 0; i < modules.length; i++) {
            if (modules[i] == address(this)) return true;
        }
        return false;
    }
```

Which uses `getModules`, which is paginated and limited to the first 10 enabled modules

https://etherscan.io/address/0x34cfac646f301356faa8b21e94227e3583fe3f5f#code
```solidity
    function getModules()
        public
        view
        returns (address[] memory)
    {
        (address[] memory array,) = getModulesPaginated(SENTINEL_MODULES, 10);
        return array;
    }
```

It's worth noting that if this module where to be used with a lot of other modules setup, the check could fail

However, current there are no other modules set, meaning that the code is safe as is

Additionally, even if the check were to fail, no particular damage would be caused to the Safe, at worst the `checkUpkeep` would always return false, making no upkeep run, but causing no DOS to the Safe

**Mitigation**

Add a comment to the module and make sure to have less than 10 modules

---
## Gas Optimizations

### [G-01] Gas Optimization: Can skip check to save 200 gas

**Impact**

`performUpkeep` has a check to verify if there's any re-lockable balance

https://github.com/onchainification/aura_locker_v2/blob/07294ae3638909ecd768a6a0f831fa513abe91a0/src/AuraLockerModule.sol#L112-L113

```solidity
    /// @notice The actual execution of the action determined by the `checkUpkeep` method (AURA locking)
    function performUpkeep(bytes calldata /* _performData */ ) external override onlyKeeper {
        if (!_isModuleEnabled()) revert ModuleNotEnabled();

        (, uint256 unlockable,,) = AURA_LOCKER.lockedBalances(address(SAFE));
        if (unlockable == 0) revert NothingToLock(block.timestamp);
```

If the check fails, the call will revert.

However, vlAURA already has this check

https://etherscan.io/address/0x3Fa73f1E5d8A792C80F426fc8F84FBF7Ce9bBCAC#code
```solidity
    function _processExpiredLocks(
        address _account,
        bool _relock,
        address _rewardAddress,
        uint256 _checkDelay
    ) internal updateReward(_account) {
/// OMITTED

require(length > 0, "no locks"); /// @audit Reverts here
```

Meaning you can skip the call to save around 200 gas

**Mitigation**

Consider removing the check to save 200 gas

----
## Informational

### [I-01] Nitpick: `SAFE` and `BALANCER_MULTISIG` are the same value and are used inconsistently

**Impact**

The casting of `address(SAFE))` are unnecessary if you use `BALANCER_MULTISIG`

https://github.com/onchainification/aura_locker_v2/blob/07294ae3638909ecd768a6a0f831fa513abe91a0/src/AuraLockerModule.sol#L20-L21

```solidity
    address public constant BALANCER_MULTISIG = 0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f;
    IGnosisSafe public constant SAFE = IGnosisSafe(payable(BALANCER_MULTISIG));
```

Which is what's done in `onlyGovernance`

https://github.com/onchainification/aura_locker_v2/blob/07294ae3638909ecd768a6a0f831fa513abe91a0/src/AuraLockerModule.sol#L67C1-L71C6

```solidity
    /// @notice Enforce that the function is called by governance only
    modifier onlyGovernance() {
        if (msg.sender != BALANCER_MULTISIG) revert NotGovernance(msg.sender);
        _;
    }
```

**Mitigation**

You could alternatively change `onlyGovernance` to use `address(SAFE)` and make the code consistent

This has no impact on the bytecode so it's not a big deal

### [I-02] Address Checks

The addresses are correctly set

address public constant BALANCER_MULTISIG = 0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f
Confirmed, 6/11 multi 
https://etherscan.io/address/0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f#code

IERC20 public constant AURA = IERC20(0xC0c293ce456fF0ED870ADd98a0828Dd4d2903DBF); // OK
https://etherscan.io/address/0xC0c293ce456fF0ED870ADd98a0828Dd4d2903DBF#code

ILockAura public constant AURA_LOCKER = ILockAura(0x3Fa73f1E5d8A792C80F426fc8F84FBF7Ce9bBCAC); // OK
https://etherscan.io/address/0x3Fa73f1E5d8A792C80F426fc8F84FBF7Ce9bBCAC#code



