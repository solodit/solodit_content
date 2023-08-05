**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Medium Risk

### [M-01] Malicious owner could arbitrage sales

**Impact:**
High, as it will charge users more than the should be charged

**Likelihood:**
Low, as it requires a malicious/compromised owner

**Description**

Currently, the `setFeeRate` and `setLocalFeeRate` methods do not have an upper bound on the fee rate being set by the owner. This opens up a centralization attack vector, where the owner can front-run trades by setting a bigger fee. Consider the following scenario:

1. Alice puts a 100 ETH bid for an Alien Punk, considering fee is 1% and she actually is bidding 99 ETH
2. Bob puts an Alien Punk for sale for 98 ETH
3. Now instead of Alice paying 99 ETH (giving 1 or 0.9 to the protocol as fee) and being left with the punk + 1 ETH, the admin can set the fee to 2% and then execute the trade, essentially taking 1 ETH more from Alice.

**Recommendations**

Set upper bounds (limits) to both `setFeeRate` and `setLocalFeeRate` methods and revert if the value getting set is higher. This way users will know that fees can maximally go up to a particular number.

## Low Risk

### [L-01] The `chainId` is cached but might change

Caching the `chainId` value is not a good practice as hard forks might change the chainId for a network. The better solution is to always check if the current `block.chainid` is the same as the cached one and if not, to update it. Follow the approach in [OpenZeppelin's EIP712 implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/2271e2c58d007894c5fe23c4f03a95f645ac9175/contracts/utils/cryptography/EIP712.sol#L81-L87).

### [L-02] The `ecrecover` precompile is vulnerable to signature malleability

By flipping `s` and `v` it is possible to create a different signature that will amount to the same hash & signer. This is fixed in OpenZeppelin's ECDSA library like [this](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/dfef6a68ee18dbd2e1f5a099061a3b8a0e404485/contracts/utils/cryptography/ECDSA.sol#L125-L136). While this is not a problem since there is the `canceledOrFilled` mapping, it is still highly recommended that problem is addressed by using ECDSA.
