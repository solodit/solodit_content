**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Arbitrary walletAddress leads to potentially locked assets

**Severity**: High

**Status**: Resolved

**Description**

Function `createEscrow` initiates an asset vault that buyer and seller deposit assets to. A risk arises due to the fact that `walletAddress` is being arbitrarily decided as an argument to the function by the caller. An incompatible `walletAddress` can lead to several risks:
- One of the parties buyer/seller have control on that address and drain the funds before the settlement is reached.
-The `walletAddress` refers to a Smart Contract that is not considered a compatible ERC20 holder, hence it is not capable of approving `PreMarketEscrow` to transfer assets leading to assets getting locked in the Smart Contract.

**Recommendation** 

Several measures can be taken to mitigate this issue:
- Create a smart contract to resemble a vault for that `walletAddress` that is controllable by the trusted entities (i.e. `PreMarketEscrow`).
- Receive the funds into the `PreMarketEscrow` itself.

### Method `revertEscrow()` doesn’t work as expected

**Severity**: High

**Status**: Resolved


**Description**

Method `revertEscrow()` has the following logic:
```solidity
function revertEscrow(string calldata refId) external onlyOwner {
   Escrow storage escrow = escrows[refId];


   require(escrow.seller != address(0) || escrow.buyer != address(0), "Escrow does not exist.");


   bool isSeller = msg.sender == escrow.seller; 
   bool isBuyer = msg.sender == escrow.buyer;


   require(isSeller || isBuyer, "Only the escrow creator can cancel before funding.");
… }
```
Here. this method is supposed to be called only by owner but later on it checked that `msg.sender` is either seller or buyer which is contradicting as `msg.sender` can not owner and seller/buyer.

This leads to that issue where seller/buyers wont be able to revert their escrow if it was not accepted.

**Recommendation**: 

Allow only owner to revert the escrow in case sellers/buyers dont find a matching buyer/seller.

## Medium Risk

### Possible loss of funds due to `destroyEscrow()` method

**Severity**: Medium

**Status**: Resolved

**Description**

Method `destroyEscrow()` deletes the escrow details without considering the tokens passed to the wallet address if a seller or buyer has deposited tokens.
If owner calls the `destroyEscrow()` method for such escrows then there is possibility of tokens in wallet address to be stuck.

**Recommendation**: 

Consider checking if seller and/or buyer funds are locked before deleting an escrow and proceed accordingly.

### Method `confirmFailureToTransfer()` doesn’t check for seller funds

**Severity**: Medium

**Status**: Resolved

**Description**

Method `confirmFailureToTransfer()` doesn’t check if the seller has locked funds or not but only checks for buyer funds. 
There could be a case where seller funds are not locked and still the buyer gets `2*amount` when owner calls this method.
This would be loss of funds for the protocol if enough tokens are present in the wallet address to transfer to the buyer.

**Recommendation**: 

Ensure that both seller and buyer funds are locked before transferring 2X funds to the buyer.

### Centralization risk

**Severity**: Medium

**Status**: Acknowledged

**Description**

The smart contracts employ modifiers (e.g. `onlyOwner`) in functions responsible for carrying out actions which can conflict with caller's interest (e.g. `setFeeReceiver`, `destroyEscrow`, emergency withdraw funds...etc). This approach centralizes control, allowing a single owner to exclusively perform critical actions, that involves asset transfer, posing a risk to decentralization principles.

**Risk**: Single Point of Failure: Centralized control introduces a single point of failure, where the compromise of the owner's account can lead to unauthorized access and manipulation of critical functions.

**Recommendation** 

To mitigate the centralization risk, it is recommended to:
- Implement Access Control Lists (ACL): Utilize Access Control Lists to assign different roles with specific permissions, allowing for a more granular control structure.
- Multi-Sig or Governance Contracts: Consider implementing multi-signature schemes or governance contracts to distribute decision-making authority, reducing the risk of a single point of failure. Multi-Sig can be utilized in the project after deployment without altering the codebase. 


## Low Risk

### Missing events for important state updates

**Severity**: Low

**Status**: Resolved

**Description**

Method `emergecyWithdrawForEscrow(...)` and method `emergecyWithdrawForProject(...)`  allows admin to withdraw user funds in case contract is under attack or malicious activity is possible. These methods are not emitting events which can help admins later on to distribute emergency withdrawn funds to users since escrow details in contract is deleted.

**Recommendation**: 

Consider adding events for above mentioned methods as well.

### CEI pattern is not followed

**Severity**: Low

**Status**: Resolved

**Description**

Most of the methods of the contract follow this pattern: (Check, Interaction, Effects)
```solidity
function methodName(string calldata refId) external onlyOwner {
// Checks
   Escrow storage escrow = escrows[refId];
   require(escrow.sellerFundsLocked && escrow.buyerFundsLocked, "Funds not locked");


   uint256 feeAmount = (escrow.amount * feePercentage) / 10000;
// Interaction
   IERC20 token = IERC20(escrow.tokenAddress);


   _transferFromEscrow(token, escrow.walletAddress, escrow.seller, escrow.amount);
   token.safeTransferFrom(feeReceiver, escrow.seller, feeAmount);


   _transferFromEscrow(token, escrow.walletAddress, escrow.buyer, escrow.amount);
   token.safeTransferFrom(feeReceiver, escrow.buyer, feeAmount);
// Effects
   _removeEscrow(refId);


   emit EscrowCancelled(refId);
 }
```
Here, it is advised to update the state before making external calls such as transferring ERC20 tokens to avoid reentrancy attack.

In case, ERC777 tokens are used here as one of the whitelisted tokens, the reentracncy attack is possible if owner is malicious or owner private is compromised. Although the likelihood is low, it is advised to follow CEI pattern.

**Recommendation**: 

Consider calling `_removeEscrow` before token transfers.

### Escrow amount not validated for 0

**Severity**: Low

**Status**: Resolved

**Description**

Method `createEscrow(...)` allows a seller/buyer to create an escrow with amount. But this amount can be 0 as it is not checked if amount > 0 or not.

**Recommendation**: 

Add a validation to check if escrow amount > 0 or not.

### Approved tokens not validated for address(0)

**Severity**: Low

**Status**: Resolved

**Description**

Method `approveTokens()` approves tokens in which sellers/buyers can deposit the tokens but it is not validated for `address(0)`.

If accidentally `address(0)` is approved, `createEscrow()` can be called with `tokenAddress` as `address(0)` and this can lead to unexpected results.

**Recommendation**: 

Add a validation to check `approvedTokens()` are not `address(0)`.

### Missing disable initializer

**Severity**: Low

**Status**: Resolved

**Description**

Contract `PreMarketEscrow.sol` implements the `initialize(...)` method with the initializer modifier without disabling the initializers for the implementation contract as recommended by OpenZeppelin here[https://docs.openzeppelin.com/upgrades-plugins[/1.x/writing-upgradeable#initializing_the_implementation_contract].

**Recommendation**: 

Disable the initializers for the implementation method as suggested by OpenZeppelin here[https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable#initializing_the_implementation_contract].

## Informational

### Unnecessary Safe Math is being utilized

**Severity**: Informational

**Status**: Resolved

**Description**

The default safe math operation in solidity versions ^0.8.x incurs extra gas overhead due to it requiring more computation. The following operation, that is being carried out on the iterator of the for-loop, can be more gas-efficient by using the `unchecked` statement.
```solidity
In function approveTokens(), we have:
 function approveTokens(address[] calldata tokenAddresses) external onlyOwner {
    for (uint256 i = 0; i < tokenAddresses.length; i++) {
      approvedTokens[tokenAddresses[i]] = true;
      emit TokenApproved(tokenAddresses[i]);
    }
  }
```

As well in function `revokeTokens()`.
```solidity
 function revokeTokens(address[] calldata tokenAddresses) external onlyOwner {
    for (uint256 i = 0; i < tokenAddresses.length; i++) {
      approvedTokens[tokenAddresses[i]] = false;
      emit TokenRevoked(tokenAddresses[i]);
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

### Gas Consumption Concerns Due to Extended Error Messages

**Severity**: Informational

**Status**: Resolved

**Description**

The smart contract employs detailed error messages within require statements, offering developers and users specific insights into the conditions that trigger a requirement failure. However, the extensive length of these error messages result in heightened gas consumption.

**Recommendation**

Employ shorter concise error messages.
Use Custom Error objects available from solidity 0.8.0.


### Redundant check

**Severity**: Informational

**Status**: Resolved

**Description**

In line 69, `require(approvedTokens[tokenAddress], "Token not approved")` is a redundant and unnecessary check. Modifier `onlyApprovedToken(tokenAddress)` already carries out that check.
```solidity
60  function createEscrow(
61  address tokenAddress,
62  uint256 amount,
63  string calldata refId,
64  address walletAddress,
65  string calldata creatorRole
66  ) external onlyApprovedToken(tokenAddress) {
67    require(escrows[refId].seller == address(0) && escrows[refId].buyer == address(0), "Escrow already exists!");
68    require(walletAddress != address(0), "Invalid wallet address");
69    require(approvedTokens[tokenAddress], "Token not approved");
```

**Recommendation** 

No need to carry out the require statement checking `approvedTokens[tokenAddress]`.


### Floating Pragma Version in Solidity Files

**Severity**: Informational

**Status**: Resolved

**Description**

The Solidity files in this codebase contain pragma statements specifying the compiler version in a floating manner (e.g., ^0.8.0). Floating pragmas can introduce unpredictability in contract behavior, as they allow automatic adoption of newer compiler versions that may introduce breaking changes.

**Recommendation**: 

To ensure stability and predictability in contract behavior, it is recommended to:
Specify Fixed Compiler Version: Instead of using a floating pragma, specify a fixed compiler version to ensure consistency across different deployments and prevent automatic adoption of potentially incompatible compiler versions.

### Use ENUM for seller/buyer roles

**Severity**: Informational

**Status**: Resolved

**Description**

Method `createEscrow()` has a parameter `createrRole`  to check the role of the `msg.sender`. String uses a lot more gas in this case.

**Recommendation**: 

Use ENUM for the roles.

