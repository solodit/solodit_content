**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Inconsistent Loan Impairment Handling Due to Missing Reset of isImpaired Flag

**Severity**: Medium

**Status**: Resolved

**Description**: 

The removeLoanImpairment function in the LoanManager contract fails to reset the isImpaired flag to false after removing a loan's impairment. As a result, the loan remains incorrectly marked as impaired, leading to inconsistencies in how the loan is managed by the contract. This can cause incorrect calculations of unrealized losses and improper handling of loan defaults.

**Scenario**:

A loan is impaired by calling the impairLoan function, which correctly sets the isImpaired flag to true.
Later, the removeLoanImpairment function is called to remove the impairment. However, the function does not reset the isImpaired flag to false.
Despite the loan no longer being impaired, subsequent operations (such as triggerDefault) continue to treat the loan as impaired, leading to incorrect behavior and calculations.
For example, the unrealized losses calculated in PoolConfigurator::triggerDefault may be overstated because the loan is still incorrectly considered impaired.

**Recommendation**: 

Update the removeLoanImpairment function to include a line that resets the isImpaired flag to false after successfully removing the impairment. This ensures that the loan's state is correctly managed and that subsequent operations handle the loan as expected

### Lack of Validation for Loan Funding Status

**Severity**: Medium

**Status**: Resolved

**Description**: 

The withdrawFunds function does not validate whether the loan has been fully funded before allowing the seller to withdraw funds. This could lead to a situation where a seller attempts to withdraw funds before the loan is adequately capitalized.

**Scenario**:

A loan is created, but for some reason, the funding process is delayed or incomplete.
The seller, without realizing the loan isn't fully funded, calls the withdrawFunds function.
Since there’s no validation check, the function might proceed with transferring funds, leading to an unexpected contract state or even financial discrepancies.

**Impact**: 

This can lead to an inconsistent state where the loan is not properly funded but funds are withdrawn. This could also potentially open up avenues for abuse, where a seller could try to withdraw funds from a partially funded loan.

**Recommendation**: 

Add a validation check in the withdrawFunds function to ensure that the loan's funding status is confirmed before allowing any withdrawal. This can be achieved by checking if the loan’s startDate is set (indicating the loan is funded).

**Client comment**: 

This is confirmed to be an issue: if the Pool Admin has not invoked LoanManager::fundLoan and the seller has already triggered LoanManager::withdrawFunds, the receivable will be transferred from the seller to the LoanManager contract. However, no asset tokens will be transferred since the loan's drawableFunds is not a non-zero amount. This prevents the user from withdrawing the funds again, making the receivable and the corresponding loan unusable.


###  Rounding operations for late interest payments may result in slight overcharges

**Severity**: Low 

**Status**: Acknowledged

**Location**: LoanManager.sol#_getLateInterest,_getInterest

**Description**: 

Users of the protocol are able to utilize the LoanManager contract in order to take out loans which are required to be repaid with interest. Should a user be late on repayments, they will incur a late modifier on their interest rate. This is denoted by the periodicInterestRate through the following formula:

Let interest be 0.12*1e6, currentTime be block.timestamp and dueDate be block.timestamp - 1 days meaning repayments are late by one day. If this is inserted into the formula, a one day late periodic interest rate of 328767123287671 is paid. With this in mind, if the equation is reevaluated to modify the dueDate to block.timestamp - (2 days + 5 hours) meaning the user is late on repayments by 2 days and 5 hours. This would return a result of 986301369863013. If 986301369863013 is divided by 328767123287671 it can be seen that the periodic interest of the 1 day late value perfectly fits into the overdue value by 2 days and 5 hours exactly 3 times which means if a user is late on their repayments by just one second, they will be required to pay the full days interest (in this case, the user will have to pay a 3 day late interest rate when in reality they are late by 2 days and 5 hours). The result is that users could be overpaying on their late interest rates resulting in an unnecessary loss of funds. 

**Recommendation**

It’s recommended that the developers refactor the late interest repayment units to reflect an interest rate per second as opposed to interest rate per day. This is because Solidity will truncate values when dividing in mathematical operations and as a result, rounding errors may cause users to pay more than what is necessary to return their position to a healthy state. 

**Client comment** :

We acknowledge  optimizing the interest calculation per second would improve the accuracy and fairness of the protocol, however, using the current rounding does not pose a significant risk to users.

### Interest rates can be arbitrary which can allow the loan requester to set them to zero bypassing interest and late interest rates if funded

**Severity**: Low

**Status**: Acknowledged

**Location**: LoanManager.sol#requestLoan

**Description**: 

The LoanManager allows a buyer to request a loan from the pool using the requestLoan function. The buyer can pass various parameters to the function such as the receivable asset, the receivables token id, grace period, principal requested and the interest rates as an array. The interestRate (as defined by the 0th index of the rates_ array) and the lateInterestPremiumRate (as defined by the 1st index of the rates_ array) can effectively be arbitrary. This can allow the buyer to specify these as zero in order to bypass interest rates.

**Recommendation**

It’s recommended that the contract admin sets these interest rates using setters within the contract. The interest rates array should be validated against these state variables.

**Client comment**: 

Pool Admin use a multi-signature wallet, such as Safe, instead of a single wallet. This will enhance security and ensure that loan approvals are more robust, reducing the risk of arbitrary or bad loans being approved by a single party..

### The gracePeriod_ parameter in the requestLoan function of the LoanManager can be set indefinitely allowing the attacker to protect themselves from defaults if funded

**Severity**: Low

**Status**: Acknowledged

**Location**: LoanManager.sol#requestLoan

**Description**: 

The requestLoan function of the LoanManager contract allows a buyer when requesting a loan to specify a grace period for the loan. This is seen as a “warning” period for which the borrower will have to return their loan to a healthy state or else their position will be put into a defaulted state by the pool configurator admin. There exists a condition in the triggerDefault function of the loan manager which will check if the loan is past the supplied grace period. 

Because this value can be set to an indefinite value (ie. a date that is excessively far into the future), this can prevent the admin from triggering a default on the loan if repayments aren’t made.

**Recommendation**

Consider allowing the pool configurator admin to set a maximum threshold after a loan due date for which grace periods can be set to prevent overly extensive grace periods. It’s recommended that there are additional validations made on the grace period when requesting a loan. 

**Client comment**: 

 This scenario is similar to the one mentioned in this issue's comments. While the buyer can set arbitrary parameters in the LoanManager::requestLoan function, the request may still be rejected by the Pool Admin.
Pool Admin use a multi-signature wallet, such as Safe, instead of a single wallet. This will enhance security and ensure that loan approvals are more robust, reducing the risk of arbitrary or bad loans being approved by a single party..


### Governable is an abstract upgradeable contract but doesn’t contain any storage gaps which could result in collisions on upgrade

**Severity**: Low

**Status**: Resolved

**Location**: Governable.sol

**Description**: 

The Governable.sol contract is a part of the Receivable and IsleGlobals contracts. For Upgradeable contracts, there must be a storage gap in order to freely add new state variables in future versions without compromising the integrity of the storage compatibility. The absence of storage gaps can result in the overwriting of variables in the child contract if new variables are added to the Goverenable contract. 

**Recommendation**

It’s recommended that storage gaps are added to the Governable contract in order to avoid risk of storage collisions.

**Fix**:  Client addressed and fixed the issue at commit  0a7741f  . 



### Users Might Miss The Withdrawal Window While The Protocol Is Paused

**Severity** - Low


**Status** - Acknowledged

**Description**: 

User withdrawals only happen in respective windows , which is current window +2  , but the user might miss the withdrawal window while the protocol was paused . The pausing mechanism does not extend the window time frame therefore if a user had his withdrawal window between x and y and the protocol was paused within this time frame , he would miss his window and now he has to remove some of his tokens when the protocol resumes in order to push his window to a new current window where he can withdraw.

Also, a loan might get subject to a default while the protocol was paused . In the paused state the user won’t be able to repay the loan and as soon as the system gets unpaused the loan is subject to a default and gets defaulted.

**Recommendation**:

Account for the paused timeframe within the withdrawal window or the users should be acknowledged of such behavior in advance.

**Client comment**: If for some reason the protocol must be temporarily paused, for example, if the pool admin is undergoing legal procedures to recover defaulted funds, we will inform users in advance that the protocol will be temporarily paused.



### NFT Holder Cannot Withdraw Repaid Amount Due to Loan Seller Check

**Severity**: Low

**Status**: Acknowledged

**Location** LoanManager.sol, Receivable.sol

**Description**: 

The Receivable NFT, which represents a transferable debt claim, restricts the ability to withdraw the repayment of the debt to the original lender (loan_.seller) while allowing the transfer of the NFT. In the LoanManager.sol contract, the withdrawFunds() function checks if the caller is the original lender, preventing new NFT holders from accessing repaid funds. This disallows the transfer of the debt along with its claim, leading to potential disputes and misrepresentations in transactions involving the NFT.

**Recommendation** 

Since that the debt is meant to be claimed by the original lender, it would be consistent to have non-transferrable Receivable NFT. This change will ensure that the current owner of the Receivable NFT can rightfully claim the repayment of the debt.

**Client comment#1** : The amount that can be withdrawn is not the repaid amount, it is the early payment amount from Isle’s pool. Right now we fix this issue through a removed seller address check in the withdrawalFunds(), we let the holder of NFT have the right to withdraw.

### Incorrect Sharing of ERC20 Allowance State Across Functions in Pool Contract

**Severity**: Low

**Status**: Acknowledged

**Description**: 

In the Pool contract, the ERC20 allowance state of the shares is shared across multiple functions (requestRedeem, redeem, and removeShares), which can lead to potential misuse by a spender in a manner not intended by the owner. Specifically, if a user approves a certain amount of shares to one spender for requestRedeem and later to another spender for redeem, the second spender could mistakenly or maliciously call requestRedeem or removeShares, exploiting the granted allowance.
This scenario demonstrates that sharing approval states across different functionalities introduces complex behavior not originally intended in the ERC20 standard, which is designed primarily for token transfers between wallets. Such complexities necessitate a more robust logic to ensure security and proper handling of ERC20 allowances.

**Recommendation**: 

To mitigate this issue, it is recommended to introduce a new wrapped token in the Pool contract to represent the shares under the custody of the WithdrawalManager. This wrapped token should have its own allowance and transfer mechanisms, separate from the primary ERC20 token. By doing so, the risk of misuse shall be reduced and align the implementation more closely with the original ERC20 standard's intended use case.
 
Since the ways of spending the allowance are clearly defined within the contract, there’s no need to implement a 1:1 allowance control logic like approve() and transferFrom(). For example, if a user approves an allowance to a spender using an external ERC20, we implement transferFrom() within our requestRedeem, redeem, and removeShares functions. This means the user still can’t control what specific actions the spender might take (if the spender is an EOA). However, if the spender is a verified contract (open source), the user can review the function logic to verify the specific actions being executed.
The reason ERC20 requires a 1:1 approve() and spending logic is to maintain compatibility and flexibility. This setup allows other contracts to directly integrate with transferFrom() to use the allowance, ensuring seamless interaction with various external contracts. Within the ERC20 standard itself, the logic is straightforward: one approve() function is directly tied to the specific spending function transferFrom(), which keeps things clean and simple. However, outside of ERC20, the actual action performed still depends on the function being executed in the contract address (CA) spender or, if it’s an EOA, on the spender’s behavior.
.


### Parallel governor value in IsleGlobal and Receivable contract may result in two separate governor addresses

**Severity**: Low

**Status**: Resolved

**Location**: IsleGlobal.sol,Receivable.sol

**Description**: 

The governor address for the Isle protocol has the authority to make changes and modify settings which are critical to the protocol’s day to day operations. This address is stored in both the Receivable and IsleGlobal contract however, since the address is set twice (once for each contract), this may cause inconsistent addresses resulting in inconsistent data if one were to be changed but not the other due to human error or some other reason. 

**Recommendation**

It’s recommended that the Governable inherited contract is removed from the Receivable contract and the governor address is obtained from IsleGlobal. 

### Incorrect ReentrancyGuard Version Used in Upgradeable LoanManager Contract

**Severity**: Low

**Status**: Resolved

**Description**: 

The LoanManager contract is intended to be upgradeable; however, it imports the non-upgradeable version of ReentrancyGuard from OpenZeppelin. This can lead to issues, as the non-upgradeable version does not have the necessary initializers and storage gap reserved for upgradeable contracts, which are essential for proxy-based upgradeability mechanisms.

**Recommendation**: 

To resolve this issue, replace the inheritance of the non-upgradeable ReentrancyGuard with the upgradeable version from OpenZeppelin, i.e., ReentrancyGuardUpgradeable. Make sure to call the __ReentrancyGuard_init function as part of the initialization process in the initializer function of LoanManager. Additionally, ensure that the contract abides by the storage gap requirements to maintain proper alignment and compatibility between contract upgrades.

**Comment**: 

The client is considering updating the library accordingly.

### maxWithdraw() of ERC4626 Vaults must not revert

**Severity**: Low

**Status**: Resolved

**Description**: 

According to EIP-4626, maxWithdraw() must not revert. But the function maxWithdraw()  reverts when called in the Pool contract which uses the ERC-4626 standard.
According to the EIP, if withdrawals are entirely disabled (even temporarily) it MUST return 0.

**Recommendation**: 

It is advised to NOT revert the maxwithdraw() function but to instead return 0 in order to comply with the original EIP-4626 standard.

### Missing zero address check in Governable can lead to loss of Ownership

**Severity**: Low

**Status**: Resolved

**Description**: 

In Governable.sol’s transferGovernor() function, there is missing zero address check. This can lead to accidentally losing ownership if a zero address is passed as a parameter to this function. This will lead to the admin losing access to the onlyGovernor() functions forever and make it unusable.

```solidity
   function transferGovernor(address newGovernor) external virtual override onlyGovernor {
       // Effects: update the governor.
       governor = newGovernor;


       // Log the transfer of the governor.
       emit IGovernable.TransferGovernor({ oldGovernor: msg.sender, newGovernor: newGovernor });
   }
```
In addition to this, the Receivable.sol’s initialize() function is missing zero address check for initialGovernor_ parameter, which can also make onlyGovernor functions uncallable and make transferGovernor unusable too.


**Recommendation**: 

It is advised to do the following changes:

Introduce a zero address check in the Governable.sol’s transferGovernor() function for the newGovernor parameter
Introduce a zero address check for Receivable.sol’s initialize() function for the initialGovernor_ parameter

### Single-step ownership transfer

**Severity**: Low

**Status**: Resolved

**Location**: Governable.sol

**Description**: 

The transferGovernor function in the Governable.sol contract performs the governance transfer in a single step. This approach poses a risk of accidental ownership transfers, which can lead to potential loss of control over the contract. Implementing a two-step governance transfer process, where the new owner must confirm the acceptance of governance, would mitigate this risk and ensure a more secure ownership transfer mechanism.

**Recommendation**: 

Modify the transferGovernor function to implement a two-step governance transfer process. In the first step, set the new owner as a pending governor. In the second step, require the pending governor to accept the ownership transfer explicitly. This will ensure that the governance transfer is intentional and verified by the new owner, reducing the risk of unintended ownership changes.

**Comment**: 

The client confirmed to implementation a mechanism to prevent unexpected ownership transfers.


### Centralization of Critical Loan Management Functions and Parameter Configurations

**Severity**: Low

**Status**: Acknowledged

**Description**: 

Loan management functions, such as approving loans, marking loans as impaired, reversing impairment decisions, and declaring borrower defaults, are centralized and controlled by an admin. Parameter settings like adminFee, maxCoverLiquidation, minCover, poolLimit, windowDuration, and cycleDuration are also managed by the admin. Although a multisig wallet is intended for these functions to reduce individual risks, significant centralization remains inherent in the protocol.

**Recommendation**: 

To mitigate the risks associated with centralization, consider the following recommendations:

Multisig Wallet Adoption: Implement the intended multisig wallet to add multiple layers of approval for critical functions. Ensure that the multisig contract is robust and secure.
Decentralized Governance: Gradually migrate to a decentralized governance model where the community can partake in significant decisions. Utilize governance tokens and voting mechanisms to distribute the decision-making process.
Time-lock Mechanism: Introduce a time-lock mechanism for critical administrative functions, allowing the community to review and react to changes before they are finalized.

## Informational

### Reentrancy Vulnerability in the createReceivable Function Allowing Multiple Asset Minting

**Severity**: Informational

**Status**: Acknowledged

**Location**: Receivable.sol

**Description**: 

The createReceivable function in the Receivable contract is vulnerable to reentrancy attacks. This vulnerability arises when the _safeMint function is called, leading to the potential for reentrancy exploits where the minting event can execute multiple times, causing inconsistency between the actual state and emitted events.

**Recommendation**: 

To mitigate the reentrancy vulnerability, it is recommended to use a reentrancy guard to prevent the _safeMint function from being invoked multiple times within the same transaction.

**Client comment**:  

This is a valid operation; however, the seller can mint multiple receivables through Receivable::createReceivable by default. Although multiple receivables are created, they must still be approved by the Pool Admin before proceeding with the LoanManager::requestLoan operation. We are uncertain about the potential impact of minting a large number of receivables at once. While re-entrancy can occur during minting, the event emission remains consistent and in the correct order.

**Suggested action**: 

Go back through the code and absolutely assert and triple check that free minting receivables doesn't have any impact to the protocol

### Incorrect Shares Attributed While Depositing 

**Severity** - Informational

**Status** - Acknowledged

**Description**: 

When depositing the Pool contract follows the ERC4626 standard as follows →
```solidity
 function deposit(uint256 assets, address receiver) public override returns (uint256 shares) {
        // Checks: receiver is not the zero address.
        if (receiver == address(0)) revert Errors.Pool_RecipientZeroAddress();


        // Checks: deposit amount is less than or equal to the max deposit.
        if (assets > maxDeposit(receiver)) revert Errors.Pool_DepositGreaterThanMax(assets, maxDeposit(receiver));


        shares = previewDeposit(assets);
        _deposit(_msgSender(), receiver, assets, shares);


        return shares;
    }
```
The shares minted are calculated via the previewDeposit function(which uses _convertToShares()) in the ERC4626 library , but in the isle ecosystem this would be incorrect and more than required shares would be minted to the user . This is because this calculation does not include the unrealized losses in the system . To correctly account for the unrealized losses , instead of using previewDeposit , use  _convertToExitShares() .

The same is applicable for the minting function where _convertToExitShares should be used instead.

**Recommendation**:

Use the recommended functions instead.

**Client comment**: We use unrealizedLoss to create a less favorable exchange rate, hoping to discourage users from withdrawing in the event of a loan impairment. This approach serves as a soft deterrent rather than a direct prevention of withdrawals. The unrealizedLoss itself only affects withdrawals and does not impact deposits.


### Optimization: Redundant Checks in depositWithPermit Function

**Severity**: Informational

**Status**: Resolved

**Location**: Pool.sol

**Description**: 

The current implementation of the depositWithPermit function uses the deposit function, which performs redundant checks that depositWithPermit has already handled.
```solidity
66      shares_ = deposit(assets_, receiver_);
```
This inefficiency can be mitigated by using the previewDeposit function to calculate the share amount and then directly calling _deposit for the actual deposit.

**Recommendation**: 

Replace the line shares_ = deposit(assets_, receiver_); in the depositWithPermit function with:
```solidity
shares = previewDeposit(assets);
_deposit(_msgSender(), receiver, assets, shares);
```
This change eliminates redundant checks and improves the efficiency of the depositWithPermit function.

**Comment**: 

Client confirms to follow the recommendation to fix it.

### The redeem() function reverts on valid values of shares

**Severity**: Informational

**Status**: Resolved

**Description**: 

In Pool.sol, the redeem() function does not allow any value of shares as a parameter to be passed other than a value equal to lockedShares. This is because, redeem() function calls processRedeem() from PoolConfigurator, which itself calls processExit() from WithdrawalManager. 
Line: 200 in Pool.sol
```solidity
 (redeemableShares_, assets_) = IPoolConfigurator(configurator).processRedeem(shares_, owner_, _msgSender());
```
Line: 228 in PoolConfigurator.sol
```solidity
(redeemableShares_, resultingAssets_) = _withdrawalManager().processExit(shares_, owner_);
```
Now in this processExit() function, there is an if statement on line:252 which uses strict equality (or inequality here) which results in a revert if requestedShares_ is not equal to the lockedShares. 
Line: 252 in WithdrawalManager.sol
```solidity
       if (requestedShares_ != lockedShares_) {   
           revert Errors.WithdrawalManager_InvalidShares(owner_, requestedShares_, lockedShares_);
       }
```
        But this strict inequality results in the pool.redeem(uint256 shares) to fail for any value of shares other than lockedShares_. 
        Ideally this is not expected as the function should work for any valid value of shares which can be set to any value as a parameter shares_ of the redeem() function.

**Recommendation**: 

It is advised to change the if statement to > instead of != on line: 252 of WithdrawalManager.sol's processExit() function. Also review the business logic and operational logic for the said changes.

### Lack of reentrancy guards where ERC721 is used increases the risk profile for reentrancy issues

**Severity**: Informational

**Status**: Acknowledged

**Location**: Receivable.sol#createReceivable,LoanManager.sol#withdrawFunds

**Recommendation**
 
There are various locations in the codebase where an ERC721 token is being transferred or minted; however, there are no reentrancy guards used. It’s recommended that for functions that are interacting with external contracts that support a callback functionality (for instance ERC721.safeMint and ERC721.safeTransferFrom calls upon an onERC721Received callback during a safe transfer) uses a reentrancy guard where recursive programming is not desired to mitigate the risk of undiscovered reentrancy vulnerabilities.

### Use of Floating Solidity Version (pragma solidity ^0.8.19) 

**Severity**: Informational

**Status**: Resolved

**Description**: 

The current smart contracts use a floating pragma for the Solidity version (pragma solidity ^0.8.19). While this approach allows for compatibility with future minor releases, it introduces potential risks. Future updates might include changes and newly discovered bugs or vulnerabilities that could affect the contract's functionality and security. Locking the Solidity version would ensure the contract behaves consistently as tested and accounts for all known issues at that version.

**Recommendation**: 

It is recommended to specify a fixed Solidity version for your contracts, such as pragma solidity 0.8.19;, instead of using pragma solidity ^0.8.19;. This will ensure that the contract operates as expected without unintended changes due to future compiler updates.

**Comment**: 

Client confirms to follow the recommendation to fix it in order to prevent any existing compiler-related issues.

### Imprudent Withdrawal of Funds from Impaired Loans

**Severity**: Informational

**Status**: Acknowledged 

**Description**: 

In the current implementation of the LoanManager::withdrawFunds function, there is no explicit check to prevent withdrawals from impaired loans. As a result, even if a loan has been impaired, which indicates a high-risk or non-performing status, the seller can still withdraw the drawable funds associated with that loan. This loophole undermines the purpose of impairing loans and can lead to significant financial losses.

**Scenario**:

A loan is created and funded, with the seller having drawable funds.
Due to some unforeseen circumstances, the loan is impaired, indicating that the loan is at high risk or is no longer performing as expected.
Despite the impairment, the seller calls the withdrawFunds function, withdrawing the funds associated with the impaired loan.
The platform or the pool suffers financial losses because the impaired loan was allowed to proceed with fund withdrawal, contrary to standard risk management practices.

**Impact**: 

Allowing withdrawals from impaired loans can lead to significant financial losses, undermining the integrity of the lending platform. The impairment status should act as a safeguard to prevent any further financial transactions on high-risk loans until the issues are resolved.

**Recommendation**: 

Add Impairment Check:
Modify the withdrawFunds function to include a check that prevents withdrawals if the loan is impaired

**Client Comment** : 
In our protocol design, roles are separated to ensure that the seller can always withdraw funds, while any impairment primarily affects the lender. The issue description states that LoanManager::withdrawFunds prevents potential financial losses, but this is not entirely accurate. The Pool Admin can call LoanManager::fundLoan, transferring funds from the pool contract to the LoanManager contract. As a result, restricting LoanManager::withdrawFunds does not mitigate the impairment scenario
