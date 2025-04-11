**Auditor**

[@IAm0x52](https://twitter.com/IAm0x52)

# Findings

## Medium Risk

### [M-01] PT donation attack will DOS spell deposit permanently

**Details**

[PendleSpell.sol#L128-L137](https://github.com/Blueberryfi/blueberry-core/blob/d0ed24769704cf5d9a8b0616cf534f29db32f6ca/contracts/spell/PendleSpell.sol#L128-L137)

    (uint256 ptAmount, , ) = IPendleRouter(_pendleRouter).swapExactTokenForPt(
        address(this),
        market,
        minPtOut,
        params,
        input,
        limitOrder
    );

    if (ptAmount != IERC20Upgradeable(pt).balanceOf(address(this))) revert Errors.SWAP_FAILED(pt);

After swapping from debt token to PT, the contract makes an exact check against the balance of the contract to ensure the swap executed as expected. The issue with this is that even if it is a single wei over, the contract will revert. This makes it trivial to permanently DOS opening positions via the contract by donating a small amount of PT to the contract.

**Lines of Code**

[PendleSpell.sol#L101-L147](https://github.com/Blueberryfi/blueberry-core/blob/d0ed24769704cf5d9a8b0616cf534f29db32f6ca/contracts/spell/PendleSpell.sol#L101-L147)

**Recommendation**

Check should be `>` rather than `!=`

**Remediation**

Fixed as suggested.
