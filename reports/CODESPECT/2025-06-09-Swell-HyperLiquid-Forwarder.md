**Auditors**

[Talfao](https://x.com/talfao1), [Kalogerone](https://x.com/kalogerone)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/023_CODESPECT_SWELL_HYPERLIQUID_FORWARDER.pdf)

# Findings

## High Risk

### [H-01] The HYPE bridge address is hardcoded as the WHYPE bridge address

**Files:** [`HyperliquidForwarder.sol`](https://github.com/SwellNetwork/hyperliquid-forwarder/tree/59910084b7669f721330be6dc68557b6ac747c4b/src/HyperliquidForwarder.sol#L45)

**Description:**

The `HyperliquidForwarder` contract has 2 hardcoded addresses:

```solidity
address public constant WHYPE = 0x5555555555555555555555555555555555555555;
address private constant WHYPE_BRIDGE = 0x2222222222222222222222222222222222222222;
```

However, according to the HyperLiquid docs ([link](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/hypercore-less-than-greater-than-hyperevm-transfers#system-addresses)), the `0x2222222222222222222222222222222222222222` address is HYPE’s system address and not WHYPE’s. These values are implemented in the `addTokenIDToBridgeMapping(...)` function:

```solidity
function addTokenIDToBridgeMapping(address tokenAddress, address bridgeAddress, uint16 tokenID)
    public
    requiresAuth
{
    // HYPE/WHYPE is an exception and is handled separately, do not allow an owner to incorrectly set it
    if (tokenAddress == WHYPE) {
        tokenAddressToBridge[WHYPE] = WHYPE_BRIDGE;
        return;
    }
    ...
}
```

**Impact:** Any WHYPE tokens that will be sent through this contract will be lost.

**Recommendation:** Replace the WHYPE for the HYPE token to correctly use the bridge. However, HYPE transfer should be treated like native ETH transfer; therefore, the implementation of the `forward(...)` function should be changed.

**Status:** Fixed

**Client response:** Resolved with commit [56e4679ae55aff64738274209fb526774f3a2d54](https://github.com/SwellNetwork/hyperliquid-forwarder/commit/56e4679ae55aff64738274209fb526774f3a2d54)

## Informational

### [I-01] The arbitrary evmEOAToSendToAndForwardToL1 parameter allows for sending tokens to the wrong address

**Files:** [`HyperliquidForwarder.sol`](https://github.com/SwellNetwork/hyperliquid-forwarder/tree/59910084b7669f721330be6dc68557b6ac747c4b/src/HyperliquidForwarder.sol#L67)

**Description:**

In the `forward(...)` function, the `msg.sender` sends his own tokens from the HyperEVM to the HyperCore where the `evmEOAToSendToAndForwardToL1` address receives them. Considering that the protocol will use 5 multi-sig addresses, such an address could be verified against them in contract level to remove the possibility of sending tokens to the wrong address.

**Status:** Fixed

**Client response:** Resolved with commit [56e4679ae55aff64738274209fb526774f3a2d54](https://github.com/SwellNetwork/hyperliquid-forwarder/commit/56e4679ae55aff64738274209fb526774f3a2d54)

### [I-02] There are no checks that the system address on HyperCore has sufficient supply

**Files:** [`HyperliquidForwarder.sol`](https://github.com/SwellNetwork/hyperliquid-forwarder/tree/59910084b7669f721330be6dc68557b6ac747c4b/src/HyperliquidForwarder.sol)

**Description:**

According to the [HyperLiquid documentation](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/hypercore-less-than-greater-than-hyperevm-transfers#caveats), when bridging tokens between HyperEVM and HyperCore, there are currently no checks to verify that the system address has sufficient supply. Furthermore, these checks cannot be implemented at the contract level, meaning transactions that send funds to the bridge cannot be reverted automatically in such cases.

Therefore, the protocol team should always exercise caution before performing large token transfers.

**Status:** Acknowledged

**Client response:** Acknowledged
