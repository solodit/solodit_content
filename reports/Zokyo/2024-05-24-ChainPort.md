**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Bridge can appear to be out of funds of tokens managed by the liquidity manager

**Severity**: High

**Status**: Acknowledged 

**Description**

ChainportMainBridge.sol, deposit / release flow
In current iteration, the main bridge adopted the Liquidity+anger as an additional entity for
holding the liquidity. In case if the token project is considered as managed (is listed on the
Liquidity+anager), all deposited liquidity is transferred to the Liquidity+anager. &nd vice
versa - during the release it is requested from the Liquidity+anager. However, currently the
project has 2 scenarios, when the liquidity will become unavailable (or partly unavailable) for
the withdraw
1) Token project is manually stopped from being manage
     - it is possible in case if liquidity is withdrawn from the Liquidity+anger contract (by the authority or project owner) and the owner is removed
2) Liquidity is unavailable - in case of being borrowed by the arbitrageManager
In both cases, there is no trigger to return the necessary amount of tokens to the bridge.
&lso, since there is no track of the liquidity deposited from the bridge and validation of the
amount borrowed by the ArbitrageManager against this track, the moment of the balancing
liquidity back can be missed
**Recommendation**:
Verify the current logic and provide the necessary sanitizing actions:
- add tracking of funds deposited from the bridge
- add validation of the funds borrowed by the &rbitrage+anager against the deposited amount
- leave some liquidity on the manager contract for the release from the bridge
- add triggers for returning the liquidity for bridge depositors in case of its withdraw from the liquidity manager by the authority/ownex
- check that owner removal is impossible in case if liquidity was borrowed to the
ArbitrageManager



**Post-audit**:
The Hord team verified that all mentioned checks are performed by the protocol backend. It
tracks the balances and supplies sinatures to:
- track the amount of funds borrowed by the ArbitrageManager and ensure no more than
the allowed amount is being borrowed.
- validate the liquidity left on the manager contract aainst the minThreshold parameter
durin release/mint events and ensure that there always is available liquidity.
- ensure that both arbitrae operator and project owner cannot withdraw funds that are
currently in flux and should be delivered to a portin user.

The Hord team commented that all actions (token movements) are stored onchain toether
with appropriate sined action from the responsible backend service, thus the full history of
actions can be retrieved. The whole process of cross chain arbitrage is packed into the
atomic arbitrae executions, thus the liquidity cannot be resolved in a sinle contract, but
only at both ends toether with the appropriate sined action from the backend worker&
Still, the security team leaves a concern reardin hih dependency on the backend service
- in terms of deleation of most of the crucial checks.


### Restricted setter open for unauthorized users.

**Severity**: High

**Status**: Resolved 

**Description**

ChainportMainBride.sol
ChainportSideBride.sol
`setAddressLength()` setter can be called by anyone. The `onlyClassifiedAuthority` modifier
was deleted durin the code uprade.

**Recommendation**:

Set the appropriate restricting modifier to the function.

### Signature replay during the release/deposit operations on the LiquidityManager.

**Severity**: High

**Status**: Resolved 

**Description**

LiquidityManager.sol: preRelease(), depositToClosePreRelease()

1) In the preRelease() operation: nonce is not a part of the signature and is not validated to
be used, thus it can be re-used. And in that case the amount in `amountsPerNonce` is
overridden, making it unreliable in terms of the amount to return.

2) in the `depositToClosePreRelease()` operation: the nonce is not checked to be used
(isArbitrageNonceUsed is not validated), making its replay possible

3) Looks like nonces from preRelease() and depositToClose are mismatched:
- `amountsPerNonce` is only set in `preRelease()` and never validated/decreased in
depositToClose
- `isArbitrageNonceUsed` is set in `depositToClose`, but never validated and never related to
preRelease record bound to nonce.

The issue is marked as High, but not Critical, as despite the unreliable records and replayed
nonces, the total amount is still manually controlled by the Operator (in terms of how many
to release/deposit) and Authorities (in terms of liquidity withdraw from the manager).

**Recommendation**:

Match the nonce usage:
- include nonce into the signature in preRelease()
- in preRelease() validate amountsPerNonce against the nonce re-usage OR avoid the
overriding by increasing the amount
- in depositToClose validate nonce against the amountsPerNonce amount to decrease it OR
validate nonce against re-usage
- verify if release and deposit nonce are independent or they represent the same entity
opened in release and closed in deposit - and match both operations for it.
**Post-audit:**
The Chainport team provided necessary nonce validations and verified, as the nonce is an
identifier of the crosschain operation, the backend service monitors it to be used in that way
(deposited on one side and released on another side within the same nonce). Since
operations are on different sides of the bridge, nonce is validated offchain (in terms of
matching both ends). Still, auditors highly recommend to make a part of the signature.

## Medium Risk

### Arbitrage manager can return unlisted tokens to the liquidity manager
ArbitrageManager.sol: _depositOrBurn()
LiquidityManger.sol: depositToClosePreRelease()

**Severity**: Medium

**Status**: Resolved 

**Description**

Arbitrage manager can send any token (based only on the operator decision) to the liquidity
manager, omitting the check if the project is managed. Such flow may cause tokens
prepared for the Bridge to be transferred to the LiquidityManager, or it may cause the
LiquidityManager to receive tokens it does not support.

**Recommendation**:

Add validations that the token is actually managed and expected on the LiquidityManager

**Post audit.**

LiquidityManager checks that Arbitrage manager returns managed tokens only (via the call
of depositToClosePreRelease())

### Authorized function is never used

**Severity**: Medium

**Status**: Resolved 

**Description**

LiquidityManger.sol: resetTokenStateByArbManager ()

The function is never used, though it is supposed to be called from the `ArbitrageManager`
contract

**Recommendation**:

Verify and correct the logic

**Post audit.**

The Chainport team verified, that it is a function to be called periodically to reset the state of the arb manager withdrawals and deposits, and that it should be called by the backend or
congress. The access control was changed from the arbManager to Authorized.

### Airdrop pool can be drawn by a maintainer.


**Severity**: Medium

**Status**: Resolved 

**Description**

ChainportMainBridge.sol, releaseTokens()
ChainportSideBridge.sol, mintTokens()

The caller can set any airdropAmount up to the max available, drawing all available liquidity.
Despite that the airdrop funds are considered extra and may not be present at the contract
at all, the user still has the ability to withdraw more than expected.

**Recommendation:**

Include airdropAmount into the signature payload or provide the max airdrop value which
can be withdrawn.

**Post audit**

airdropAmount parameter is now included into the signature

### No change returned in case of extra fee value

**Severity**: Low

**Status**: Acknowledged 

**Description**

ChainportMainBridge.sol: _depositTokens()
ChainportSideBridge.sol: burnTokens()

No change returned from the msg.value in case if deposited more than the needed. Since
the check is not strict, it allows such operation and the user may leave more funds than
necessary.

**Recommendation**:

Verify the approach and provide clear documentation and warning for the user about such
risk OR provide the strict equality against the necessary amount of fee OR return the change
back to the user

**Post audit.**

The Chainport team verified it as intended behavior - as a measure of avoiding the risk of
rounding errors issues (at least on one side) between frontend and backend. Though, the
team ensured that the frontend of Chainport makes sure the values are exact, and any
integration partner will also receive the exact values to send in msg.value from the Chainport
api endpoints as per the docs.

### Signature replay with the same fee value

**Severity**: Low

**Status**: Acknowledged 

**Description**

ChainportMainBridge.sol, deposit functions
ChainportSideBridge.sol, burnTokens()
Currently, the protocol delegated the fee calculation to the offchain logic, therefore the ,ee
is considered valid in case it is signed by the validator. However, the process does not
include the nonce, so the same signature
once signed - the change o, ,ee will not take e,,ect until the deadline, as the user will be
able to re-use the same signature.
There,ore there are 2 risks:

- replay of the same signature - though limited to the re-use o, the same gas ,ee (and
same parameters
- fee validation delegated offchain - thus open to the human error
  
Issue is marked as Low, as although the signature replay is considered as high-risk
vulnerability, in this case the impact is low and limited in time (until the deadline as part o,
the signature), thus the overall criticality can be considered as low.

**Recommendation:**

Verify the functionality, or add the nonce validation for each operation, and/or provide the
onchain fee validation - e.g. for the minimum/maximum boundaries

**Post audit.**

The Chainport team verified, that the signature mechanism currently takes care of
issues:

1) checking address against the blocklist
2) charging proper gas fees for the target chain tx of the port.

As the time to live o, the sig will be a ,ew minutes, the Chainport team accepts the risk o,
gas price ,luctuations and possible short time replay, assuming gas prices on target chain
and blocklist status o, the address won't change in these ,ew minutes. So in such case, it's
ok ,or the user to reuse the signature to port multiple times with the same params.
Since the potential impact is Low, auditors are ok with accepting the risk, though still ,ind it
worth mentioning in the report.

### Missing case for signatures validation.

**Severity**: Low

**Status**: Resolved 

**Description**

Validator.sol, verifySignature()

The function recovers the signer from the signature and compares it with the signatory
address. However, the signaioryAddress is not validated during the deployment and may
appear to be zero address, allowing invalid signatures.

The issue is marked as Low, as auditors recognize, that the functionality is expected to be
used during the upgrade of current valid contracts. However the issue may appear during
the redeployments on other networks or protocol versions.

**Recommendation:**

Add the check that the signatoryAddress is not zero address (either into the initializer or into
the function)

**Post audit.**

The Chainport team verified, that the deployment procedure include checksum scripts which
ensure the signatory was properly updated. Though the check was added into the initialized.

## Informational

### Lack of events for settings changes

**Severity**: Informational

**Status**: Acknowledged 

**Description**

ChainportMainBridge. setLiquidityManager()
The liquidity manager plays a crucial part in the logic of the contract. Therefore, it is
recommended to keep track of all addresses that have been set for the liquidity manager.

**Recommendation:**

Add event emitting to the function adopting the approach from Main and Side bridges
contracts.
**Post audit.**
The Chainport team is aware of events' absence, though events are not added because of
the bytesize restrictions. Since the info can be retrieved from the tx itself, it is acceptable for this setter.

### Unreachable check

**Severity**: Informational

**Status**: Resolved 

**Description**

LiquidityManager.updateProjectOwner()

The modifier `_updateProjectOwnerModifier` will fail before the check for token against the
zero address (while getting balance of the token). Therefore, the check for zero address will
never be reached.

**Recommendation:**

Consider implementation of the modifiers with zero address check with the approach
adopted in `MainBridge` and `SideBridge` contracts - making the correct order of checks (zero
address as the first one, project owner as the second one).

### Unused parameter

**Severity**: Informational

**Status**: Resolved 

**Description**

ChainportMainBridge.sol, deposit functions

ChainportSideBridge.sol, burnTokens(), cross chain transfer functions
Parameter targetAirdrop is not used except as in the event. While it is a common approach to
have some information stamped onchain without storing (adding it to the event) - there are
no documentation or explanation of the parameter. Therefore - except the extra gas
spending - there is a risk of some functionality to be missing or of misbehavior. Since there
are no validations of the parameter, and it is not a part of the signature - the user can set
any value and impact the dApp logic (in case if the parameter is traceable offchain).
The issue is marked as Info, as currently it cannot be classified because of the unknown
impact on the offchain logic.

**Recommendation:**

Verify the parameter usage and add the natspec documentation with the info about the
parameter.
**Post audit.**
The Chainport team verified the parameter to be a part of preparation for the functionality to
allow users to specify their own preference of how much target chain gas token to receive
(And respectively they would pay for it in the source chain gas token) - planned to the next
protocol updates. For now the parameter is not used, though it is already added to the
verification process with the signature.

### Removed safety check

**Severity**: Informational

**Status**: Acknowledged 

**Description**

ChainportMainBridge, _depositTokens()

The check against the inflated tokens (that the final balance does not exceed the expected)
was removed.
The issue is marked as Info, as currently its impact cannot be validated without the
feedback from the Chainport team. Since it provides another layer of the responsibility on
the offchain elements (in terms of sanitizing measures against the listed tokens) such
security delegation may impact the protocol.

**Recommendation:**

Verify the necessity of the check, verify the presence of the sanitizing actions at the offchain
part of the protocol, provide the strategy of tokens allowlisting (in terms of support for fees
on transfer, burn on transfer, buyback on transfer, inflation on transfer).
**Post audit.**
The Chainport team verified, that the protocol works with verified tokens only. The allowed
tokens are maintained by the signature (which will only be delivered to porters of verified
tokens). Nevertheless, this check was moved to the backend and it still won't deliver ports if
the actual amount received on the bridge is different than the amount declared.


### Parameter to be included into the event

**Severity**: Informational

**Status**: Acknowledged 

**Description**

ArbitrageManager, releaseOrMintAndSwap()
Add the operation origin (onLiquidityManager parameter) into the event. Currently the event
tracks the release/mint operations parameter, thus it seems that the liquidity manager
origination should be tracked as well to fully track the operators choice.
The issue is marked as Info, as it refers to the business logic decision and cannot be
classified at this moment.

**Recommendation:**

Add the onLiquidityManager parameter to the event.

**Post audit.**

The Chainport team verified, that the parameter cannot be added because of compiler
restrictions, though the info is retrievable from the tx itself, founds by nonce.

### Optimizations, best practices violations, style issues

**Severity**: Informational

**Status**: Resolved 

**Description**

1) ChainportMainBridge.sol, ChainportSideBridge.sol, withdrawGas()
Consider splitting the function into two separate to avoid double meaning and split
operations for separate fees (gas fee and airdrop funds)
2) ChainportMainBridge.sol, ChainportSideBridge.sol, withdrawGas()
feeCollector is validated against 0 address, but not the gasFeeAirdropReceiver
3) ChainportMainBridge.sol, setLiquidityManager(), setSignatureValidator(),
setBlacklistRegistry()
ChainportSideBridge(), mintNewTokenFromNonEVM()
Use the modifier for validation against the zero address
4) ChainportMainBridge.sol, releaseTokens()
Use _sendValue() for the airdrop sending
5) ChainportMainBridge.sol
isZeroAddress() is not used and can be removed from the contract
6) ChainportMainBridge.sol, ChainportSideBridge.sol, withdrawGas()
Validate that gasGee is > 0 to avoid gas spendings on 0 value. Or use min DUST constant.
7) ChainportSideBridge.sol, _burnTokens()
Remove the commented code
8) LiquidityManager.sol, setArbitrageManager()
Add check against the zero address
9) LiquidityManager.sol, resetTokenStateByArbManager()
Save the length of the array of the tokens as the variable before the loop `for` and use this
variable as check in loop.


**Recommendation:**

Proceed with optimizations and following best practices
**Post audit.**
The Chainport team resolved most of items. items 1, 3 and 6 were omitted because of the
bytecode restrictions.
