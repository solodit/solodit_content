**Auditors**

[Talfao](https://x.com/talfao1)

[suspiciousbandicoot](https://github.com/suspiciousbandicoot)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/038_CODESPECT_CARINA.pdf)

# Findings

## Medium Risk

### [M-01] Malicious solver can drain NativeTokenFlow contract via reentrancy

**Files:** [`NativeTokenFlow.sol`](https://github.com/carina-finance/carina-sc/blob/6ff2ad112b6b62acb054411b8181d9414970b08b/src/NativeTokenFlow.sol#L121)

**Description:**

The `NativeTokenFlow` contract allows users to create an order using the native token. Such an order is then fulfilled by a solver through the `Settlement` contract, after which the user receives the desired assets. A user can cancel their order via `cancelOrder(...)` before it is settled. If the order is successfully cancelled, the user is refunded the corresponding native token amount.

```solidity
if (orderFilledAmount < order.amountIn) {
    uint256 refundAmount = order.amountIn - orderFilledAmount;
    if (refundAmount > 0) {
        // ...

        (bool success,) = payable(orderMeta.owner).call{value: refundAmount}("");
        if (!success) {
            revert NativeTokenTransferFailed();
        }
    }
}
// ...
}
// ...
orderMeta.status = NATIVE_FLOW_ORDER_STATUS_CANCELLED;
```

However, as shown above, the cancellation status for the order is set after the low-level `call(...)`, which introduces a reentrancy vector. A malicious solver could exploit this vulnerability as follows:

1. Create their own order in the `NativeTokenFlow` contract;
2. Call `cancelOrder(...)` in the same contract;
3. When the refund is issued via `call(...)`, the attacker’s `receive(...)` function is triggered. From there, they can call `settle(...)` on the `Settlement` contract. This succeeds because `NativeTokenFlow::isValidSignature(...)` still considers the order valid (its status remains `NATIVE_FLOW_ORDER_STATUS_CREATED`).
4. As a result, the solver effectively uses the same `tokenIn` amount twice — first receiving a refund, and then fulfilling the order again.

**Impact:** Potential draining of the `NativeTokenFlow` contract. However, the overall impact is constrained by the actor’s ability to perform the attack and the available liquidity in the contract.

**Recommendation:** Set the cancellation status before performing the low-level call, e.g.:

```solidity
orderMeta.status = NATIVE_FLOW_ORDER_STATUS_CANCELLED;

// ...
(bool success,) = payable(orderMeta.owner).call{value: refundAmount}("");
```

**Status:** Fixed

**Client response:** Fixed in [3adbb8b541e3be631235179acfe1b3a416b2c230](https://github.com/carina-finance/carina-sc/pull/10/commits/3adbb8b541e3be631235179acfe1b3a416b2c230)

## Informational

### [I-01] EIP-1271 signature verification allows for a callback to user-controlled contracts

**Files:** [`mixins/OrderSigning.sol`](https://github.com/carina-finance/carina-sc/blob/6ff2ad112b6b62acb054411b8181d9414970b08b//src/mixins/OrderSigning.sol#L208-L210)

**Description:**

The settlement process, initiated by a Solver’s call to the `settle(...)` function, involves verifying user signatures for each trade. The protocol supports various signing schemes, including EIP-1271 for smart contract-based wallets. The verification for this scheme is handled in the `recoverEIP1271Signer(...)` function.

This function makes an external call to the user’s contract to invoke `isValidSignature(...)`, as required by the EIP-1271 standard. This callback occurs after the pre-settlement actions (`actions[0]`) have been executed but before the main liquidity-providing actions (`actions[1]`) and the final token transfers.

This creates an attack surface where a malicious user, via their smart contract wallet, can execute arbitrary logic during the settlement process. This could be used to manipulate the state of external protocols (e.g., AMM pool prices) that the Solver’s actions rely on. Such manipulation could lead to outcomes like transaction reverts due to slippage (denial of service against the Solver) or achieving a more favorable trade execution for the user at the Solver’s expense.

```solidity
// src/mixins/OrderSigning.sol

function recoverEip1271Signer(bytes32 orderDigest, bytes calldata encodedSignature)
    internal
    view
    returns (address owner)
{
    assembly {
        // owner = address(encodedSignature[0:20])
        owner := shr(96, calldataload(encodedSignature.offset))
    }

    bytes calldata signature = encodedSignature[20:];

    // @audit-issue This makes an arbitrary external call to a user-controlled contract.
    // It executes between the pre-settlement (setup) and main settlement actions.
    if (IEIP1271(owner).isValidSignature(orderDigest, signature) != EIP1271_MAGIC_VALUE) {
        revert InvalidEip1271Signature();
    }
}
```

Unlike traditional front-running, the logic that affects the Solver is encoded into the order’s own verification process via the smart contract wallet. This means that private mempools and other front-running protections are ineffective against this vector.

**Impact:** This issue is informational, as the behaviour is part of a standard integration. It aims to highlight the potential risk for Solvers, who are the primary actors exposed to this manipulation vector.

**Recommendation:** Consider documenting this behaviour to ensure that Solvers are aware of the potential for re-entrancy through EIP-1271 signature verification. Solvers should implement their own safeguards, such as performing pre-flight checks on external conditions and setting strict slippage parameters within their settlement actions to mitigate manipulation risk.

**Status:** Acknowledged

**Client response:** We acknowledge it and will document it in the solver integration guide for Carina.
