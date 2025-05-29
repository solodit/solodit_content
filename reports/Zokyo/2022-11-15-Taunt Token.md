**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Custom unprotected initializer in upgradeable token.
**Description**

Being upgrabeable, the token utilizes the init() custom initializer and not the recommended, more common initialize()) with no restrictions for admins-only calls. This fact and the absence of deployment scripts makes it impossible to verify if the contract's proxy will be deployed and initialized properly. Thus, it is open to a set of possible exploits.

**Recommendation**

Provide deployment scripts/procedures so that the team of auditors can verify the correctness of Proxy deployment and contract initialization.

**Re-audit comment**

Verified.

Post-audit:

The Taunt team has provided the deployment script with the correct way of utilizing the custom initializer.

## Low Risk

### Incorrect Solidity pragma version.
**Description**

Currently, the contract uses Solidity ^0.8.1, which violates the standard security checklist since it is set as a float version, not fixed, and it is not the latest stable version (which is 0.8.17). Using any other Solidity version except for the latest stable one may lead to unexpected behavior since the latest version contains new features, updates to the optimizer, and security fixes. Also, since there is no deployment scripts provided, it is impossible to verify which version will be used during the deployment. The issue is marked as low since it appears in thestandard security checklist and best practices.

**Recommendation**

Use the latest stable Solidity version (0.8.17)

**Re-audit comment**

Fixed

## Informational

### Use of 3rd-party contract for bot protection.
**Description**

TauntToken.sol: function_transfer()

The function uses a 3rd party contract. It is named "bot protection", though, since it is out of the scope and its code is absent, it cannot be validated, thus it leaves place for unexpected behavior and backdoors. Also, since the address is not set during the initialization and is set later by the admin, it creates a dangerous backdoor for the system. Besides, there is no validation against zero address, which allows setting parameters for the transfer to fail every time. The issue is marked as informational as it requires verification from the Taunt team as for the 3rd party contract. Yet, if no verification is provided the code of the 3rd-party contract), the issue will influence the rating since unknown logic may completely block the transfer.

**Recommendation**

1) Provide the address of the deployed contract or the contract code in order to verify the 3rd-party contract.
2) Add validation to the_transfer() function for the case when the BP contract is not set (it equals address).
3) The better way to ensure the security of the contract is to have the 3rd party set in the constructor. Thus, provide BP variable initialization in the constructor. If it is impossible (since contract may be created later), provide code of that contract.

**Re-audit comment**

Verified.

From the client:

The Taunt team has verified that BP is the bot protection contract. Its code is not verified to make it as hard as possible for attackers to subvert the protection and attack the listing. Since unverified production code is a sensitive matter, the Taunt team took precautions:
1. The BP code can only be controlled by the internal team that is in charge of every deployment.
2. The client that uses the BP code can always turn it on and off.
3. The Taunt team always instructs its clients to include a global flag that disables the BP permanently so that the code cannot be executed shortly after a successful launch. Even if the team does not disable the BP, the Taunt team code automatically disables itself after a couple of days. Therefore, investors only need to make sure that the feature was disabled once by the project managers after the launch.

### Upgradeable token implementation.
**Description**

TauntToken.sol

The contract is upgradeable. Usually, such implementation has no impact on security (though it creates a controllable backdoor). But it is highly recommended to avoid using upgradeable contracts for the token since it influences the trust to the token from exchanges and other 3rd parties that may use the token. The issue is marked as informational since it is mostly connected to the implementation, but it needs to be mentioned in the report.

**Recommendation**

Use non-upgradeable version of the contract OR verify that the Taunt team will stay with the upgradeable solution.

**Re-audit comment**

Verified.

Post-audit:

The Taun team has verified the upgradeability of the token. Nevertheless, the team of auditors needs to mention that it creates a controllable backdoor in the token logic, which may be crucial for the token usage in exchanges, listings, and DeFi protocols.

### Potential use of _beforTokenTransfer hook.
**Description**

TauntToken.sol

In order to decrease the code size and use recommended best practices, it is advised to use the_beforeToken Transfer() function for additional checks (e.g., for the bot protection) instead of overloading the whole_transfer() method. The issue is marked as informational as it refers to the best practices and does not influence the security of the project.

**Recommendation**

Consider using the_beforToken Transfer() method instead of_transfer() overloading.

**Re-audit comment**

Unresolved
