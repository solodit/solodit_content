**Auditors**

[AuditOne](https://twitter.com/auditone_team)

# Findings

## High Risk

### setTradeOer and cancelTradeOer functions does not use ReentrancyGuard

**Description:** In the smart contract, the ReentrancyGuard from OpenZeppelin is imported to prevent reentrancy attacks. However, the functions setTradeOer and cancelTradeOer are not utilizing the ReentrancyGuard, making them vulnerable to reentrancy attacks. Both the setTradeOer function and the cancelTradeOer transfers tokens and Ether to a contract address, If an attacker can create a recursive call and re-enter either of these functions before they finish executing, they can potentially manipulate the state of the contract and exploit it to their advantage.

**Recommendations:** To prevent this type of attack, it is important to use the ReentrancyGuard or other similar methods to ensure that the functions are executed atomically and not re-entered until the previous invocation has been completed.



### Contract funds can be drained out

**Description:**

User can call the setT[radeOer and pl](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/TradingPost.sol#L86)ace the trade.

For this, the contract will receive the fund msg.value to the contract.

Later if the user wants to cancel the trade order, they can call the cancelTr[adeOer and the use](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/TradingPost.sol#L154)r would get the refund.

The issue here is, the cancelTradeOer makes external call to transfer the msg.value to the user.

this external call opens the route for reentrancy. The user can re-enter again and withdraw the funds. This can be repeated till all the contract funds are drained.

Note : the function cancelTradeOer does not have the reentrancy protection.

It is updating the stat after calling call . Refer the link

**Recommendations:**

- Add non-reentrancy guard.
- make the state changes before making the external call.



### makeBuyOer &withdrawBuyOer can be used to brick the acceptBuyOer

**Description:**

There are two ways,the token owner can sell the tokens.

1.Purchase - called by non-token owner

2.acceptBuyOer - called by token owner.

if there are no user to call the Purchase ,then the token owner can use the acceptBuyOer route to sell the token.

The issue is,if the buyer wants to buy the token by placing the makeBuyOer. ,new user can overwrite this makeBuyOer by slightly paying extra amount. as seen in the below line of codes, [https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/HeroMarketplace.sol#L209-L214.](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroMarketplace.sol#L209-L214)

after calling the makeBuyOer and overwriting the existing activeBuyOers,the malicious buyer could call the withdrawBuyOer to withdraw.

So there will not be any activeBuyOers for the token.

**Recommendations:** 

have some time delay to call the withdrawBuyOer once the makeBuyOer is called and placed the activeBuyOers



### makeBuyOer can be used to place the activeBuyOers with zero msg.value

**Description:** 

non-token owner can place the buy order using the function makeBuy[Oer.](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroMarketplace.sol#L197-L240)

The function checks that the existing active activeSellOers and activeBuyOers as shown below,

But it does not check whether the msg.value is greater than 0.

when there are no active order placed , the value activeBuyOers[tokenId].price is zero. so, the second if condition will not be executed.

So, the non-token owner can place , without any msg.value

Here comes the issue,

when the token owner calls the function acceptBuyOer( function, it transfer the tokens to the buyer and sends the msg.value to the seller.

If the above mentioned scenario happens, the seller would receive zero msg.value but transfers token to the buyer.

**Recommendations:**

check msg.value is > 0 when calling the makeBuyOer()




### Reentrancy exploit in initiateAdventure function.

**Description:** 

The \_initiateAdventure function is called both by the goAdventure function and the contractAdventure function. In the contractAdventure function,the user needs to pay an amount of ether to rent the hero before starting the adventure. However,the ether transfer to the contract owner happens before the \_initiateAdventure function is called. An attacker could potentially create a malicious contract that would call the contractAdventure function and then recursively call it again before the first call is complete,exploiting the goAdventure function as well. 

**Recommendations**:

One possible solution is to use the "Checks-Eects-Interactions"pattern. This means that the contract should first check if the calling conditions are met,then perform all internal state changes,and finally interact with other contracts. By following this pattern,it becomes more dicult for an attacker to exploit the contract's logic. Specifically,in this case,the developer should ensure that the ether transfer to the contract owner happens after the

_initiateAdventure function is called,rather than before. This can be achieved by using a withdrawal pattern where the contract owner can withdraw the funds after the adventure is completed and the payout function has been called.

It is also recommended to use the ReentrancyGuard library from OpenZeppelin


### calling mintFounderHero() may accidentally lock user funds


**Description:** 

Lets understand the issue with the help of an example.

The price of founderHeroPrice = 1ether and the user wants to mint 5 NFTs but instead of 5 ether if the user sends 5.5 ether,

then the extra .5 ether sent cannot be recovered by the user.

**Recommendations:** 

There are currently two ways to fix this.

- update the condition on [https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1 584015e98/contracts/Factory.sol#L62 to be](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/Factory.sol#L62) msg.value == founderHeroPrice \*amount
- Add a condition to check if some extra ether was sent and allow the user to access it.



### Using weak source of randomness via block.timestamp for generating the Anity 

**Description:** 

Use of block.timestamp is insecure. There is no true source of randomness present in the random function. The input parameters are known values. This makes is easier for the attacker to guess the value. The attacker can precompute multiple block.number,where if he mints the MintFounderHero,he gets the maximum values for strAnity,agiAnity,intAnity. 

**Recommendations:** 

Using external sources of randomness via oracles like Chainlink VRF.



## Medium Risk

### Use of low level calls

**Description:**

The call function is a low-level function in Solidity that allows contracts to call other contracts and pass in arbitrary data. While it can be useful in certain situations,it can also be dangerous if not used properly. The main issue with the call function is that it allows external code to be executed within the context of the calling contract, which can potentially lead to reentrancy attacks and other vulnerabilities. In addition, the call function does not perform any input validation or type checking, which can lead to unexpected behavior if the input data is not formatted correctly.

In the first call statement, the contract is sending a value amount to the \_feeReceiver address. An attacker could enter in a malicious or zero address,thereby leading to a loss of funds. 

**Recommendations:** 

To mitigate this issue, it's generally recommended to avoid using the call function wherever possible, and to use higher-level functions like send or transfer instead. These higher-level functions provide more safety guarantees and are less prone to errors and vulnerabilities.. It is also important to validate the \_feeReceiver addresses before sending ether.

**Status:** Resolved 



### Incorrect value is emitted during fulfillTradeOer for trade.askValue and for trade.oerValue when there are fee is deducted

**Description:** 

During fu[lfillTradeOer,fee](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/TradingPost.sol#L185) is deducted for askValue and oerValue at below location,

[https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/TradingPost.sol#L207-L228](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/TradingPost.sol#L207-L228)

so after deductin the fee,the actual askValue and oerValue are lesser than previous.

But when we see the emit event at the end of the function,it still use the old value to emit. refer the below link [https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/TradingPost.sol#L263-L264](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/TradingPost.sol#L263-L264)

**Recommendations:** 

use the correct askvalue and oerValue if there are any fee is deducted. 



### P[urchase tak](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroMarketplace.sol#L153)es more than what is needed.

**Description:** 

To purchase the token,use can call the purcha[se.](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroMarketplace.sol#L130-L190)

After checking all the conditions,it checks for the msg.value >= price . This mean the send msg.value could be larger than the price value.

At [line176,](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroMarketplace.sol#L176)all the msg.value is used for further calculation like fee deduction and pay to the seller. If the msg.value is greater than the price,purchaser would incur loss.

**Recommendations:** 

Consider the price vale for further calculations at below mentioned line, [https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/HeroMarketplace.sol#L176](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroMarketplace.sol#L176)



### Missing input validation for various functions

**Description:**

The functions above do not explicitly check whether the inputted address is a zero address or a malicious address. This can allow an attacker to set an arbitrary address as the token address, potentially leading to funds being sent to the attacker or other unintended consequences. **Recommendations:** It would be a good practice to add input validation to ensure that the inputted address is not a zero/malicious address. This can be done by adding a require statement at the beginning of each function to check whether the inputted address is valid. For example:



### Lack of re-entrancy guard for \_mint[Hero](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroNFT.sol#L68)

**Description:**

 \_[mintHero fu](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroNFT.sol#L68)nction can be called by the minter to mint the Hero NFT.

This function can be called in above mentioned places.

When we see the minting place,it uses the safeMint which means that during minting,the process verified whether the receiver address is able to receiee the NFTby calling the onERC721Received.selector. This places allows for reentrancy.

after minting the NFT,count is increased.

**Recommendations:** 

Add non reentrancy protection in following places

[https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/HeroNFT.sol#L68 ](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroNFT.sol#L68)[https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/Factory.sol#L49 ](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/Factory.sol#L49)[https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/Factory.sol#L72 ](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/Factory.sol#L72)[https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/Factory.sol#L155 ](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/Factory.sol#L155)[https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/Factory.sol#L172](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/Factory.sol#L172)



### Lack of implementation to remove the whitelisted address

**Description:** 

In Shop.sol ,mapping mapping(uint256 => mapping(address => bool)) storeIdToUserInWhitelist;is used to store the whitelisted address for each store id. The whitelisting process is done by onlyStoreManagers with help of the functio[n .](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/Shop.sol#L188-L206)

Later this whitelisted addresses are allowed to interact with function buyProduct(to buy the product even if the shop is closed,

refer the l[ine](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/Shop.sol#L111-L116)



### ROYALTY's are hardcoded

**Description:** 

The percentage of royalty or service fee is hardcoded. In future,if the developers or the community decide to update the percentage via governance etc.,it will be not be possible. 

**Recommendations:**

Add a function to update the percentage of fee collected protected by Owner.

**Status:** Resolved 



## Low Risk

### Missing or insufficient Natspec comments

**Description:** 

The issue of missing/insufficient Natspec in Solidity code is one that can have a significant impact on the quality of the code,and ultimately,the functionality of the smart contract. Natspec is a tool that allows developers to add human-readable comments to their code,providing context,and explanations of the code's purpose and functionality. When Natspec is missing from Solidity code,other developers who read the code may struggle to understand what the code is doing,making it harder to modify,maintain,and reuse the code. 

**Recommendations:**

Consider including Natspec comments in the contract code, explaining its purpose and functionality.



### Cache the array length and use it in loops

**Description:** For example,refer the link [in Tra](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/TradingPost.sol#L282-L300)dingPost.sol

**Recommendations:** Change the code to have control over loops



### Malicious Minter can mint tokens to zero address

**Description:** 

Minting the tokens to zero address will inflate the supply of tokens but they cannot be recovered.

**Recommendations:** 

add a zero address check

require(account != address(0),"cannot mint to zero address");



### Missing return value check for transferFrom

**Description:** 

If the transferFrom returns false instead of revert,the chargeReforgingStation()will execute successfully but no transfer of gold ERC20 tokens will have taken place. 

**Recommendations:** 

check for Boolean return value of transferFrom



## Informational

### Missing event logging

**Description:** 

This contract does not emit any events to notify external actors of changes to its state,which makes it harder to monitor and detect potential exploits or attacks. 

**Recommendations:** 

Consider logging events and adding event emissions to functions.



### Inconsistent naming convention

**Description:** 

The inconsistency in naming convention in the setfeeReceiver function of the contract can cause confusion and errors. It should be changed to setFeeReceiver to match the naming convention used in other functions of the contract. This will help improve readability and avoid confusion when interacting with the contract.

**Recommendations:** 

setfeeReceiver should be changed to setFeeReceiver



### No use of Constant,View and Pure to mark functions

**Description:** 

It’s generally a good practice to mark functions as constant,view,or pure whenever possible,especially if they do not modify the state of the contract or perform external calls or I/O operations. However,it’s important to carefully consider the behavior of the function and whether it is appropriate to mark it as constant,view,or pure.

**Recommendations:**

Consider indicating functions as constant,view,or pure whenever possible.



### Potential gas limit issues

**Description:** 

The research()function has multiple for loops that could cause gas limits to be exceeded if the reward structure is too complex.

**Recommendations:**

A potential fix could be to limit the maximum number of rewards that can be added to a token identifier,reducing the number of iterations needed to calculate the reward probability.



### unused function and variable found

**Description:**

Unused variable - [(https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e158401 5e98/contracts/HeroMarketplace.sol#L31)](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/HeroMarketplace.sol#L31)

unused function - [https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015 e98/contracts/TradingPost.sol#L271](https://github.com/Crelde/IOTA-Heroes-Contracts/blob/3e345747f723637c0a1ce884d1ae0e1584015e98/contracts/TradingPost.sol#L271)

**Recommendations:**

Remove unused variables and functions



### Use of floating pragma 

**Description:** 

Locking the pragma helps to ensure that contracts do not accidentally get deployed

using,for example,an outdated compiler version that might introduce bugs that affect the contract system negatively.

**Recommendations:**

Lock the pragma version and also consider known bugs [(https://github.com/ethereum/solidity/releases)for the](https://github.com/ethereum/solidity/releases) compiler version that is chosen.



### Missing events for only functions that change critical parameters

**Description:** 

The functions that change critical parameters should emit events. Events allow capturing the changed parameters so that o-chain tools/interfaces can register such changes. For Example :The Graph

**Recommendations:**

Add events to all functions that change critical parameters.

