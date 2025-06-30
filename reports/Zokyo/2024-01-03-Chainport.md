**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Fail because of incompatibility of Standard 'approve()` Function with USDT Token.
**Description**

Arbitrage Manager: release OrMintAndSwap, swap, swapAnd Deposit OrBurn, _depositOrBurn.
It is known that the USDT implements an ERC20 version that does not return a 'bool' value from the 'approve() function, whereas the expected behavior for ERC20 tokens is to return true` upon successful execution of 'approve()`. This discrepancy leads to the possibility that standard 'approve()` calls may revert transactions when interacting with USDT. The issue is marked as High, as it blocks the work of the contract, and auditors confirmed the case with the appropriate test.

**Recommendation**

Implement the 'forceApprove() function from OpenZeppelin's SafeERC20 library, which accommodates such exceptions and ensures correct approve()` behavior for tokens with non-standard implementations. OR confirm that USDT (or any other sub-standard token implementations) will not be used within the protocol.

**Re-audit comment**

Resolved.

Post audit.
The usual approve has been replaced with safeApprove since contracts currently use a solidity version lower than 0.8.0.

## Low Risk

### Unused variables and event.
**Description**

ChainportSideBridge.sol.
The contract has unused variables maxLpDrainage Percent, minting Protection Time, erc20ToBep20Address, functionNameToNonce, maintainer WorkInProgress, lastLpDrainageInitTimestamp, totalMinted FromLastDrainageInit, isSignatureWhitelisted, token To Threshold and event SignatureWhitelist, which may lead to confusion and potential issues in the future.

**Recommendation**

Remove the unused variables to avoid confusion and potential issues OR add logic in which these variables will be used OR verify that they are needed for future functionality.

**Re-audit comment**

Verified.

Post audit.
The Chainport team verified that variables cannot be removed due to the proxy pattern. Though variables are marked as deprecated.

## Informational

### Lack of validation
**Description**

ChainportMainBridge:
1) initialize() → address_maintainers RegistryAddress, address_chainportCongress, address _signatureValidator;
2) release Fees By Maintainer() → address token, address user;
3) releaseTokens By Maintainer() → address token;
4) releaseTokens() → address token, address payable beneficiary;
5) nonEVMDeposit Tokens() → address token, string calldata receiverAddress;
6) depositTokens() → address token;
7) setPathPauseState() → address token;
8) setFeeCollector() → address feeCollector_;
9) setArbitrage ManagerAnd Limit() → address_arbManager;
Arbitrage Manager:
1) initialize() → addresses_congress, _maintainers Registry;
2) swapAndDepositOrBurn() → address[] memory path;
ChainportSideBridge:
1) initialize() → address_chainportCongress, address_maintainersRegistry, address _signatureValidator;
2) mint New Token() → address original TokenAddress;
3) mint New Token FromNonEVM() → string memory original TokenAddress;
4) burnTokens() → address bridgeToken;
5) nonEVMCrossChainTransfer() → address bridge Token, string calldata receiverAddress;
6) setSignatureValidator() → address_signatureValidator;
7) setFeeCollector() → feeCollector_;
8) setArbitrage ManagerAndLimit() → address_arbManager.
The zero address check is a standard validation to prevent initializing contracts without valid addresses. Add necessary checks to ensure that none of the addresses are equal to the zero address before setting them.

**Recommendation**

Add the necessary validation.

**Re-audit comment**

Unresolved.

Post-audit.
ChainportMainBridge:
1. initialize(): Client has added necessary checks for _maintainers RegistryAddress and _chainportCongress. However, the client stated that the '_signatureValidator can be set to a zero address on initialization.
2-3. release Fees By Maintainer() and release Tokens By Maintainer(): The client's response indicates that the use of 'safeTransfer precludes the need for an additional zero address check for the token. However, it is noted that the client's response did not address the absence of validation for the 'user' parameter within the 'release Fees By Maintainer()` function. While the function is expected to be called by a maintainer, the lack of validation introduces the risk of inadvertently passing a zero address. This omission could result in incorrect event logging, where the 'user' is used exclusively for events and should ideally be validated to prevent any zero address from being recorded
4-7. release Tokens(), nonEVMDeposit Tokens(), deposit Tokens(), and setPathPauseState(): The client trusts the 'safeTransfer function to handle checks against the token being a zero address. Also, beneficiary is validated by the signer and the receiver address is validated by its length, which might be sufficient given the design.
8. setFeeCollector(): The client prefers the flexibility to disable a role by setting it to zero, which is countered by checks in other functions against the feeCollector being a zero address.
9. setArbitrage ManagerAndLimit(): As with the fee collector, the client wants the ability to set_arbManager to zero for disabling purposes.
Arbitrage Manager:
1. initialize(): Checks for '_congress` and `_maintainersRegistry are accounted for in middleware initialization according to the client. This is acceptable provided that the middleware effectively enforces these checks.
2. 'swapAndDepositOrBurn(): The client relies on Uniswap contract reverts for path validation and does not find additional checks necessary.
ChainportSideBridge:
1. initialize(): The client has ensured checks for_chainportCongress and _maintainersRegistry in middleware but not for_signatureValidator. It remains optional for the same reasons mentioned in the Chainport MainBridge.
2-3. mintNew Token() and mint New Token FromNonEVM(): The client sees no issue with minting a zero address, stating it would not harm the project. While this might not have immediate detrimental effects, preventive measures or validations can be beneficial.
4. burnTokens(): The client thinks that burning with a zero address results in a revert, but due to the absence of explicit error messaging, documentation and caution are advised.
5. nonEVMCrossChainTransfer(): The same response is given as in nonEVMDeposit Tokens on the main bridge, relying on other checks.
6. setSignatureValidator(): Necessary checks have been added.

### Inaccurate version pragma.
**Description**

`pragma solidity` ^0.6.12 is used in contracts. It is recommended to use the latest version of Solidity and specify the exact pragma. Older solidity versions may contain bugs and vulnerabilities, and be less optimized in terms of gas.

**Recommendation**

Specify the latest version of Solidity in the pragma statement.

**Re-audit comment**

Acknowledged.

Post-audit.
The latest version of Solidity in the pragma statement wasn't specified. Still, the team verified that they plan a full architecture upgrade in the future, including custom errors and the latest Solidity version.

### Custom errors should be used.
**Description**

Starting from the 0.8.4 version of Solidity, it is recommended to use custom errors instead of storing error message strings in storage and using "require" statements. Using custom errors is more efficient regarding gas spending and increases code readability.

**Recommendation**

Use custom errors.

**Re-audit comment**

Acknowledged.

Post-audit.
Custom errors aren't in use now, but the team verified that they plan a full architecture upgrade in the future, including custom errors and the latest solidity version.

### Inconsistent require statements.
**Description**

The following functions have require statements without error messages:
ChainportMainBridge.sol:
1) setAddresses WhitelistState() → require(addresses[i] != address(0));
2) setFeeCollector() → require(feeCollector != feeCollector_);
3) withdrawGasFee() → require(success);
4) setArbitrage Manager And Limit() require(_arbManagerReleaseLimit PercentB $<=10000$ && _arbManager ReleaseLimitPercentTS $<=10000)$;
ChainportSideBridge.sol:
1) setFeeCollector() → require(feeCollector != feeCollector_);
2) withdrawGasFee() → require(success);
3) setArbitrageManagerAndLimit() → require(_arbManagerMintLimitPercentTS $<=10000)$.
ArbitrageManager.sol:
1) initialize() →
a) require(_mainBridge != address(0)),
b) require(_sideBridge != address(0)),
c) require(_router != address (0) && routerId[_router] $==0$,
d) require(_operator != address (0) && operatorId[_operator] $==0$,
e) require(_whitelist != address(0) && whitelistsId[_whitelist] == 0);
2) getRouterById() → require(id != 0);
3) getOperatorById() → require(id $!=\emptyset$);
4) getWhitelistById() → require(id $!=\emptyset$);
5) withdraw() → require(tokens [i].balanceOf(address(this)) >= amounts[i] && amounts $[i]>0)$.

**Recommendation**

Consistency in error handling improves the comprehensibility and debuggability of the code. It is recommended to either add descriptive error messages to all "require" statements or follow the best practices suggested in the issue Custom errors should be used, adopt the use of custom errors, which provides gas optimization and improves the developers' and users' ability to understand the failure reason. This approach can enhance the overall security and usability of the contracts.

**Re-audit comment**

Verified.

Post-audit.
The Chainport team verified that the mentioned function will be called by authorities and thus does not need additional bytes spent for messages due to the short troubleshooting process.

### Inheritance version is different.
**Description**

ChainportSideBridge and ChainportMain Bridge are found to inherit from ChainportMiddleware, while ArbitrageManager inherits from an apparently more recent version, Chainport MiddlewareV3.

**Recommendation**

Verify that the use of ChainportMiddleware and Chainport MiddlewareV3 aligns with project requirements.

**Re-audit comment**

Verified.

Post-audit.
Verified by the Chainport team.

### Lack of events
**Description**

1) ChainportSideBridge.sol: setSignatureValidator, setOfficialNetworkId, setMinting FreezeState, setToken ProxyAdmin, setTokenLogic, setFeeCollector.
2) ChainportMainBridge.sol: setOfficial NetworkId, setFeeCollector.
In order to keep track of historical changes of storage variables, it is recommended to emit events on every change in the functions that modify the storage.

**Recommendation**

Emit events in all functions where state changes to reflect important changes in contract.

**Re-audit comment**

Resolved.

Post-audit.
Events are emitted now.

### Missing Transaction Reverts for Zero Address Checks.
**Description**

ChainportSideBridge: _setTokenProxyAdmin, _setTokenLogic.
The functions_setTokenProxyAdmin() and_setTokenLogic() contain checks to prevent setting their respective addresses to the zero address. However, these checks are currently silent that is, if a zero address is provided as an argument, the function will simply not update the state, but the transaction will not revert and will appear successful. This behavior might lead to a misunderstanding, where calling functions believe that they have successfully set a new address when, in fact, no update has occurred due to providing an invalid zero address. Ensuring that transactions revert with an informative error message when critical checks fail is a fundamental security best practice in smart contract development. If the function's intended behavior is to disallow a zero address, it should be enforced with a revert statement that clearly indicates why the transaction failed. This ensures transparency and helps prevent potential errors during contract interactions.

**Recommendation**

Update the functions to explicitly revert if the input argument is the zero address.

**Re-audit comment**

Verified.

Post audit.
The team verified that this behavior is intentional as the initialization flow is made to cross the roles of maintainer and congress authorities, and there is an initialization function so that initially, the maintainer can do it but may lack an address, so it is possible to set only one address and leave the other one zero.

### Unused Role.
**Description**

The address of fundManager is declared within the Chainport Main Bridge contract, with functions provided to set and update its value. However, no active functions or codes within the contract demonstrate any interaction by the fund manager. This suggests that the fundManager currently does not have a defined purpose or functionality. Leaving the fundManager without explicit use in the contract can lead to confusion regarding the contract's capabilities and purpose.

**Recommendation**

Verify the need for the fundManager within the project: The fundManager is intended for future use OR if it is an unnecessary part of the contract with no planned use, consider removing the related functions to simplify contract management and mitigate any associated risks.

**Re-audit comment**

Resolved.

Post-audit.
The functionality including fundManager was removed, the variable remained but marked as deprecated.

### Inconsistent Error Reporting in Require Statements.
**Description**

Throughout the provided smart contracts ChainportSideBridge and ChainportMainBridge, there is an observed inconsistency in the format of error messages used within require' statements. Some contracts use short, code-based error messages like "E33," whereas others use descriptive text such as "Only maintainer can call this function." While both methods serve to halt execution when conditions are unmet, the inconsistency may lead to confusion and could potentially hinder the debugging and maintenance process.

**Recommendation**

Establish a consistent error reporting convention throughout the smart contract codebase. Below are steps to improve error messaging consistency:
1. Use of custom error types introduced in Solidity 0.8.4, which can provide gas efficiency alongside descriptive power.
2. If retaining code-based error messaging, maintain a detailed and easily accessible documentation or a legend within the codebase to decode error messages.
3. If descriptive text messages are preferred, ensure they are well-crafted to be informative yet concise, adequately describing the reason for transaction failure.

**Re-audit comment**

Verified.

Post-audit.
The Chainport team verified that they will strive for the convention to apply rules of errors. That will be either via custom errors once they increase the solidity version or via short naming like "AB01" which they documented/linked with proper messages.

### Violations of Solidity Style Guide Conventions - confusion naming.
**Description**

1. Variable naming inconsistencies: Presence of variables like 'value', '_value`, and value_` within the same codebase which could cause confusion.
More specifically:
a) In ChainportSideBridge some functions have parameters in the form of_value, while other functions do not. Functions with parameters in the form of_value: setArbitrage ManagerAndLimit, setBlacklistRegistry, initialize TokenProxyAdminAndLogic, setTokenLogic, setTokenProxyAdmin, setSignature Validator, setAssetFreezeState → bool _isFrozen, initialize.
b) Same in ArbitrageManager with: initialize.
c) Same in ChainportMainBridge: initialize, setSignatureValidator, setAsset Freeze State → bool_isFrozen, setBlacklistRegistry, setArbitrage ManagerAndLimit.
d) There are parameters of the form_value, and value, and value_ in ChainportMiddlewareV3:
event CongressSet(address indexed_chainportCongress) and
event Maintainers RegistrySet(address indexed_maintainersRegistry) while event BlacklistRegistrySet(address indexed blacklistRegistry);
_Chainport_Middleware_V3_init() → chainportCongress_, maintainersRegistry_; setChainportCongress, setMaintainers Registry, setBlacklistRegistry, _setChainportCongress, _setMaintainersRegistry, _setBlacklistRegistry → all have parameters of the form value_.

**Recommendation**

Align variable naming with the style guide, ensuring that parameters are named clearly without prefix or postfix underscores unless indicating scope as per the style guide.

**Re-audit comment**

Acknowledged.

Post-audit.
In terms of variable naming inconsistencies, the team is looking forward to sorting this out in the near future with part of the inconsistencies resolved via: https://github.com/chainport/ smart-contracts/commit/8fe461c2c8de5982ad9b048dad712b58d3647d0a

### Violations of Solidity Style Guide Conventions - magic numbers.
**Description**

2. Magic numbers usage. Hard-coded constants like:
a) ChainportSideBridge: setArbitrage ManagerAndLimit $()\rightarrow10000$ preMint Threshold() → 10000;
b) ChainportMainBridge: setArbitrage ManagerAndLimit() → 10000, 10000;
preRelease Threshold() → 10000, 10000.
are not self-explanatory or defined as named constants, making it difficult to understand their meaning or how they were derived.

**Recommendation**

Replace magic numbers with named constants that provide context and make the code more maintainable and understandable.

**Re-audit comment**

Resolved

### Violations of Solidity Style Guide Conventions - typos.
**Description**

3. A typo in the case of a reverted transaction
ChainportMain Bridge:closePreRelease() has require that checks amount for a specific nonce is correct and in case of falling this statement error says: "Amount is correct" which is opposite and can cause misunderstanding.

**Recommendation**

Make sure that the reverted message is correct.

**Re-audit comment**

Resolved.

Post-audit.
In terms of variable naming inconsistencies, the team is looking forward to sorting this out in the near future. In terms of magic numbers usage, magic numbers were replaced with named constants. In terms of typo, the reverted message was fixed.

### Use of Zero Address in Blacklist Check.
**Description**

ChainportSideBridge: _burnTokens().
The function_burnTokens() uses the modifier only Non BlacklistedAddress(address $(\emptyset\times\emptyset))$, passing the zero address rather than the transaction initiator msg.sender. This could lead to incorrect behavior of blacklist enforcement.

**Recommendation**

Verify the use of only NonBlacklistedAddress in_burnTokens() to ensure it correctly enforcing blacklist controls on the correct participant in the transaction.

**Re-audit comment**

Verified.

Post-audit.
The team verified that enforcement is correct as the mentioned modifier by default takes into consideration both tx.origin and msg.sender; the following argument is for an additional address to check, and in case of 'burn,', there is no recipient or other address to check the blacklist status.

### Dynamic Gas Fees Adjustment.
**Description**

The contracts reference a set gas fee value, which may become outdated as network gas prices fluctuate. This could lead to the bridge not covering its operational costs if the predefined fees are lower than the actual gas prices needed for transactions.

**Recommendation**

Verify if the current method and frequency for updating gas fees (gasFees PerNetwork [networkId]) is effective.

**Re-audit comment**

Verified.

Post-audit.
The Chainport team verified this is an expected behavior. Still the auditors team raises a warning that such an approach may lead to losses in operational costs.

### Backend Dependency and Centralization Concerns.
**Description**

The overall functionality of the bridge heavily relies on backend processes, even though numerous checks are present within the contract, equating to a centralized bridge operation. It is worth discussing the extent to which centralized components are used and the potential implications for the system's decentralization and security stance. The issue is marked as info, referring to the business logic decision and core architecture solution. However, it should be listed in the report due to the potential risk it may pose.

**Recommendation**

Verify the role of backend services within the bridge infrastructure and evaluate the necessity of each centralized component.

**Re-audit comment**

Verified.

Post-audit.
The Chainport team verified that this is an expected behavior. Still, the auditor's team warns that such an approach may create a risk of a single point of failure for the protocol.

### Unclear Commission Parameters Source.
**Description**

ChainportSideBridge, ChainportMainBridge.
It has been observed that commission fees are applied within various functions. However, the source and calculation method for these commission values is not explicitly defined within the contracts. It appears that the backend system may be inputting these values. This lack of clarity could result in transparency issues and potential inconsistencies in fee applications. The issue is marked as info, referring to the business logic decision and core architecture solution. However, it should be listed in the report due to the potential risk it may pose.

**Recommendation**

Verify how commission values are derived.

**Re-audit comment**

Verified.

Post-audit.
The Chainport team verified that this is an expected behavior. Still, the auditor's team warns that such an approach may create a risk of a single point of failure for the protocol because of dependency on another component.

### Potential Overreach of Modifier.
**Description**

1. ChainportSideBridge: _cross Chain Transfer.
The ChainportSideBridge contract's function_crossChainTransfer employs the modifier isPathNotPaused (bridge Token, "crossChainTransfer") to check whether a transfer path is currently paused. However, this modifier is used as a precondition for both crossChainTransfer` and `nonEVMCrossChainTransfer functions. If the crossChainTransfer pathway is paused, it inadvertently impacts the `nonEVMCrossChainTransfer functionality as well, given that the same 'bridgeToken' is used in both functions. This could unintentionally restrict token transfers to non-EVM chains.
2. ChainportMainBridge: _deposit Tokens.
The ChainportMainBridge contract's function_depositTokens employs the modifier isPathNotPaused (token, "deposit Tokens") to check whether a transfer path is currently paused. However, this modifier is used as a precondition for both `deposit Tokens and nonEVMDeposit Tokens functions. If the 'deposit Tokens' pathway is paused, it inadvertently impacts the 'nonEVMDeposit Tokens` functionality as well, given that the same token' is used in both functions. This could unintentionally restrict token transfers to non- EVM chains.

**Recommendation**

Verify the intention behind the shared pause functionality. If these functions should indeed have independent pause controls, consider introducing separate pause states for deposit Tokens` and `nonEVMDeposit Tokens`.

**Re-audit comment**

Verified.

Post-audit.
The team verified that the behavior of these modifiers is intentional.

### Potential withdrawal gas fee to zero address.
**Description**

ChainportMainBridge: withdraw GasFee.
The ChainMain Bridge contract's function withdraw Gas Fee implements the withdrawal of the whole balance of the contract to the fee collector address. The function 'setFeeCollector' has no requirement for checking that the new 'feeCollector` can not be a zero address. However, there is no check that the 'feeCollector address is set, and it may cause a case when the whole balance could be sent to a zero address.

**Recommendation**

Add check that the 'feeCollector address is set to non-zero address to prevent withdrawal to zero address.

**Re-audit comment**

Resolved.

Post-audit.
Check was added along with gas optimization.

### Unused Functionality.
**Description**

ChainportMiddlewareV3: modifiers only Maintainer(), only NotBlacklisted(), onlyNotBlacklistedWithUser().
The ArbitrageManager contract inherits from ChainportMiddlewareV3, which contains certain functionalities and modifiers. It has been observed that not all inherited capabilities of ChainportMiddlewareV3 are utilized within the Arbitrage Manager. Given that ChainportMiddlewareV3 is solely inherited by ArbitrageManager and no other contracts, this leads to the presence of redundant code paths and inherited properties that are not applicable to the Arbitrage Manager's operations.

**Recommendation**

Verify the necessity of these modifiers for Arbitrage Manager. If certain parts of ChainportMiddlewareV3 are not needed by ArbitrageManager, consider refactoring to enhance clarity.

**Re-audit comment**

Verified.

Post-audit.
Team verified that described modifiers might be used in the future by Arbitrage Manager and will either way be used by the other contracts which will inherit Chainport MiddlewareV3.
