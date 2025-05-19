**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Contracts’ code size exceeds limit.

**Description**

Contracts: CashVaultV1.sol, CashSecuredLeveragedPutVaultV1.sol,
CashSecuredPutVaultV1.sol, CondorVaultV1.sol, CoveredCallVaultV1.sol.
Contracts’ code size exceeds the limit(24576 bytes, a limit introduced in Spurious Dragon),
thus they cannot be deployed to most chains.

**Recommendation**:

Reduce contracts’ code size.

**Post-audit**:

After the consultation with the Client marked as not an issue. The Client uses the
configuration for compilation with optimizer enabled (200 runs), with solc
0.8.9+commit.e5eed63a, evm_version istanbul, and with a bytecode size of ~16Kbytes.
Also, the Client claimed to have the successfully deployed code.

### Owner is able to withdraw any token balance without restrictions.

**Description**

CommonV1.sol
IndexVaultV1.sol
PutPhyVaultV1.sol
function emergencyWithdraw().
Owner of the contract should not have direct access to users’ funds, such backdoors are
considered unsecured.

**Recommendation**:

Validate that the owner cannot withdraw all funds to himself at any time.

**Post-audit**:

After the consultation with the Client marked as not an issue as it is a deliberate design:
emergencyWithdraw can only be called by the Owner, which is assigned to an OwnerProxy,
which only allows multisigAlpha to have the ability to call multisigAlpha keys will be highest
tier, held by investors and members of community. this will be eventually delegated to a
timelock/governance combo.

### User’s funds may get stuck.

**Description**

CommonV1.sol: function initWithdrawOnBehalf().
In case, user has queued LP tokens to withdraw in epoch 0, he is no longer able to withdraw
these funds(due to assertions in CommonV1.sol, line 117 and CashVaultV1.sol, line 66), until
he queues another withdrawal in next epoch.

**Recommendation**:

Verify, that users who queue their LP token in epoch 0 would be able to withdraw them.

**Post-audit**:

Was verified after the consultation with the Client. When expiry == 0, initWithdraw(amt) leads
to L141, directWithdraw(msg.sender). As expiry cannot be > 0 in epoch 0, when a user
initWithdraw in epoch 0, it will trigger call directWithdraw.

This is corroborated by unit tests,
Scenario 4 - Multiple deposits, and immediate exit: test_double_deposit_and_early_withdrawal.

### Queued LP tokens cannot be withdrawn due to subtraction underflow

**Description**

CashVaultV1.sol: withdraw()
In case user has queued LP tokens to be withdrawn later(this happens if ‘expiry’ is not equal
0), he is not able to withdraw his funds due to underflow in function withdraw() (line 75
“totalPendingExit -= exitAmt”). This happens because “totalPendingExit” is updated in function
directWithdraw(), but this function is not executed if LP tokens are queued to be withdrawn
later.

**Recommendation**:

Update “totalPendingExit” for queued LP tokens to exit, so that subtraction doesn’t revert.

**Post-audit**:

Was verified after the consultation with the Client.

totalPendingExit is updated at 2 points:
1) When initWithdraw is handled at expiry > 0, totalPendingExit is updated at _settleStrike (e.g.
CashVaultV1.sol, L149).
2) When initWithdraw is handled at expiry == 0, this triggers the mentioned L75 update for
directWithdraw -> totalPendingExit.

### Functions withdraw() and directWithdraw() called from initWithdrawOnBehalf()

**Description**

withdraw funds for the wrong account.
CommonV1.sol: function initWithdrawOnBehalf().
In case function initWithdrawOnBehalf() is called by owner for 'onBehalfOf', functions
withdraw() and directWithdraw() will withdraw for msg.sender, which is owner address instead
of address 'onBehalfOf'.

**Recommendation**:

Process withdraw() and directWithdraw() for right address in case functions are called from
initWithdrawOnBehalf().

**Post-audit**.

Main issue was resolved, but auditors recommend to verify that reading balance and burning
LP tokens should also be performed with 'onBehalfOf' instead of msg.sender.(Lines 130, 137,
143). Though the Client has verified that initWithdraw is meant to burn balance from the
contract/EOA calling it

### Anyone can initiate withdrawal on behalf of

**Description**

CommonV1.sol: function initWithdrawOnBehalf().
There is no validation that the caller of the function is the owner, in case the sender of the
transaction is not 'onBehalfOf'. The comments to the function say that only the owner can
initiate withdrawal.

**Recommendation**:

Add a validation, that the caller is the owner, in case the sender of the transaction is not
'onBehalfOf'.

**Post-audit:**
Was verified after the consultation with the Client. initWithdrawOnBehalf is meant to burn
balance from the user calling it

### Function might revert due to underflow.

**Description**

In case, function _settleStrike() is called during epoch 0, function will revert with underflow in
line 136 in 'epoch - 1'.

**Recommendation**:

Validate that arithmetic operation won’t revert due to overflow/underflow.

**Post-audit**:

Was verified after the consultation with the Client to be working as intended. _settleStrike will
always revert in epoch 0, as expiry will always be 0 in epoch 0, and will revert on require(expiry
> 0) check.

### Assertion always fails in case there are lent tokens.

**Description**

CommonV1.sol, modifier checkInvariant().
“Require” in line 90 will always fail in case there are tokens, sent to Aave, since balance cannot
be greater, than “vaultCollatSize + collatLentOut”

**Recommendation**:

Verify that the modifier works as intended and doesn’t prevent operation of the vault.

### Incorrect amount of LP tokens might be stored.

**Description**

IndexVaultV1.sol, function rebalance().
After depositing collateral tokens to target vault, an amount of collateral is added to vault’s
amount instead of received LP tokens(line 145). It is possible, that the ratio between target
vault LP and collateral isn’t 1:1, thus an incorrect amount of vault’s LP tokens will be tracked,
which might lead to incorrect operation of the vault.

**Recommendation**:

Add actual amount of received LP tokens instead of received collateral amount.

### Assets, sent to earning, can’t be withdrawn.

**Description**

PutPhyVaultV1.sol
There is no interface for withdrawing asset from earning in the vault. Once, asset is sent to
Earning with functions sendBaseCollatToVault() or sendQuoteCollatToVault(), the user is not
able to withdraw them, and funds cannot be withdrawn from earning.
The state of vault cannot be switched from EARNING to OPEN, thus user is not able to use
initWithdraw(), when funds are sent to earning.
Once, the quote asset is sent to earning with function sendBaseCollatToVault(), the state of
vault is switched to EARNING, thus functions such as sendQuoteCollatToVault(),
initNewRound(), initWithdraw() cannot be performed.

**Recommendation**:

Add an interface to withdraw assets from earning and switch vault state from EARNING.

**Post-audit**:

Was verified after the consultation with the Client to be working as intended. Once in
EARNING mode, there's no need to initWithdraw, as the vault will no longer be selling Put
options. The user will be prompted to call withdraw(), and would enter L212, then enter L227,
where the code will use vaultCollat to transfer the assets to the user.

### Not all amounts of quote asset or base asset may be sent to earning.

**Description**

PutPhyVaultV1.sol, functions sendBaseCollatToVault(), sendQuoteCollatToVault()
Transfer amount is calculated based on totalSupply of vault(lines 331, 348). However, in case a
user has queued part of LP tokens, this amount is burnt, thus calculations based on total
supply are not accurate.

**Scenario**:
1) User deposits 10 tokens.
2) New round is inited.
3) User calls initWithdraw() an queues 5 tokens.
4) strike is settled and quote asset is sent to earning(5 tokens only)
5) User call witdraw(). Vault doesn't send quote asset(Line 203) because condition in lines
189-190 is False.
Then Vault LP tokens, queued to exit, are minted again(line 213), exitAmt is calculated baed on
restored balance and vault is trying to send vault collateral LP tokens(line 226), but reverts
because PutPhyVault has insufficient balance of collateral vault balance, because 5 tokens
were sent to earning and 5 tokens were left on vault balance.

**Recommendation**:

Calculate transfer amount, based on exit lp as well.

### Incorrect interface of vault.

**Description**

OwnerProxy_V1.sol: functions emergencyWithdraw(), settleStrike_MM().
function CommonV1.emergencyWithdraw() has a parameter of token address, but owner
proxy doesn’t pass it(Line 114).
There is no storage variable or function with name “COLAT_ADDRESS” in vault contracts(Line
115, 177) or “PRICE_FEED”(line 148).

**Recommendation**:
Use the correct interface of vault contracts.

## Medium Risk

### Queued lp to exit might not be restored.

**Description**

In case, user has queued his LP to exit for current epoch, it is also tracked in variable
'totalLPExit. After that settleStrike_chainlink_fallback() can be called by anyone, this function
calls _settleStrike() and in CashVaultV1._settleStrike(), 'totalLPExit' is set to zero. After this, in
case the user calls initWithdrawOnBehalf() with 0, in line 126 function call will revert with
underflow.

**Recommendation**:

Validate, that user is still able to restore LP after settleStrike_chainlink_fallback() since such
functionality is provided.

**Post-audit**:
Was verified after the consultation with the Client to be working as intended. Once
_settleStrike is called, the next call to initWithdrawOnBehalf triggers withdrawOnBehalf first,
which sets state and memory of userQueuedExitLP to 0. Thus, totalLPExit -=
userQueuedExitLP (which is 0) will not revert after settleStrike.

### Signed init new round might fail.

**Description**

CommonV1.sol, function initNewRoundSigned().
CashVaultV1.sol, function initNewRound().
Execution of initNewRoundSigned() might fail in function CashVaultV1.initNewRound() in line
124(When checking that signedExpiry equals expiry) in case settleStrike_chainlink() was
executed in line 105 and expiry was reset to 0. Passing “_signedExpiry” as 0 in order to pass
the validation in this case laso fails in CommonV1.initNewRoundSigned(), line 159.

**Recommendation**:

Validate “signedExpiry” with actual “expiry” value. Issue can also occur in
SynMiningVaultV1.sol(line 170), because validation is performed before expiry is updated.

### Use ECDSA library.

**Description**


CommonV1.sol, function initNewRoundSigned().
Current verification of signature did not work as intended during testing with Hardhat, ethers
and web3. Function executing failed on decoding r, s, v parameters from signature.

**Recommendation**:

Use ECDSA from standard openzeppelin library for correct signature validation. An example of
ECDSA usage:
![image](https://github.com/user-attachments/assets/3b791bd3-313c-483c-97d2-09256a223935)

### Possible Denial-of-Service attack

**Description**

OwnerProxy_V1.sol: function depositOnBehalf().
Function deposits provided an amount of tokens to the vault, which are transferred from the
caller of function(Line 178). At the end of function, balance is checked to be zero.
In case, some amount of tokens were transferred before on the ownerProxy contract(even a
small amount like 1 wei) balance, validation in line 181 will always fail, which would cause
denial of service. The only way to empty the balance of the contract is to call
emergencyWithdraw().

**Recommendation**:

Either deposit the whole balance of contract or remove the validation in line 181.


## Low Risk

### Check that the function parameter is not zero address.

**Description**

PutPhyVaultV1.sol, function setOwner(). Parameter ‘_newOwner’ should be validated not be a
zero address.

**Recommendation**:

Add validation that the parameter is not zero address.

### Wrong asset parameter passed to lending.

**Description**

IndexVaultV1.sol, function depositIntoLendingPool(), withdrawFromLendingPool().
Based on the logic of contract, collateral assets are stored on sub vaults. There are only LP
tokens of vaults on index vault’s balance, and in order to deposit them to lending, they need
to be listed on Aave first.

**Recommendation**:

Verify the correction of functions which interact with lending protocol and which asset is
deposited.

**Post-audit**:
Was verified after the consultation with the Client to be working as intended.

## Informational

### Emit events in setter functions.

**Description**

OwnerProxy_V1.sol: setMultisigAlpha(), setMultisigBeta(), setTeamKey().
Such setter function should emit an event for saving storage changes history.

**Recommendation**:

Emit event in setter function.

### Validate ‘startTime’ parameter in constructor.

**Description**

CashSecuredLeveragedPutVaultV1.sol, CashSecuredPutVaultV1, CondorVaultV1,
CoveredCallVaultV1: constructor().
Parameter ‘startTime’ should be validated to be greater than block.timestamp.

**Recommendation**:

Add validation for constructor parameter.

**Post-audit**:

Was verified after the consultation with the Client to be working as intended.

### Visibility for storage variables is not set.

**Description**

CommonV1.sol: allowInteractions
IndexVaultV1.sol: _name, _symbol
It is recommended to explicitly set visibility for storage variables for better code readability.

**Recommendation**:

Explicitly set visibility for storage variables.

**Post-audit**:

Was verified after the consultation with the Client to be not an issue.

### Commented code

**Description**

PutPhyVaultV1.sol, line 302
SynMiningVaultV1.sol, line 28
Pre-production contracts should not contain unfinished logic or commented code.

**Recommendation**:

Remove commented code.

### “Require” never reverts

**Description**

CashVaultV1.sol: function: function _settleStrike(), line 139.
Function _settleStrike() is called from CommonV1.settleStrike_chainlink() and
CommonV1.settleStrike_chainlink_fallback(), parameter “oracleTime” is assigned to “expiry”
which means that “oracleTime” cannot be lower that “expiry”.

**Recommendation**:

Remove unnecessary require.

### Unnecessary storage update.

**Description**

IndexVaultV1.sol: function removeVault().
During removal of the vault, the storage variable ‘totalWeight’ is being updated, where the
weight of the vault is subtracted from total weight. However, due to restriction in line 93(The
weight of the vault must be 0), subtracting 0 from ‘totalWeight’ is not necessary and only
consumes extra gas.

**Recommendation**:

Remove unnecessary writing to the storage;
