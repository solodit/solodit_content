**Auditors**

[ZOKYO](https://x.com/zokyo_io)

# Findings

## High Risk

### Use of arbitrary token addresses.
**Description**

LockToken.sol: functions lock Tokens(), lockNFTs(), migrate(). These functions are public, and any token addresses are passed in their parameter lists. This creates a vulnerability for the use of malicious tokens.

**Recommendation**

Add token whitelisting to validate any token addresses provided by users.

**Re-audit comment**

Verified.

Post audit:

TrustSwap team verified that this is an intended logic and users should be able to use arbitrary token addresses. Yet, usage of nonReentrant modifier in all functions with token address should be enough to mitigate any critical vulnerabilities.

### Reentrancy in main functions.
**Description**

LockToken.sol:

1. withdraw Tokens(). External calls of .safeTransferFrom() on line 345, .transfer() on line 364, .burn() on lines 349 and 367 before writing of the nftMinted[_id] state variable with the false value on lines 350 and 368 respectively.
2. migrate(). External calls of .createAndInitialize PoollfNecessary(), v3Migrator.migrate(), .transfer(), and so on before writing of the state variables with_removeERC20Deposit(), and so on.
3. lockNFTs(). External calls of .safe TransferFrom(), .mintLiquidityLockNFT() before writing of a lot of state variables.
4. lockTokens(). Similarly to the function lockNFTs().

**Recommendation**

Use the nonReentrant() modifier from the OpenZeppelin ReentrancyGuard contract for these functions.

**Re-audit comment**

Resolved

## Medium Risk

### Returned value after the call is not checked.
**Description**

LockToken.sol: function_chargeFees(), line 762.

Variable "refund Success", returned after the call is not verified to be true. Thus, in case the call is reverted, the whole transaction won't revert.

**Recommendation**

Add validation of "refundSuccess" or verify that the check is not necessary.

**Re-audit comment**

Verified.

From client:

The call is performed for refunding extra ETH to the user. A failure of such a refund should not affect call execution.

## Low Risk

### Extra check.
**Description**

LockToken.sol: there is an unnecessary requirement of (_tokenld $>=0,$ "Invalid token id") on line 174 because this condition is provided by checking for the uint256 type.

**Recommendation**

Remove extra check.

**Re-audit comment**

Resolved

## Informational

### Outdated Solidity version usage.
**Description**

Currently the protocol utilizes Solidity version 0.6.2. The general auditor's security checklist recommends to utilize the newest stable version of Solidity. The newest version of Solidity includes the latest optimization for the compiler, the latest bugfixes and features such as built-in overflow and underflow validations.

**Recommendation**

Use the latest version of the solc and OpenZeppelin products.

**Re-audit comment**

Unresolved.

From client:

Due to the contract architecture, the version won't be updated as this can lead to possible storage issues connected to Openzeppelin contracts.

### Non-informative LogWithdrawal event.
**Description**

LockToken.sol: the LogWithdrawal event only reports the destination address and transferred amount but does not carry information about the token address and token ID in the case of an NFT.

**Recommendation**

Extend this event and add an event for the NFT case.

**Re-audit comment**

Resolved

### Unnecessary gas consumption points.
**Description**

LockToken.sol:

• There are unaggregated writings of struct objects to the storage on lines 138-142, 184-189, 677-682.

• There are multiple accesses to storage without a pointer. On lines 135, 181, 276, 280, 304, 308, 685, 701, 716, 730-735, etc.

The following functions are not called internally but have public visibility: lockTokens(), lockNFTs(), extendLockDuration(), transferLocks(), withdraw Tokens(), setNFTContract(), getTotal TokenBalance(), getTokenBalanceByAddress(), getAllDepositlds(), getDeposit Details(), getDepositsByWithdrawalAddress(), migrate(), mintNFTforLock(), transferOwnershipNFTContract().

LockNFT.sol:

• The following functions are not called internally but have public visibility: setLockTokenAddress(), burn().

**Recommendation**

Optimize gas usage by:
1. Using struct assignments for storage writes (e.g., for lines 138-142 "lockedToken_id] = Items(_tokenAddress, _withdrawalAddress, amountin, _unlockTime, false);", similarly for others).
2. Using storage references for multiple accesses to the same storage variables.
3. Declaring functions called only externally as `external` instead of `public`.

**Re-audit comment**

Resolved

### Extra getter function.
**Description**

LockToken.sol: the walletTokenBalance mapping has a public visibility, so there is no need in the function getTokenBalanceByAddress().

**Recommendation**

Remove the extra getter.

**Re-audit comment**

Resolved

### Lack of events for state variable changes.
**Description**

LockToken.sol: there are no events about writing the state variables for functions extendLockDuration(), setFeeParams(), setFeesInUSD(), setCompanyWallet(), setNon Fungible Position Manager(), setV3Migrator(), setNFTContract(), addTokenToFreeList(), remove Token FromFreeList().

LockNFT.sol: setLockTokenAddress().

This makes it difficult to track changes of the storage when using the contract.

**Recommendation**

Add events about writing the state variables.

**Re-audit comment**

Resolved.

From client:

Events were added where necessary. In order not to increase contract size, some setters won't have an event.

### Unused function parameters.
**Description**

NFTSVG.sol: generateSVG1(), generateSVG2(), generateSVG3(), lines 53, 72, 85: function parameter "params" is not used.

LockNFT: mintLiquidity LockNFT(), line 39: parameters "_tokenAddress", "_tokenAmount", "_tokenType", "_unlockDate" are not used.

NFTDescriptor.sol: tokenURI(), line 10: variable "nftTokenld" is not used.

**Recommendation**

Remove unused parameters.

**Re-audit comment**

Unresolved

### Failing tests in the project.
**Description**

All tests show an error in deploying the LockNFT contract (missing NFTDescriptor implementation). If the implementation is fixed, there is a problem in "Transfer/Withdrawal Lock TestCases" in revert reason (Expected transaction to be reverted with the reason 'Unauthorized to unlock', but it is reverted with the reason 'Unauthorised to unlock'). In "LockToken TestCases for ERC20 Tokens", check the withdrawal address and owner of lockNFT. In "LockToken TestCases for ERC20 Tokens", not enough funds to send transaction.

**Recommendation**

Add an implementation of NFTDescriptor, fix revert string in the contract, fix the address check, send less or top up account to send tx.

**Re-audit comment**

Unresolved
