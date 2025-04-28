**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Owner is able to withdraw reward tokens.

**Description**

Bribes.sol: function recoverERC20().
Owner is able to withdraw any tokens from the contract, including reward tokens, which
might lead to errors in reward distribution. In case, owner is set to a DAO contract, this can
potentially be used by malicious actors.

**Recommendation**:

Validate that reward tokens cannot be withdrawn.

### depositFor() transfers tokens from target account instead of msg.sender.

**Description**

Gauge.sol: function depositFor().
Function is supposed to stake the amount of tokens for “account” by sender of transaction,
however in function _deposit() funds are transferred from “account” instead of msg.sender
which prevents function from correct execution. In case, “account” had approved tokens but
never deposited, his funds will be deposited without his knowing.

**Recommendation**:

Transfer funds from msg.sender instead of “account”.

**Post-audit**.

Client assured, that they know about this functionality and that it is purposely designed to
deposit the balance of account in case, account has approved his tokens.

### Deprecated Eth transfer

**Description**

GaugeProxy.sol: function withdrawFee().
Due to the Istanbul update there were several changes provided to the EVM, which made
.transfer() and .send() methods deprecated for the ETH transfer. Thus it is highly
recommended to use .call() functionality with mandatory result check, or the built-in
functionality of the Address contract from OpenZeppelin library.

**Recommendation**:

Correct ETH sending functionality.

**Post-audit**.

Functionality of withdrawing ETH was removed from contract as well as ability of contract to
collect ETH.

## Medium Risk

### Unused mapping.

**Description**

GaugeProxy.sol: mapping deprecated.
Values from mapping “deprecated” are never read or written.This might signalize about
unfinished contract logic. Contract should not contain unused variables or unfinished logic.

**Recommendation**:

Either remove unused mapping or implement logic, based on it.

**Post-audit**.

Mapping was removed.

### Unit-tests fail due to changes.

**Description**

GaugeProxyBribeMiddleManTesting.js: before() hook.
GaugeProxyDistributeTesting.js: before() hook, it('Admin functions only, all tests to revert) test.
GaugeProxyBribeTesting.js: before() hook, it('Add a new Gauge for LP4') test.
Call to function addGauge() from GaugeProxy.sol fails due to change of parameters.

**Recommendation**:

Pass the correct parameters to function call.

## Low Risk

### Function exit() always reverts.

**Description**

Gauge.sol: function exit().
Functions exit() calls nonReentrant modifier multiple times(in functions exit(), _withdraw(),
getReward()). nonReentrant is designed in such a way, that it must be called only once during
a transaction otherwise it reverts. Issue is marked as low, since it is still possible to call
withdraw() and getReward() separatle, however function exit() remains invalid.

**Recommendation**:

Call nonReentrant modifier only one time during a function execution.

**Post-audit**.

NonReentrant modifier was removed from function exit(), however it still remains on both
functions _withdraw() and getReward(). The recommendation would be to move the logic of
getReward() to private function without NonReentrant modifier and call it instead.

**Post-audit.**

Function was removed.

### Function is not restricted.

**Description**

BribeFactory.sol: function createBribe().
Function createBribe() deploys a new instance of Bribe.sol, where the second parameter is the
sender of the transaction. In the constructor of Bribe.sol, the second parameter is supposed
to be GaugeProxy.sol, however, there is no validation that the caller of createBribe() is
GaugeProxy.sol. Issue is marked as low, since it is not yet clear what impact an issue might
cause the protocol and whether function is supposed to be restricted.

**Recommendation**:

Either confirm, that the function should not be restricted or validate, that the caller of function
is verified contract GaugeProxy.sol.

**Post-audit**.

Client verified, that function should not be restricted.

### SafeMath usage can be omitted.

**Description**

Contracts set utilizes Solidity 0.8.x, which has built in support of overflow and underflow
warnings, thus SafeMath library can be omitted for gas savings and code simplification.

**Recommendation**:

Remove SafeMath library.

## Informational

### Outdated Solidity version.

**Description**

Currently the protocol utilizes Solidity version 0.5.17, 0.6.7, . It works well, though the general
auditors security checklist recommends to utilize the newest stable version of Solidity which is
0.8.11. The newest version of Solidity includes last optimization for the compiler, last bugfixes
and last features such as built-in overflow and underflow validations.

**Recommendation**:

Update Solidity version to the latest version 0.8.11.

### Emit event when creating a new bribe.

**Description**

BribeFactory.sol: function createBribe().
In order to keep track of all created bribes, emit an event, so that an address can always be
recovered.

**Recommendation**:

Emit event with an address of newly created bribe.

**Post-audit**.

Client verified that there is no need to emit an event.
