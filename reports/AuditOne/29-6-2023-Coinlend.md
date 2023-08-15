**Auditors**

[AuditOne](https://twitter.com/auditone_team)

# Findings

## High Risk

### Loans can be liquidated without checking loan duration

**Description**: 

In the provided forceLiquidateExpiredLoan function, it appears that loans are automatically liquidated if the current timestamp is beyond the loan's expiration buffer. However, the function does not account for the loan's duration before proceeding to the liquidation process. The loan might be within its valid duration but due to an insufficient expiration buffer, it may be prematurely marked for liquidation.

**Recommendations**: 

We recommend including a check to ensure that the loan has exceeded its duration before proceeding with the liquidation.



### Assumption of Constant Token Decimal Places Can Lead to Incorrect Value Calculation

**Description:** 

The getLatestPrice function in the provided code snippet assumes that all tokens have 18 decimal places. It multiplies the received price value, which comes with 8 decimal places, by 1e10 to adjust it to 18 decimal places. This is a common assumption as many popular tokens such as Ether use 18 decimal places. However, this is not universally true for all tokens. Some tokens use fewer decimal places and others use more, depending on the token's specific implementation. This could lead to incorrect price calculations if the function is used with tokens that do not have exactly 18 decimal places.

**Recommendations:** 

We recommend dynamically determining the number of decimal places for each token rather than assuming a constant value. Most ERC20 tokens include a decimals() function which returns the number of decimal places for the token. We suggest modifying the getLatestPrice function to call this function and adjust the price calculation accordingly. Please see the sample code below:



### Lack of Token Whitelisting Leads to Deposit of Fake Tokens

**Description:**

 The depositToken function in the current contract implementation allows the deposit of any ERC20 token without a check for its legitimacy. A malicious actor could use this loophole to deposit fake or non-valuable tokens into the contract.

**Recommendations:**

 A token whitelist should be implemented, where only approved tokens can be deposited into the contract. An onlyOwner function should be added to the contract which allows the owner to add and remove tokens from the whitelist.



### Re-entrancy Vulnerability in ERC777 Token 

**Description:** 

The buyCredits function in the CoinlendCredits contract may lead to a potential re-entrancy attack due to the use of an external function call (payToken.transferFrom) before updating the state of the contract (\_mint function).

In a re-entrancy attack, a malicious contract can repeatedly call the buyCredits function within a single transaction, leading to an unpredictable state of the contract and potential theft of tokens.

**Recommendations:**

Consider implementing re-entrancy guard. Status: Resolved.



### Token Can Be Minted With Fake Token Severity: High

**Description:**

 The current implementation of the buyCredits function in the CoinlendCredits contract allows minting of the token using any arbitrary ERC20 token. This could potentially lead to fraudulent activities where a malicious user could mint CoinlendCredits tokens using worthless or "fake" ERC20 tokens.

**Recommendations:**

The buyCredits function should include a check to verify the legitimacy and value of the ERC20 token used for minting CoinlendCredits. This could be done by maintaining a list of acceptable ERC20 tokens that the buyCredits function accepts. This list can be managed by the owner of the contract to ensure the authenticity and value of the tokens accepted.



### Significant Rounding Error in Loan-to-Value Calculation

**Description:**

In the getLTVOfLoan function, the calculation of the loan-to-value (LTV) ratio involves a division by 1e18 for both tokenLendingTotalValue and tokenCollateralTotalValue. This division can lead to a significant loss of precision, especially for small amounts of tokens. This could potentially result in the LTV ratio being rounded down to zero, even when the actual value should be higher. This could prevent loans from being liquidated when they should be, leading to potential loss of funds for lenders.

**Recommendations:** 

To mitigate this issue, remove division by 1e18 as it is not necessary due to the fact that we divide tokenLendingTotalValue by tokenCollateralTotalValue afterwards, which cancel the decimal effect. This will help to increase the LTV precision, to get an accurate value of the borrower's position.



### USDC used to buy COINC are locked in the contract forever

**Description:**

In the CoinlendCredits contract, users can use USDC to mint COINC tokens. However, there is no way to redeem or withdraw the USDC in the contract. As a result, the amount of USDC deposited to the contract is locked forever.

**Recommendations:**

Consider adding a function to allow users to redeem COINC for USDC in the CoinlendCredits contract.



## Medium Risk

### Lack of Upper Bound Check on interestRate May Lead to Excessive Interest Rates 

**Description:** 

The createLoanLend & createLoanBorrow functions currently check whether the interestRate is at least 1, but there is no upper bound check, allowing for excessively high interest rates. Moreover, the code seems to mistakenly check ltv again in the same line, which is probably a coding error.

**Recommendations:**

 An upper bound should be set for the interestRate to ensure that it does not exceed a reasonable value. The appropriate limit can be determined based on platform policies or market conditions. Also, correct the error in the require statement, where ltv is being checked instead of interestRate.



### Incompatibility With Rebasing/Deflationary/Inflationary tokens

**Description:**

 The protocol do not appear to support rebasing/deflationary/inflationary tokens whose balance changes during transfers or over time. The necessary checks include at least verifying the amount of tokens transferred to contracts before and after the actual transfer to infer any fees/interest.

**Recommendations:** 

Ensure that to check previous balance/after balance equals to amount for any rebasing/inflation/deflation

Add support in contracts for such tokens before accepting user-supplied tokens

Consider supporting deflationary / rebasing / etc tokens by extra checking the balances before/after or strictly inform your users not to use such tokens if they don't want to lose them.



### Incorrect Fee Verification in Loan Payback Severity: Medium

**Description:** 

The hasBorrowerEnoughFundsForPayback function checks if the borrower has enough funds to pay back the loan, the interest, and the fee. However, the fee is a portion of the interest amount, not an additional cost. This means that the borrower is required to have more funds than necessary to pay back the loan.

**Recommendations:**

The fee should not be considered in the verification, not added to it. This will ensure that the borrower is only required to have enough funds to cover the loan amount and the interest (minus the fee).



### Should check return data from chainlink aggregator Severity: Medium

**Description:**

 The latestRoundData function in the contract Coinlend.sol fetches the asset price from a Chainlink aggregator using the latestRoundData function. However, there are no checks on roundID.

Stale prices could put funds at risk. According to Chainlink's documentation, This function does not error if no answer has been reached but returns 0, causing an incorrect price fed to the PriceOracle. The external Chainlink oracle, which provides index price information to the system, introduces risk inherent to any dependency on third-party data sources. For example, the oracle could fall behind or otherwise fail to be maintained, resulting in outdated data being fed to the index price calculations of the [liquidity.](https://github.com/code-423n4/2021-08-notional-findings/issues/18)

**Recommendations:** 

Consider to add checks on the return data with proper revert messages if the price is stale or the round is incomplete.



### Uninitialized Discount Variable Severity: Medium

**Description:**

 The coinlendCreditsDiscount variable is used in the payBackLoanInternal function to calculate the fee discount for users who hold CoinlendCredits tokens. However, this variable is never initialized and does not have a setter function, which means it defaults to zero. As a result, users who hold CoinlendCredits tokens will never receive a discount on their fees.

**Recommendations:**

To fix this issue, a setter function should be added to allow the contract owner to set the coinlendCreditsDiscount variable. This function should only be callable by the contract owner to prevent unauthorized changes to the discount rate.

Additionally, the coinlendCreditsDiscount variable should be initialized in the contract constructor to a sensible default value. This will ensure that users who hold CoinlendCredits tokens receive a discount on their fees from the moment the contract is deployed.



### ERC20 Approve Race Condition 

**Description:** 

The CoinlendCredits contract is vulnerable to a well- known race condition in the ERC20 standard. This race condition can occur when a user calls the approve function of an ERC20 token, then calls a transferFrom on the same token.

The issue arises when, between the approve and transferFrom calls, another transaction is inserted that also calls transferFrom. This can lead to the user's tokens being spent twice.

**Recommendations:**

The issue can be remediated by the use of increaseAllowance and decreaseAllowance functions to modify the approvals.



### Lack of Ownership Transfer Functionality

**Description:**

 The contract does not provide a way to change the owner of the contract. This is a potential issue because if the owner's keys are compromised, there is no way to transfer ownership to a new address. Additionally, if a governance system is implemented in the future, there is no way to transfer ownership to a DAO or other governance contract.

This is a common feature in many smart contracts and is usually implemented with a transferOwnership function that can only be called by the current owner. The function takes a new owner address as a parameter and sets the owner state variable to this new address.

**Recommendations:**

Implement a transferOwnership function that allows the current owner to transfer ownership to a new address. This function should emit an event to record the change of ownership. Also, consider implementing additional security measures, such as a delay or multi-signature requirement for ownership transfers.



### Incorrect interestRate check allows users create unreasonable high interestRate loan

**Description:**

The interestRate check in the createLoanLend() function is incorrect. Instead of checking interestRate <= 1000, it mistakenly checks ltv <= 1000. The value of ltv is already validated earlier to always be smaller than maxLTV. This missing upper bound check allows users to create loans with interest rates of 100% or 1000%.

**Recommendations:**

Consider fixing the interestRate check to interestRate <= 1000 to ensure that users cannot create loans with unreasonably high-interest rates.



### New fee should not be applied to ACTIVE loans Severity: Medium

**Description:** 

The owner can change the fee by using the setFeePercent() function. The problem is that this new fee is also applied to ACTIVE loans, which is not agreed upon by the users at the time of creation. Users only agreed to pay the old fee when they created a loan in the protocol, so existing loans in the protocol should not be applied with the new feePercent.

**Recommendations:** 

Consider recording the current fee to the Loan struct at the creation time to avoid this issue.



### Owner Should Not be Able to Mint COINC to Their Wallets

**Description:**

 In the CoinlendCredits contract, the owner can mint an infinite amount of tokens to their wallet, even though it is only necessary to mint to the contract address for users to call buyCredits().

**Recommendations:**

 Consider minting to address(this) instead in the mint() function.



### Borrower can abuse late repayment to always repay the loan late

**Description:** 

In the protocol, borrowers are allowed to repay the loan late without any penalty or fees. Even when the lender calls the function forceLiquidateExpiredLoan(), it still checks if the borrower can repay first and tries to repay instead of liquidating the collateral. However, during the late repayment period, the borrower only needs to pay the same interest rate and no penalty or fee is applied. As a result, it could be an incentive for borrowers to always repay late, in spite of the duration agreed upon by both parties.

**Recommendations:**

Consider applying a fee when the borrower repays late or a higher interest rate for the late repayment period.



### Inability to use common ERC20 tokens without success value return in protocol 

**Description:**

 Currently, the protocol relies on the require() function to validate the return value of ERC20 token transfers. For instance, in the depositToken() function:

However, this approach poses a challenge when dealing with tokens that do not return a success value. Notably, tokens like USDT, as seen in their contract code, may not be compatible with the protocol. The require statement would fail even if the transfer is successful, as it expects a boolean value to be returned.

**Recommendations:**

To address this issue, it is recommended to incorporate the SafeERC20 library. This library enables the transfer of ERC20 tokens that do not provide a return value upon successful transfers, ensuring compatibility with the protocol.



## Low Risk

### Lack of Comparison between minLTV and maxLTV Could Break Functionality

**Description:**

In the current contract design, there are setter functions setMinLTV() and setMaxLTV() for updating the minLTV and maxLTV variables. However, there is no validation logic to ensure that minLTV is always less than or equal to maxLTV. If minLTV is accidentally or maliciously set to be greater than maxLTV, it could cause the loan creation function to fail or behave unpredictably.

**Recommendations:** 

Introduce a comparison check in the setter functions to ensure that minLTV is always less than or equal to maxLTV. In the setMinLTV() function, check if the \_minLTV being set is less than or equal to the current maxLTV. In the setMaxLTV() function, check if the \_maxLTV being set is greater than or equal to the current minLTV. If either condition is not met, the function should revert with an appropriate error message.



### Ownership can not be transferred Severity: Low

**Description:** 

In the constructor function of the contract, the ownership of the contract is set to be the address that deploys the contract (msg.sender). There is no mechanism provided to transfer the ownership of the contract to another address. The absence of such a feature means that if the initial owner loses access to their account, the contract cannot be administered, leading to potentially serious consequences.

**Recommendations:** 

Introduce a function that allows the contract owner to transfer ownership to another address. This function should only be callable by the current owner to ensure security. A common practice is to use the Ownable pattern (from OpenZeppelin, for example), which includes an onlyOwner modifier and transferOwnership function. The transferOwnership function should emit an event to provide visibility of ownership changes.



### Centralization Risk Due to Exclusive Minting Rights for Contract Owner

**Description:**

The mint function of the contract is currently designed to be callable only by the contract owner (onlyOwner). This functionality enables the contract owner to arbitrarily create (mint) new tokens without any restrictions or predefined rules.

**Recommendations:** Consider introducing decentralized governance for minting new tokens, or establishing clear, predefined rules for when and how new tokens can be minted. These rules could be based on specific network events, thresholds, or tokenomics principles.

Alternatively, you might want to consider having a multisignature contract or a DAO to govern the minting process. This would ensure a more democratic process for minting new tokens, with decisions made collectively rather than by a single entity.



### The Contract Owner Will Always Be an Externally Owned Account (EOA)

**Description:**

During the code review, It has been noticed that owner is directly set to msg.sender in the constructor. From that reason, an owner always will be EOA.

**Recommendations:** 

Consider using Ownable2Step contract from Openzeppelin.



### Lack of Upper Bound Check in setFeePercent Function 

**Description:**

 The setFeePercent function currently does not enforce an upper limit on the value of \_feePercent. This could potentially lead to scenarios where the fee is excessively high.

**Recommendations:** 

Implement an upper bound check in the setFeePercent function to prevent setting unreasonably high fees.



### depositToken Accepts Zero Amount 

**Description:**

In the depositToken function, there is no check for the \_amount parameter. If a user calls this function with 0 as the amount, the function will still execute, incrementing the transferCount and emitting a Deposit event with a zero amount. .

**Recommendations:** 

It's recommended to add a requirement that the \_amount must be greater than 0.



### Loan with id = 0 can be taken without creation

**Description:**

In the takeLoan() function, \_loan.id == \_id is checked to confirm the existence of the loan. However, when \_id = 0, this check is passed even if no loan has been created on the protocol yet. Additionally, since LoanState.OPEN = 0, which is also the default value, the check for the loan state is also passed. As a result, users can take out a loan with id = 0 before any loan has been created on the protocol.

**Recommendations:**

Consider fixing the issue by initializing loanCount = 1 so that a loan with id = 0 is considered invalid.



## Informational

### Redundant Hardhat Console 

**Description:**

During the code review, It has been noticed that the contract uses redundant console log. It should be deleted before deployment.

**Recommendations:** 

Consider deleting redundant console output.



### Redundant Timestamp Update Severity: Quality Assurance

**Description:**

 The takeLoan function updates the timestamp field of a Loan struct twice with the same value (block.timestamp). This is redundant and increases the gas cost of the function.

**Recommendations:**

 Remove one of the timestamp updates to reduce the gas cost of the takeLoan function. Since the timestamp value is the same in both updates, removing one of them will not affect the functionality of the contract.



### Storage Optimization

**Description:**

 The Loan struct uses uint256 for all its numeric fields. However, some of these fields such as interestRate, loanDuration, and timestamp can be represented with smaller data types, which would reduce the storage cost.

**Recommendations:** 

Consider using smaller data types for interestRate, loanDuration, and timestamp. For example, interestRate and loanDuration can be represented as uint32 (since the maximum interest rate is 1000 and the maximum loan duration is 3650), and timestamp can be represented as uint64 (since the Unix timestamp will not exceed 2^64 - 1 until the year 584942417355).

Additionally, Solidity packs multiple smaller variables into a single storage slot if possible, so you can further optimize storage by ordering the fields in the struct to maximize packing.



### Importing Debugging Libraries in Production Code Severity: Quality Assurance

**Description:**

The contract imports the Hardhat console library import "hardhat/console.sol";. This library is used for debugging during development and should not be included in production code. Including this library in production code could potentially introduce security vulnerabilities and increase gas costs.

**Recommendations:**

Remove the import statement for the Hardhat console library from the production version of the contract. If logging is required for production, consider implementing a custom event that emits the necessary information.

 

### Floating Pragma Statement Severity: Quality Assurance

**Description:**

 The contract uses a floating pragma statement pragma solidity ^0.8.7;. This means that the contract can be compiled with any compiler version from 0.8.7 (inclusive) up to, but not including, version 0.9.0. This could potentially introduce unexpected behavior if the contract is compiled with a newer compiler version that includes breaking changes.

**Recommendations:** 

It is generally recommended to lock the pragma statement to a specific compiler version to ensure that the contract behaves as expected. This can be done by removing the caret (^) from the pragma statement.

