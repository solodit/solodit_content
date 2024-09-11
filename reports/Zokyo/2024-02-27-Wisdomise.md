
**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk


### Strict equality in `migrate()` method

**Severity** : Medium

**Status** : Resolved

**Description** :

In Contract TokenMigration.sol, the method migrate() allows users to migrate their TWSDM tokens to WSDM tokens. 

This method contains the following logic:
```solidity
       uint256 angelInvestorShares = angelInvestors[msg.sender];
       uint256 strategicInvestorShares = strategicInvestors[msg.sender];
       require(
           balance == angelInvestorShares + strategicInvestorShares,            "TokenMigration: twsdm balance does not match with investor shares"
       );
```

If the balance is not equal to the investor shares, it will fail to migrate the tokens. Anyone can front-run a user’s migrate() txn to transfer 1 wei of TWSDM to the user account. This will make the txn revert because of the above `require` statement. 

This will lead to DoS attacks for users especially users with huge amounts of TWSDM tokens and deny them migrating their tokens. Given that these contracts are deployed on Polygon, the cost of the attack will be less as well.

**Recommendation**: 

Update the `migrate()` logic to handle the case when the balance is more than the investor shares as well. Also, revert in case investor shares are 0 for msg.sender.

### Investor can be front running the procedure in `_addInvestor`

**Severity**: Medium

**Status**: Invalid

**Location**: TokenMigration.sol

**Description**:

This can take place in case owner intentionally attempts to overwrite a previous assignment for an investor. Investor can capitalize on that and seize both amounts.
That is described in the following scenario:
**Scenario**:
txn.a1:  _addInvestor(0xB0B,  100, ANGEL)
txn.a2:  _addInvestor(0xB0B,  50, ANGEL)    // Admin realized a mistake and intended to overwrite the quantity
txn.b1:  migrate()   // called by 0xB0B
txn.b2:  migrate()   // called by 0xB0B

0xB0B tunes the gas payment in order to squeeze txn.b1 in between txn.a1 and txn.a2. Therefore 0xB0B receives 100 then from txn.b2 he receives 50 which is the fair amount intended to be assigned by admin but 0xB0B ends up capitalizing on that scenario to take both amounts.
This issue should be addressed along side: Records can be overridden without checking & Strict equality in migrate() method. Refactoring of the migration logic is recommended.
**Recommendation** 

This issue is similar to the ERC20 approval frontrunning which has been solved by introducing increaseAllowance and decreaseAllowance in later versions of ERC20, therefore a similar mechanism helps avoid this. It is recommended though to address that issue alongside the 2 other issues and refactor the logic accordingly.
**Fix** - The manual setting of shares by the admin is only meant to allocate strategic and angel shares. The balance of twsdm remains the single source of truth. Function migrate() burns all the balance of the caller and it can only be called once.

### Emergency Exit Logic Undermines Pausable Mechanism

**Severity** : Medium

**Status** : Resolved

**Description** :

The contract Locking implements an emergency exit mechanism alongside the Pausable functionality inherited from OpenZeppelin's Pausable contract. However, the logic used to determine whether actions can proceed during an emergency or paused state could potentially compromise the security and intended functionality of the contract.



This line is part of the whenNotPausedOrEmergencyExitActive modifier, which is intended to restrict certain contract functionalities when the contract is paused, except in cases of an emergency exit. The issue arises from the logical OR (||) operator, which allows actions to proceed if either the contract is not paused (!paused()) or if the emergency exit is active (emergencyExit). This means that during an emergency (when emergencyExit is set to true), operations can still be executed even if the contract is paused.
This logic undermines the purpose of the emergency stop mechanism, which is to halt operations in the face of detected vulnerabilities or ongoing attacks. Allowing operations to proceed during an emergency, even when the contract is paused, could open up avenues for exploitation, especially if the emergency exit mechanism is not tightly controlled or if the pause state was specifically activated to prevent certain actions from being taken.

**Recommendation** :

Create separate functions for actions that should only be available during an emergency. This clear separation ensures that only intended actions are executable in an emergency scenario, reducing the risk of exploitation.
Redefine the whenNotPausedOrEmergencyExitActive modifier to ensure it strictly enforces the intended restrictions. The modifier should be designed to prevent any operations from proceeding when the contract is paused, with the only exceptions being the newly defined emergency functions.
For the emergency functions, implement strict access controls to ensure that only authorized entities can activate or execute these functions.

```solidity
contract Locking is Pausable {
    bool private emergencyExit = false;

    modifier whenNotPausedOrEmergencyExitActive() {
        require(!paused() || emergencyExit, "Contract is not paused and not in emergency");
        _;
    }

    function activateEmergencyExit() public onlyOwner {
        emergencyExit = true;
    }

    function emergencyFunction() public whenNotPausedOrEmergencyExitActive {
        // Emergency function logic here
    }

    function normalFunction() public whenNotPaused {
        // Normal function logic here
    }
}
```
### Use of `SafePermit` is removed Based on recommendations from @trust1995

**Severity** : Medium

**Status** : Resolved

**Description** :

In the latest openzeppelin-contracts safePermit has been removed

https://github.com/OpenZeppelin/openzeppelin-contracts/pull/4582 

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/192e873fcb5b1f6b4b9efc177be231926e2280d2/CHANGELOG.md

**Recommendation** : 

Create custom permit operation (considering all the drawbacks of traditional permit operation) like 
 There are two important considerations concerning the use of `permit`. The first is that a valid permit signature expresses an allowance, and it should not be assumed to convey additional meaning. In particular, it should not be considered as an intention to spend the approval in any specific way. The second is that because permits have built-in replay protection and can be submitted by anyone, they can be frontrun. A protocol that uses permits should take this into consideration and allow a `permit` call to fail. Combining these two aspects, a good pattern that can be generally recommended is:
```solidity
try token.permit(msg.sender, spender, value, deadline, v, r, s) {} catch {}
```
Observe that: 1) `msg.sender` is used as the owner, leaving no ambiguity as to the signer intent, and 2) the use of `try/catch` allows the permit to fail and makes the code tolerant to frontrunning. In general, `address(this)` would be used as the `spender` parameter. Additionally, note that smart contract wallets (such as Argent or Safe) are not able to produce permit signatures, so contracts should have entry points that don't rely on permit.




### Use Ownable instead of an immutable beneficiary in vesting.sol

**Severity** : Medium

**Status** : Resolved

**Description** :


In the current implementation of the VestingWallet, the beneficiary is set at contract deployment and cannot be altered. This design choice limits the ability to transfer unvested tokens, potentially creating issues if the original beneficiary address becomes compromised or if there's a legitimate need to transfer the vesting schedule to a new address.

Use `Ownable` in `VestingWallet` instead of an immutable beneficiary by ernestognw · Pull Request #4508 · Open Zeppelin/open zeppelin-contracts (github.com)

**Recommendation** : 

Inherit from Ownable2Step or Ownable
Restrict the release() Function

## Low Risk

### Precision Loss due to truncated decimals

**Severity** : Low

**Status** : Resolved

**Description** :

This line will cause precision loss, however it mainly depends on the constructor arguments. This bug is due to the lack of support for floating-point arithmetic, which means Solidity can only handle integer values.
```solidity
uint256 tge_round_release = (totalAllocation * tgeReleasePercentage) / 100;
```



**Recommendation**: 

Consider scaling up the values involved in the calculation before performing arithmetic operations. For example, using a higher base for percentages (e.g., representing 1% as 10000 in calculations instead of 1)

### Penalty fee can be higher than the user balance

**Severity** : Low

**Status** : Resolved

**Description** :

In Contract locking.sol, the method unlock allows users to unlock their locked tokens but a penalty fee is incurred on them.

The amount fee is calculated as follows:
```solidity
   function calculatePenalty(address locker, uint256 balance, uint256 unlockTimestamp) public view returns (uint256) {
       if (emergencyExit) {
           return 0;
       }
       return (balance * calculatePenaltyFee(locker, unlockTimestamp)) / 10 ** 6;
   }
```


Here if the method calculatePenaltyFee returns a value > 10^6, then the penalty fee will be more than the balance itself. This will revert the user's unlock txn.

It is not validated anywhere that calculatePenaltyFee() will be less than 10^6 always.

**Recommendation**: 

Add a check to ensure that the max penalty fee is equal to the balance of the user locked.

### No address(0) validation for constructor parameters

**Severity** : Low

**Status** : Resolved

**Description** :

In Contract wsdm.sol, the constructor mints the total supply to the reserve address. It is not validated if that address is address(0) or not. Also no other method exists to set reserve again so if reserve is set address(0) then contract will need to be deployed again.

In Contract TokenDistributor.sol, the constructor does not validate if the wsdmTokenAddress is address(0) or not. Also no other method exists to set wsdmTokenAddress.

In Contract TokenMigration.sol, none of the constructor parameters is validated for address(0). Also no other method exists to set these constructor parameters.

**Recommendation**: 

Add the required validation for the above-mentioned contracts.


### Records can be overridden without checking

**Severity**: Low

**Status**: Resolved

**Location**: TokenMigration.sol

**Description** :

In function `_addInvestor()`, the values inside mappings angelInvestors and strategicInvestors can be overridden. Suppose a scenario in which the owner (i.e. procedure can only be triggered by owner) executes the function on same investor with a different balance value, the values are overridden.
```solidity
    function _addInvestor(address investor, uint256 balance, InvestorType investorType) private {
        require(investor != address(0), "TokenMigration: investor should not be address zero");

        if (investorType == InvestorType.ANGEL) {
            angelInvestors[investor] = balance;
        } else if (investorType == InvestorType.STRATEGIC) {
            strategicInvestors[investor] = balance;
        }

        emit InvestorInserted(investor, balance, investorType);
    }
```
This could be a feature because of the strict equality check imposed in function migrate:
```solidity
require(
    balance == angelInvestorShares + strategicInvestorShares,
        "TokenMigration: twsdm balance does not match with investor shares"
);
```
It is worth noting that the substance of the issue comes from the choice to make the process manual to a big extent. Attention is needed in this part since it might introduce a non-negligble impact if human error arises.
Overridding those mappings introduce the contracts to an attack vector that is explained in a separate finding.
**Recommendation** 

Refactor the logic of migration putting in mind less human involvement in determining the values.

**Fix** -  The manual setting of shares by the admin is only meant to allocate strategic and angel shares. The balance of twsdm remains the single source of truth. Client is adopting certain methods in order to allocate the shares minimizing the chance of the need to call the function again and override the allocations.

### `hasUnlockedBefore` is not reset back to false on `cancelUnlock`

**Severity**: Low

**Status**: Resolved

**Location**: Locking.sol

**Description** :

When users invoke unlock() to start a process to unlock their assets. The state of unlockedUsers[msg.sender] and _lockedUsers[msg.sender] is changed accordingly. In case users change their mind by calling cancelUnlock() the changes are reset back to the state before calling unlock(). There is an issue specifically in the following:
_lockedUsers[msg.sender].hasUnlockedBefore = true;

Since on calling cancelUnlock() the state before the unlock() is expected to be retrieved. But _lockedUsers[msg.sender].hasUnlockedBefore is not becoming false as it should. This affects the calculation of calculatePenaltyFee() as in that case the value of hasUnlockedBefore would be true changing the course of how penalty is calculated.
**Recommendation** 

Assign the attribute back to false.

_lockedUsers[msg.sender].hasUnlockedBefore = false;

**Fix**: Considered as a required spec by the developer team.

### Centralization risk

**Severity**: Low

**Status**: Resolved

**Location**: locking.sol, TokenDistributor.sol, TokenMigration.sol

**Description** :

The smart contracts employ modifiers (e.g. onlyOwner) in functions responsible for carrying out important actions (e.g. setEmergencyExit, addBulkAngelInvestor, ...etc). This approach centralizes control, allowing a single owner to exclusively perform critical actions, that involves asset transfer, posing a risk to decentralization principles.
Risk: Single Point of Failure: Centralized control introduces a single point of failure, where the compromise of the owner's account can lead to unauthorized access and manipulation of critical functions.
**Recommendation** 

To mitigate the centralization risk, it is recommended to:
Implement Access Control Lists (ACL): Utilize Access Control Lists to assign different roles with specific permissions, allowing for a more granular control structure.
Multi-Sig or Governance Contracts: Consider implementing multi-signature schemes or governance contracts to distribute decision-making authority, reducing the risk of a single point of failure. Multi-Sig can be utilized in the project after deployment without altering the codebase.

### Lack of Event Emission in Privileged Functions

**Severity**: Low

**Status**: Resolved

**Location**: locking.sol, TokenDistributer.sol

**Description** :

During the security audit of the smart contract, an issue has been identified in the implementation of privileged functions, one example is function setNumberOfFreeTrialPeriod(). The codebase has not incorporated event emission within several of these privileged functions. The absence of events in these operations limits transparency, hindering the ability to monitor and track critical changes to the contract's state.
**Recommendation** - Emit the relevant events on calling privileged functions.

**Fix**:  Addressed and fixed in commit  37e02df.

## Informational

### Short-duration logic can be manipulated by miners.

**Severity** : Informational

**Status** : Resolved

**Description** :

block.timestamp is utilized in several crucial operations, notably in the calculatePenalty and calculatePenaltyFee functions, which determine the penalty amount based on the duration a user's tokens have been locked. Since block.timestamp is used to calculate unlocking periods and penalties, there's a risk that miners could manipulate timestamps to influence these calculations. Although the Ethereum protocol enforces that each block's timestamp must be greater than the previous block's timestamp and allows only a small future time drift (around 15 seconds), there still exists a window for manipulation, especially in operations sensitive to time adjustments.


**Recommendation** : 

- Use External Time Oracles
- Introduce Grace Periods or buffers
- Increase Time Intervals (for functions that are not time-sensitive to the second, use hours or days)

### Recommendation for Upgradeable ERC20 Token Implementation

**Severity**: Informational

**Status**: Acknowledged

**Location**: wsdm.sol

**Description** :

During the security audit, it is noted that client possess a proactive approach to maintaining and evolving their projects which led them to decide migrate ERC20 token. To further enhance the project's flexibility and upgradability, it is recommended to consider the implementation of an upgradeable version of the ERC20 token standard.
While the current ERC20 implementation is sound and secure, adopting an upgradeable model demonstrates a commitment to ongoing development, adaptability to future changes, and an improved user experience by minimizing migration requirements. The proactive consideration of upgradeability aligns with industry best practices and ensures the project remains resilient and responsive to evolving standards and user needs.
Rationale:
Flexibility for Future Enhancements: An upgradeable ERC20 token allows the development team to introduce improvements, bug fixes, or additional features without requiring users to migrate to a new contract. This flexibility is particularly valuable for long-term projects.
Reduced Migration Overheads: Implementing upgradeability reduces the need for token migrations, minimizing the operational and user experience overheads associated with deploying entirely new contracts.
Adaptability to Changing Requirements: The crypto landscape evolves, and new standards or security practices may emerge. An upgradeable ERC20 token positions the project to adapt more seamlessly to any future changes or advancements in the Ethereum ecosystem.
**Recommendation** 

Consider implementing an upgradeable ERC20 token using established patterns such as the OpenZeppelin Upgrades Plugins or other industry-recognized upgradeable contract frameworks.

### Gas Consumption Concerns Due to Extended Error Messages

**Severity**: Informational

**Status**: Acknowledged

**Location**: locking.sol, TokenDistributor.sol, TokenMigration.sol

**Description** :

The smart contract employs detailed error messages within require statements, offering developers and users specific insights into the conditions that trigger a requirement failure. However, the extensive length of these error messages result in heightened gas consumption.

**Recommendation**

Employ shorter concise error messages.
Use Custom Error objects available from solidity 0.8.0.

### Unnecessary Safe Math is being utilized

**Severity**: Informational

**Status**: Resolved

**Location**: TokenDistributor.sol

**Description** :

The default safe math operation in solidity versions ^0.8.x incurs extra gas overhead due to it requiring more computation. The following operation, that is being carried out on the iterator of the for-loop, can be more gas-efficient by using the unchecked statement.

In function addBulkPayeeByOwner(), we have:
```solidity
       for (uint256 i = 0; i < accounts.length; i++) {
            _addPayee(accounts[i], shares[i]);
        }
```
As well in TokenMigration.sol, in function _addBulkInvestor().
```solidity
   for (uint256 i = 0; i < investors.length; i++) {
            _addInvestor(investors[i], balances[i], investorType);
        }
    }
```
While the code snippet correctly ensures that the addition operation will not result in an overflow, the unnecessary default safe math usage can be optimized for gas efficiency. Wrapping the operation in an unchecked statement is a recommended practice for situations where the developer can guarantee that overflows/underflows will not occur. This enhancement contributes to more efficient gas utilization without compromising safety.
**Recommendation** 

Wrap Operation in unchecked Statement, given that the condition preceding the operation ensures there is no risk of overflow. It is a common pattern in for-loops since 0.8.x to be as follow:
```solidity
       for (uint256 i = 0; i < length;) {
            ...
            unchecked {
                i++;
            }
        }
```
**Fix**:  Addressed and fixed in commit  37e02df.

### Floating Pragma Version in Solidity Files

**Severity**: Informational

**Status**: Resolved

**Location**: All

**Description** :

The Solidity files in this codebase contain pragma statements specifying the compiler version in a floating manner (e.g., ^0.8.0). Floating pragmas can introduce unpredictability in contract behavior, as they allow automatic adoption of newer compiler versions that may introduce breaking changes.

**Recommendation**: 

To ensure stability and predictability in contract behavior, it is recommended to:
Specify Fixed Compiler Version: Instead of using a floating pragma, specify a fixed compiler version to ensure consistency across different deployments and prevent the automatic adoption of potentially incompatible compiler versions.

### Rename mapping

**Severity** : Informational

**Status** : Resolved

**Description** :

In Contract TokenMigration.sol, the mapping angelInvestors and strageticInvestors store the user's shares. It is advised to rename these mappings to angelInvestorShares and strageticInvestorShares for better readability.

**Recommendation**: 

Make the suggested changes.
