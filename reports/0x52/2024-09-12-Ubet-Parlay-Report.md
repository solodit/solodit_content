**Auditor**

[@IAm0x52](https://twitter.com/IAm0x52)

# Findings

## Low Risk

### [L-01] Events in BatchBet can be spoofed using custom pool/market

**Details**

[BatchBet.sol#L89-L111](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/BatchBet.sol#L89-L111)

        for (uint256 i = 0; i < buys.length; i++) {
    @>      address market = buys[i].market;

            if (!market.supportsInterface(MARKET_MAKER_V1_2_INTERFACE_ID)) {
                revert NotAMarket(market);
            }

            token.safeApprove(market, buys[i].investmentAmount);

            (, uint256 feeAmount,) = IMarketMakerV1_2(market).buyFor(
                buyer,
                buys[i].investmentAmount,
                buys[i].outcomeIndex,
                buys[i].minOutcomeTokensToBuy,
                extraFeeDecimal,
                feeProfileId
            );
            totalFees += feeAmount;
        }

        if (totalInvestment > 0 && (data.length > 0 || affiliate != address(0x0))) {
    @>      emit BuyWithData(buyer, affiliate, address(token), totalInvestment, data);
        }

Above we see that the `market` interacted with can be arbitrarily supplied by the user. This allows specifying a contract different from an UBet owned market. Along with this take note that the `market` address is never specified inside the `BuyWithData` event. The result is that events can be spoofed to make user/affiliate volume look significantly higher than the real value. This can have a negative effect on any offchain analytics or calculations.

**Lines of Code**

[BatchBet.sol#L109-L111](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/BatchBet.sol#L109-L111)

[BatchBet.sol#L155-L157](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/BatchBet.sol#L155-L157)

**Recommendation**

Include the market address in event so that non-UBet market data can be filtered out

**Remediation**

### [L-02] Transferring ERC1155 tokens before applying parent return ops allowing reentrancy

**Details**

[MarketMaker.sol#L392-L397](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/MarketMaker.sol#L392-L397)

    @>  conditionalTokens.safeTransferFrom(address(this), receiver, positionId(outcomeIndex), outcomeTokensBought, "");
        // Last index outcome is the refund outcome. Give back the same amount of tokens as collateral invested, including fees
    @>  conditionalTokens.safeTransferFrom(address(this), receiver, positionId(refundIndex), investmentAmount, "");

        // Return collateral back to parent once everything is settled with the buyer
        _applyParentReturn(parentOps);

Above we see that inside the `buyFor` function the `conditionalTokens` (ERC1155) are transferred to the buyer prior to the the collateral being returned to the parent pool. This enable reentancy prior to a significant state change to the market funding pool. Although no significant attack vectors were discovered as a result of this reentrancy is it still recommended to resolve this issue it may cause vulnerabilities in future versions.

**Lines of Code**

[MarketMaker.sol#L392-L397](https://github.com/SportsFI-UBet/ubet-contracts-v1/blob/7b61ff6631091056be51bf0bd88377560f6986f7/contracts/markets/MarketMaker.sol#L392-L397)

**Recommendation**

Move token transfers to after \_applyParentReturn. Additionally utilized safeBatchTransferFrom to prevent reentrancy between conditional token transfers.

**Remediation**
