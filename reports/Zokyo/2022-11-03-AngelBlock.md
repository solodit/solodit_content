**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Exposure to replay attack

Severity: Critical

Status: Resolved

**Description**

CommunityRound.sol, method: reserve(_request, _message, v, r, s), _request along with _message and signature can be replayed. It is assumed that this is not intended since balances[sender_] = _request.amount; which implies that balances are set once, therefore this issue can be critical as it leads to funds being misallocated added to the fact that replay attacks need to be avoided anyway.

**Recommendation**: 

Nonce uint mapping for senders.

### Balances messed up

**Severity**: Critical

**Status**: Resolved

**Description**

CommunityRound.sol, method: reserve(_request, _message, v, r, s), balances of investors are being overwritten rather than adding up balances[sender_] = _request.amount;. Hence, funds of investors are misallocated. This issue is related to the replay attack issue and depending on the intention behind the method this issue might not be relevant if it is intended to have the investor call this method only once.


## Medium Risk

### USDT transfer is not safe

**Severity**: Medium

**Status**: Resolved

**Description**

CommunityRound.sol, method: reserve(_request, _message, v, r, s), it is recommended as a best practice to use SafeERC20 to transfer ERC20 tokens. Failed, unchecked ERC20 transfers might lead to reserving funds for an investor that is not actually deposited at the CommunityRound contract.


### Centralization risk

**Severity**: Medium

**Status**: Resolved

**Description**

Admin enjoys much authority granting roles in all the contracts which leads to enable him to manipulate funds. This can take place by calling withdraw() to take advantage of the asset by transferring all to a wallet of his choosing. Some methods can be more highly severe to be left out controlled by one wallet more than other methods. Recommendation Apply governance / use multisig wallets for certain roles.

## Low Risk

### Solidity version

**Severity**: Low

**Status**: Resolved

**Description**

Lock the pragma to a specific version, since not all the EVM compiler versions support all the features, especially the latest ones which are kind of beta versions, So the intended behavior written in code might not be executed as expected. Locking the pragma helps ensure that contracts do not accidentally get deployed using, for example, the latest compiler, which may have higher risks of undiscovered bugs.

**Recommendation** 

fix version to 0.8.13

## Informational

### Unnecessary gas consumption

**Severity**: Informational

**Status**: Resolved

**Description**

CommunityRound.sol, method: reserve(_request, _message, v, r, s), address of caller is being pushed into investorsList which is a private array of addresses, and it is not being read or used in any occasion. This storage serves no purpose, and it is considered an expensive operation.


### Decimals are not validated

**Severity**: Informational

**Status**: Resolved

**Description**

CommunityRound.sol, method: configure(bytes), address of usdt_ better validate that it refers to ERC20 with decimals equal to 6 since variables MAX_USDT_TO_RESERVE & MIN_USDT_TO_RESERVE presume that this is the decimal value of the asset to be deposited.


8. Address argument not validated
Severity: Informational
Status: Resolved
CommunityRound.sol, method: `configure(bytes)` , the decoded variable `usdt_` is not verified to be non-zero.
Recommendation: require statement on `usdt_` to be non-zero.
 
