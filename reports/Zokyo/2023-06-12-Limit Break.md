**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Edge case in business logic leads to stolen funds

**Severity**: Critical

**Status**: Resolved

**Description**

In contract PaymentProcessor.sol, to identify if the nft token is either ERC1155 or ERC721 the structure MatchOrder have a property named protocol, which is an enum and can have the values TokenProtocols.ERC1155 or TokenProtocols.ERC721, this check is done so that the contact knows which transferFrom function to call, safeTransferFrom in the case of ERC1155 or transferFrom in the case of ERC721, However this can be exploited because that parameter is not part of neither the buyer or the seller signature so it can be manipulated, let’s take the following example: 
Alice (Seller) and Bob(buyer) agree to make an otc deal where Alice sells 1 ERC1155 token at 1 eth. 
They both sign the details and everything looks normal. 
Before sending the transaction Alice changed the value of the protocol property from the MatchOrder structure from ERC1155 to ERC721.
Alice calls the buySingleListing with the correct signatures however the protocol parameter is modified. 
When the execution logic from _executeMatchOrder  will be at line #1306 where it will choose what function to call, because the protocol value is ERC721 instead of ERC1155 will call the transferFrom function, if the transferFrom function will not exist in ERC1155 the execution will revert, however if the ERC1155 will have an empty fallback function implemented, the execution will succeed but in reality no nft’s will be transferred.

**Recommendation**: 

Add the property protocol from MatchOrder structure in the both seller and buyer signatures.

**POC** : https://hackmd.io/@QyPimM2nQzSOYG0jHv2aIg/HkOuspaz3





###  Edge case in business logic lead to stolen nft 

**Severity**: High  

**Status**: Resolved

**Description**

In contract PaymentProcessor.sol, in function _executeMatchOrderSale, when the sale is paid in ether ( paymentCoin is address(0) ), the contract logic is checking that the offer is accepted by the seller through the flag sellerAcceptedOfer, if that flag is true, the execution will be reverted because it means that the transaction has been originated from the seller, and the seller will pay in ether for his own nft. However, that logic can be manipulated because a malicious marketplace or frontend could put the parameter on false even if the transaction has originated from the seller.

**Recommendation**: 

Remove the parameter from the structure and do that check using blockchain state instead of function parameters. That can be achieved by checking the msg.sender and the tx.origin variables are different from the seller of the nft.

**POC**: https://hackmd.io/@QyPimM2nQzSOYG0jHv2aIg/BJzo-C6zh






### Using transferFrom can lead to edge cases when the token is an weird implementation of erc20

**Severity**: High

**Status**: Resolved

**Description**

In contract PaymentProcessor.sol, in functions _payoutCoinCurrency, _computeAndDistributeProceeds, the transfer of tokens is done by simply calling the transferFrom function, however this is a bad implementation decision because the erc20 is an old standard and there are some erc20 tokens that will not revert the execution in case of an failed transfer and they will only return false, as the contract logic is not checking for the transferFrom function output the contract execution will proceed even if the tokens transfer in reality have failed.

**Recommendation**:  

Use the safeTransferFrom function from the SafeERC20 library instead of plain transferFrom.

**POC** : https://hackmd.io/@QyPimM2nQzSOYG0jHv2aIg/HyllXJCGn


## Medium Risk

### No sanity checks when setting a security policy to a collection

**Severity**: Medium

**Status**: Resolved

**Description**

In contract PaymentProcessor.sol, in function setCollectionSecurityPolicy, when a security policy id is set there is no check that a security policy corresponding to that id actually exists. This issue can lead to undefined behaviors as a collection owner can set an uninitialized security policy by mistake. 

**Recommendation**: 

Add a boolean property in the security policy structure called isCreated, that default will have the false value and turn to true only when the security policy is created, then check in the function setCollectionSecurityPolicy that property is true.

### Logic edge case if a chain re-organisation happens

**Severity**: Medium

**Status**: Unresolved

**Description**


In contract PaymentProcessor.sol, the flow is firstly checking if the enforcePricingConstrains is enable and if it is enable is checking if the price is set between floor price and ceiling price, if it is not enabled then the execution will continue without checking if the price is between the given values, the floor price and ceiling price are set in the “setTokenPricingBoundaries” function and the enforcePricingConstrain is set during the security policy update or create flows, that means that the enabling of eforcePricingConstrain and the setting of the floor price and ceiling price can be set in different transaction, in a case of a chain re-organisation this can be problematic, let’s explore the following scenario: 
	Scenario #1 ( normal flow ): 
			1. Bob setup his collection, creating a security policy where he enables the price constrain enforce  in tx #1 block #1
			2. Bob setup his pricing bound by calling the function “setTokenPricingBoundaries” in tx#2 block #1 
			3. Alice, Carol & David execute different nft  transactions using PaymentProcessor over Bob collection, creating tx #3, #4 & #5 that will also get indexed in block #1. 

Scenario #2 ( chain re-organisation happens): 
		1. Bob setup his collection, creating a security policy where he enables   the price constrain enforce  in tx #1 block #1
		2. Alice, Carol & David execute different nft  transactions using PaymentProcessor over Bob collection, creating tx #2, #3 & #4 that will also get indexed in block #1. 
		3. Bob setup his pricing bound by calling the function “setTokenPricingBoundaries” in tx#5 block #1. 

Because the transaction got re-organisat inside the latest block and then between the enabling of enforcing price constrain transactions  and the setting of floor price & ceiling price we’re putted other transactions that we’re executing order on PaymentProcessor, those transactions that got between them have been reverted as they are not matching the ceilingPrice value, which will now be 0 by default.	



**Recommendation**: 

When enabling the price constrains also setup the floor price and ceiling price in the same function. 





### DOS through sandwich attack on ceilingPrice 

**Severity**: Medium

**Status**: Unresolved

**Description**

In contract PaymentProcessor.sol, function setTokenPricingBounds is used to set floorPrice and ceilingPrice for certain tokenIds, there is only one sanity check at line #571 that will revert if floorPrice is bigger then ceilingPrice, however floorPrice and ceillingPrice can have the same value and execution will continue successfully, allowing the value of ceilingPrice to be set as 0, this can be leveraged through a sandwich attack if the contract owner/admin whish to deny certain transactions. . 

**Recommendation**: 

Add a sanity check to ensure that ceilingPrice is different from floorPrice or that ceilingPrice is different from 0.




## Low Risk

### No sanity checks when adding a new coin

**Severity**: Low

**Status**: Resolved

**Description**

In contract PaymentProcessor.sol, in function approveCoin ( renamed to whitelistPaymentMethod in latest commit ), there are no sanity checks over the coin parameter, if the coin parameter will be set to the zero address or to an address that have no bytecode associated to it all the calls to that address will succeed but in reality nothing will change in the blockchain state, no tokens/value will be transferred.

**Recommendation**: 

1. Add a check to ensure the coin parameter is not address(0).
2. Add a check to call the decimals functions from the erc20 ( this is a function that all erc20 should implement ) and check that the result is different from 0


 


### Missing zero address check for account parameter

**Severity**: Low

**Status**: Resolved

**Description**

The whitelistExchange() function has a missing zero address check for account parameters. This can result in a zero address being accidentally passed in the account parameter.

**Recommendation**: 

It is advised to add a missing zero address check for the account parameter.

