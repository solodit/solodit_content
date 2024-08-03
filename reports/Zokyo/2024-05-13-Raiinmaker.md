**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Withdraw Message Would Be Replayable On A Different Chain


**Severity**: High

**Status**: Resolved 

**Description**

The message digest calculated for the withdraw includes `msgSender`, `amount`, `expires`, `nonce`, and `address(this)`, and then the `withdrawSigner` is recovered. But this message does not include a chainId due to which if the contract gets deployed on another chain then the same message would be valid there with a different `chainId` and the user would essentially withdraw twice.

**Recommendation:**

Add a chainId value to the message digest calculation.

## Low Risk

### Unsafe Transfer

**Severity** - Low

**Status** - Resolved

**Description**

The `rescue()` functionality is used to rescue mistakenly transferred funds to the contract and this function transfers the token to the owner, but the transfer used here is an unsafe one, i.e. the transfer returns a bool for ERC20 implementations and that value is not checked here, therefore the transfer might fail silently and the tx would still be successful.

**Recommendation:**

Use `safeTransfer` instead.

### Centralization Risks

**Severity** - Low

**Status** - Acknowledged

**Description**

All privileged and critical state changing functions are protected with a onlyOwner modifier, it should be made sure that the owner is a multisig with a timelock. Handling all the power to a single address will make the system centralised and prone to failure.

**Recommendation:**

Ensure the owner is a multisig with a timelock with each key residing on a different server.

The client mitigates centralization risks by using multi-signature (multisig) accounts, ensuring that no single user has complete control.

## Informational

### Paused Interval Does Not Account For The `withdrawMaxPeriod`

**Severity** - Informational

**Status** - Acknowledged

**Description**

Withdrawals can be paused by the owner using the `pauseWithrawals()` function, therefore it is possible that during a paused state all user’s withdrawal timeline might go beyond the `withdrawMaxPeriod` and all the user’s mint would be dequeued making the `withdrawMintHistory` empty when the state gets unpaused.

**Recommendation:**
 
Account for the paused state appropriately.


