**Auditor**

[Kann Audits](https://kannaudits.com/audits/hyperlend-leveraged-lending-may-2026)

# Findings

## Low Risk

### [L-01] Swap Adapters Wrap Entire Contract Native Balance Which Inflates Swap Output

**Severity**

Low

**Description**

In `swapExactTokensForTokensSupportingFeeOnTransferTokens`, when `tokenOut` is `WHYPE`, the adapter wraps the contract’s entire native token balance into `WHYPE`:

```solidity
if (address(this).balance > 0) {
    WHYPE.deposit{value: address(this).balance}();
}
```

As a result, any native tokens already present in the contract before the swap execution are also wrapped and included in the final `balanceOut` calculation together with the actual swap output.

This causes the output amount to be inflated since unrelated contract funds are accounted for as part of the current swap result.

```solidity
function swapExactTokensForTokensSupportingFeeOnTransferTokens(
    ...
) external nonReentrant {
    ...

    (bool success, ) = gluex.call(gluexCallData);
    require(success, "GluexAdapter: gluex swap failed");

    if (address(this).balance > 0) {
        WHYPE.deposit{value: address(this).balance}();
    }

    uint256 balanceOut = IERC20(tokenOut).balanceOf(address(this));

    require(
        balanceOut >= amountOutMin,
        "GluexAdapter: minAmountOut > balanceOut"
    );

    IERC20(tokenOut).safeTransfer(to, balanceOut);
}
```

**Impact**

User receive more `balanceOut` than intended since unrelated native funds already present in the contract are included in the swap result and transferred as part of the output amount.

**Team Response**

Acknowledged.

## Informational

### [I-01] Redundant `block.number` check in transient storage

**Severity**

Informational

**Description**

Both `LiquidSwapAdapter` and `GluexAdapter` store the current `block.number` in transient storage during `setSwapPath` and later verify that it matches during `swapExactTokens....` The apparent intention is to ensure that the swap path is only valid if it was set within the same transaction as the swap execution.

```solidity
function setSwapPath(
        address tokenIn,
        address tokenOut,
        bytes calldata gluexData
    ) external {
        // Generate unique slots for this token pair in transient storage
        bytes32 baseSlot = keccak256 (abi.encodePacked(tokenIn, tokenOut));
        bytes32 blockSlot = keccak256 (abi.encodePacked(baseSlot, "block"));

        assembly {
            tstore(blockSlot, number())
            tstore(baseSlot, gluexData.length)
        }

        // Store data in chunks of 32 bytes
        uint256 length = gluexData .length;
        for (uint256 i = 0; i < length; i += 32) {
            bytes32 chunk;
            assembly {
                chunk := calldataload (add(gluexData.offset, i))
                tstore(add(baseSlot , add(1, div(i, 32))), chunk)
            }
        }
    }
```

However, this additional `block.number` validation is redundant in the context of EIP-1153 transient storage. Under EIP-1153, transient storage is automatically cleared at the end of every transaction, not at the end of a block. As a result, any value written via `tstore` in one transaction cannot persist into a subsequent transaction, even if it occurs within the same block.

Therefore, the safety property the check attempts to enforce ensuring same transaction usage is already guaranteed by the EVM’s transaction-scoped lifetime of transient storage. The comparison against `block.number` does not strengthen this guarantee and does not provide additional security beyond what EIP-1153 inherently enforces.

**Team Response**

Resolved.

### [I-02] Closing a Full Position Fails if the User Has Unrelated Debt in the Same Asset

**Severity**

Informational

**Description**

In `Looping.sol`, users can close a leveraged position by providing the flashloan parameters and setting `withdrawAmount` to `type(uint256).max`. This signals the contract to fully unwind the position by reading the user’s current yield token and debt token balances.

```solidity
function _executeClosePosition(bytes memory params, address debtAsset) internal {
    (
        ...
    ) = abi.decode(...);

    IERC20 hYieldToken = IERC20(
        IPool(msg.sender).getReserveData( yieldAsset).aTokenAddress
    );

    // close full position if withdrawAmount == maxUint256
    if (withdrawAmount == type(uint256). max) {
        IERC20 debtDebtToken = IERC20(
            IPool(msg.sender)
                .getReserveData(debtAsset)
                .variableDebtTokenAddress
        );

        repaymentAmount = debtDebtToken. balanceOf(user);
        withdrawAmount = hYieldToken. balanceOf(user);
    }

    ...
}
```

However, `debtDebtToken.balanceOf(user)` returns the user’s global debt balance for that specific asset across the entire lending pool. If the user has other active borrows using the same debt token (backed by different collateral assets outside of this specific looping strategy), `repaymentAmount` will be set to a value higher than the `flashloanAmount`. Because the `Looping` contract only holds the exact amount of `debtAsset` it flashloaned, the subsequent call to `IPool.repay` will revert due to insufficient balance.

If user opens first USDC/WETH position by supplying WETH then USDC/WBTC. Debt token is the same. If user wants to exit with the full WETH position, we wouldnt be able to since `repaymentAmount = debtDebtToken.balanceOf(user);` will take the debt balance from both positions.

To mitigate this we could specify the exact `repaymentAmount` amount instead of putting `uint256 max`. Or we could flashloan more amount so it repays all the debt but its not more than the collateral that will be swapped to the `debtAsset`. In this situation the flashloan amount < WETH. Then the second position is now without any debt so user can just call withdraw

**Team Response**

Acknowledged.

### [I-03] StrategyManager Cannot Receive Native Token Refunds Despite Native Cleanup Branch

**Severity**

Informational

**Relevant code**

```solidity
if (tokens[i] == address (0)){
        (bool sent,) = owner().call{value: address(this).balance}("");
        require(sent, " cleanOutTokens: failed to send native");
}
```

**Description**

`StrategyManager` contains logic for cleaning out native balances when `address(0)` is passed, which implies native-asset handling was considered. However, the contract does not implement `receive()` or a payable `fallback()`, so standard native refunds or direct native transfers to the contract revert.

**Team Response**

Acknowledged.

### [I-04] StrategyManagerFactory Registry Becomes Stale After Ownership Transfer

**Severity**

Informational

**Description**

`StrategyManagerFactory` indexes strategy managers by the address that created them, but `StrategyManager` ownership is transferable after deployment.

Once a manager is transferred to a new owner, the factory registry is not updated. The original creator remains recorded as the owner of that tuple while the real new owner is invisible to the registry.

**Team Response**

Acknowledged.
