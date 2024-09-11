**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Non-Standard ERC20 tokens could be locked in the contract

**Severity**: High

**Status**: Resolved

**Description**

The ERC20 transfer function is used to transfer `rewardToken` tokens. However, some ERC20 tokens, such as USDT, BNB, and OMG, do not return a boolean. This can cause the `require(success, "Token transfer failed")` check to always fail, potentially resulting in tokens being locked in the contract.

**Recommendation**: 

To ensure compatibility with all ERC20 tokens and to handle transfer failures properly, it is recommended to use the SafeERC20 library from OpenZeppelin. This library provides a safe transfer function that properly handles tokens that do not return a value.

**Replace**:

```solidity
bool success = IERC20(rewardToken).transfer(owner(), amount);
```
With:
```solidity
SafeERC20.safeTransfer(IERC20(rewardToken), owner(), amount);
```

## Medium Risk

### Centralization Risk due to overpowered owner

**Severity**: Medium

**Status**: Acknowledged

**Description**


The CyberFinance contract introduces a significant centralization risk due to its reliance on the owner for critical functions, including pausing/unpausing the contract, increasing/decreasing claimable balances, and withdrawing tokens. This centralized control can lead to potential abuse or single points of failure, particularly if the owner account is compromised or if the owner acts maliciously.

**Recommendation**: 

To mitigate centralization risks, consider implementing multisig wallets or time-locked functions for critical operations.

**Comment**: 

The client stated that the project is planned to set up a multisig for the owner wallet to mitigate the centralization. 

## Low Risk

### PUSH0 Opcode is Incompatible with some Chains 

**Severity**: Low

**Status**: Acknowledged

**Description**

The `CyberFinance` contract is written using Solidity version 0.8.21, which introduces the push0 opcode. This opcode is not supported by all chains, especially those not compatible with the Shanghai hardfork. Deploying this contract on such incompatible chains could lead to deployment failures.

This means that the produced bytecode won't be compatible with the chains that don't yet support the Shanghai hard fork. This could also become a problem if different versions of Solidity are used to compile contracts for different chains. The differences in bytecode between versions can impact the deterministic nature of contract addresses.


**Recommendation**: 

To ensure broader compatibility and prevent deployment issues, it is recommended to either roll back the Solidity version or hardcode the EVM version to "paris" in the Foundry configuration file (foundry.toml).

### Owner Can Renounce Ownership While System is Paused

**Severity**: Low

**Status**: Acknowledged

**Description**

The `CyberFinance` contract inherits from `Ownable2Step` and includes Pausable functionality. However, the current implementation allows the owner to renounce ownership even while the system is paused. This can lead to a scenario where the contract is left in an unusable state and the funds could be locked there forever, as no one would have the authority to unpause the contract and resume normal operations.

**Recommendation**: 

Implement a check to prevent the owner from renouncing ownership while the contract is paused. This ensures that the system remains manageable and prevents it from being locked in a paused state indefinitely.

### Contract Balance Insufficiency Prevents User Withdrawals

**Severity**: Low

**Status**: Resolved

**Description**

The claim function currently restricts withdrawals if the contract's balance of the reward token is less than the user's claimable balance. This may lead to user frustration, as they are unable to withdraw any portion of their claimable assets under these conditions. A scenario where users are unable to claim their rewards due to contract balance constraints undermines user trust and satisfaction.

**Recommendation**: 

To address this issue, modify the claim function to allow users to withdraw whatever portion of their claimable balance is available in the contract. If the contract's balance is less than the user's claimable balance, the function should allow the user to withdraw the available balance and subsequently deduct the withdrawn amount from the user's claimable balance. This ensures that users can at least partially receive their assets and improves user experience by mitigating frustrations related to withdrawal failures.

## Informational

### Front Running Vulnerability in `decreaseClaimable` Function

**Severity**: Informational

**Status**: Acknowledged

The `decreaseClaimable` function, which can only be called by an admin, is vulnerable to front running by users who call the claim function. This allows the users to drain the claimable balance before it is decreased by the `decreaseClaimable` function. This behavior can potentially lead to unexpected fund depletion or malicious draining of resources.

**Recommendation**: 

To mitigate this vulnerability, consider implementing a two-step claim process. Introduce a `commitClaim` function where the user commits to a planned claim as the first step. The user would then finalize the claim after a certain number of blocks have passed and before a specified deadline block. This mechanism would significantly hinder opportunistic claims based on admin actions, thereby reducing the risk of front-running exploits.

### Redundant Modifier Invocation in Withdraw Functions

**Severity**: Informational

**Status**: Resolved

**Description**

The `onlyOwner` modifier is invoked twice in `withdrawBalance` and `withdraw` functions, leading to inefficiency in the smart contract. Both functions perform a similar check that could be consolidated.

**Recommendation**: 

To improve the efficiency, refactor the withdraw logic into an internal function. Both `withdraw` and `withdrawBalance` can then call this internal function, ensuring that the modifiers are applied only once. This reduces redundancy and improves code readability and efficiency.

### Public Functions Could Be Declared External

**Severity**: Informational

**Status**: Resolved

**Description**

In Solidity, functions can be declared as public or external. While both visibility specifiers allow functions to be called from outside the contract, external functions are generally more gas-efficient when called externally. This is because external functions use a lower amount of gas due to optimized calldata handling.
In the CyberFinance contract, several functions that are intended to be called only from outside the contract are declared as public. These functions could be declared as external to optimize gas usage.

**Recommendation**: 

Review the contract to identify functions that are intended to be called only externally and declare them as external instead of public.


 

