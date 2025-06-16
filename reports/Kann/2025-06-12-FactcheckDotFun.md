**Auditor**

[Kann Audits](https://x.com/KannAudits)

# Findings

## Medium Risk
### [M-01] SignatureVerification BrokenUnderEIP-7702DuetoRelianceonisContractLogic

**Severity**

Medium

**Description**

In the FactCheckExchange.sol contract,the settleMatchedOrders() function attempts to
validate orders using signatures by distinguishing EOAs from contracts. It uses the following logic:

-If the address has code (isContract returns true), it calls isValidSignature() via ERC-1271.
-If the address has no code, it assumes it’s an EOA and uses ECDSA.recover() to verify the signature.

This approach breaks under the upcoming Ethereum Pectra upgrade, which includes EIP-7702. EIP
7702 allows EOAs to temporarily attach code during a transaction, meaning any externally owned
account can now appear as a contract at runtime.
As aresult:
A 7702-enabled EOA with temporary code will cause the logic to treat it as a contract.
If the code does not implement ERC-1271,the isValidSignature() call will fail.
The logic does not fall back to ECDSA.recover(), causing legitimate EOA signatures to be rejected

**Team Response**

Fixed.

### [M-02] Lack of Minimum feeRate check may allow orders to be fullfilled without paying any fees

**Severity**

Medium

**Description**

The settleMatchedOrders function allows the backend to settle matched orders that
were created and matched off-chain. While the contract enforces an upper bound for feeRate, it does
not enforce a minimum feerate.
This omission allows orders with extremely low feeRate values(e.g.,1wei)to be matched and settled.
As aresult, users can effectively bypass protocol fees altogether, undermining protocol revenue and
fairness.
Since order matching is handled off-chain and executed on-chain by an address with the BACK
END_ROLE, malicious or careless matching could lead to widespread abuse of low-fee orders.

**Recommendation**

Introduce a minFeeRate variable, configurable by protocol administrators, and
enforce it during order validation to ensure all orders pay at least a minimum protocol fee:

```solidity
require(order.feeRate >= minFeeRate, "Fee rate below minimum allowed");
```

**Team Response**

Acknowledged


## Informational
**[I-01] Missing MarketID Consistency Check in settleMatchedOrders**

**Severity**

Informational

**Description**

The settleMatchedOrders function doesn’t enforce that all makerOrders share the same
marketId as the provided takerOrder. While it correctly verifies signature validity and array length
consistency, it does not validate that each makerOrder.marketId equals takerOrder.marketId

**Recommendation**

```solidity
for (uint i = 0; i < makerOrders.length; i++) {
 require(
 makerOrders[i].marketId == takerOrder.marketId,
 "FactCheckExchange: Market ID mismatch"
 );
 }
```

**Team Response**

Fixed.

**[I-02] Ensure Taker and Maker Orders Are on Opposite Sides**

**Severity**

Informational

**Description**

In the settleMatchedOrders function, there is currently no check to enforce that the
takerOrder and each makerOrder are on opposite sides of the trade (i.e., one is a buyer, the other
is a seller). While it may be assumed that the backend prevents invalid matches, relying solely on
off-chain guarantees can introduce hidden risks.

**Team Response**

Fixed.

**[I-03] Old Treasury Doesn’t Get Revoked When Updated To New One**

**Severity**

Informational

**Description**

In the updateTreasuryWallet() function, the contract sets a new treasuryWallet address
andgrants it the TREASURY_ROLE
However, the role from the previous treasury address is not revoked. As a result, any previously as
signed treasury address retains the TREASURY_ROLE

**Recommendation**

Revoke the old treasuryWallet

**Team Response**

Fixed.
