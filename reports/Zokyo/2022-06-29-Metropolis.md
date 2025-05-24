**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Critical Risk

### Incorrect random source usage in the lottery
**Description**

Getting a random value from the Chainlink is used incorrectly. The requestRandom Words() request receives a requestld and then must wait for a response transaction from the coordinator. Only when the coordinator calls the fulfillRandomWords method, the contract will receive a valid random value with which to determine the winner. File: VRFLottery.sol 112-120

uint winner = passportlds[s_randomRange];

//add the wl to the passport

WLCont.addWLSpot(winner, city, building Name, buildingId);

//decrease win chances for winner

WinCont.updateAfter Win(winner, city, buildingId);

//loosers claim thier own increases

//add to mapping

_drawsDone[city] [buildingld] = true;

_winners[city][buildingld] = winner;

**Recommendation**

This part of the logic should be separated and should be called in or after the fulfillRandomWords method

**Re-audit comment**

Resolved.

Post-audit:

The alternative proposed by the Metropolis team had a serious vulnerability in random number generation which was based on block.timestamp, block.difficulty, msg.sender values. So the info will become known to the miner at the time of the transaction mint and can be manipulated. So, Metropolis team has moved the lottery logic off chain.

From client:

We are likely going to take this off chain so I have added a function which allows us to manually add a winner, along with a recording of the draw that selected the winner.

### Payment is calculated wrong in case a new payee is added.
**Description**

PaymentSplit.sol

In case, a new payee is added, pending payment for other users will be calculated wrong in case they have released tokens before. Example:

User has a share, equal to 10. Total shares is 100.

User1 calls release(user1 Address).

total Received $=100$ ETH.

payment $=100^{*}10/100-0.$

_released(user1Address) $=10$

_totalReleased $=10$.

After that, another user is added with a share, equal to 10. total shares is 110.

User1 calls release again.

totalReceived $=105$ ETH. (Additional ETH was transferred to contract).

payment $t=105^{*}10/110-10=9.54-10$ An underflow is occured, preventing user from claiming. Future payment might also be corrupted.

**Recommendation**

Recalculate shares for users, so that no underflow occur and payments are paid as intended.

**Re-audit comment**

Resolved.

From client:

Added a control function so that the new payees can only be added before release is called. Release is blocked until we stop allowing new payees

### Claimable soft clay amount can be claimed unlimited times.
**Description**

SoftClayContract.sol: function userClaimSoftClay(). After a user has claimed soft clay, the value from mapping_claimableSoftClay is not set to zero, which means that user can claim the same soft clay multiple times.

**Recommendation**

Set the value for mapping_claimableSoftClay[passportld] to zero after clay is claimed.

**Re-audit comment**

Resolved.

From client:

Fixed, now claimable amount is adjusted to zero once claimed.

## High Risk

### Unable to access animations
**Description**

AccessTokenContract.sol

ImageData.sol

Users will not be able to access animations at all. First, Navigator and Pioneer level animations have no way to set them anywhere (neither in the constructor nor in the setter)._navAnnimation and_pionAnnimation never initialized, but, used in getAnnimation ForElement

Secondly, when updating the passport rank to the level of the legend, the animation is not set.

**Recommendation**

Initialize the animations of the Pioner and Navigation levels, or remove them by making a stub in the getter. Change the updateRank method by adding an animation setting to it, or make a separate setter, depending on the needs of the business logic.

**Re-audit comment**

Resolved.

From client:

This is now added: so the animation is added only once and will not change when the passport level changes. A function is added to the Image Contract to set the animation.

### Users can get more passports per account than set in the limit.
**Description**

In the contract, we have the_maxAllowedPerWallet parameter, which is responsible for the limit of passports per wallet. Verification is carried out in

File:

AccessTokenContract.sol 181: function_internalMint

require(balanceOf(toWallet) <_maxAllowedPerWallet, "address already owns max allowed");

The user can bypass this limitation by following a simple scenario:

The user mints the token. The user transfers the token to another address. The user mints the token again. This is especially important if the user was given the opportunity to mint for free.

**Recommendation**

Since the ability to transfer/sell a passport to another user should be available, it may be necessary to either keep a list of users who are issued a passport, or limit the transfer by making the sale a separate controlled method.

**Re-audit comment**

Verified.

From client:

Metropolis team understands this risk and it is allowable under our model. So users are allowed to mint, transfer and mint again. Also contracts allow users minting and buying on the secondary market. Given the current market the team will likely increase this limit anyway. Broadly this is an artificial constraint to encourage urgency.

Post-audit:

There is a risk that a user (especially a user with a free mint) may overmint the entire passport limit.

From client:

Additional checks added

### No money back.
**Description**

AccessTokenContract.sol, paidMint() and referralMint()

There is no handling in case the user pays more than required. Thus extra funds are not returned to the user.

**Recommendation**

Add the processing of this case, returning the balance of the necessary funds to the user. Be careful and consider the reentrancy vulnerability.

**Re-audit comment**

Resolved.

Post-audit:

1. Add processing to the Metropolis World Passport.referral Mint method

2. Move storage changes to the end of the function.

3. Use function safeTransferETH or Address.sendValue() instead of transfer

### Eternal free mint.
**Description**

AccessTokenContract.sol

There is no function to prevent the user from free minting. If the user is allowed a free mint, this is an endless, free opportunity to mint passports with increased winChance. Especially given the real absence of a limit on the limit, as well as bulkMint - this is a potential opportunity to bypass the entire business logic for payment very easily.

**Recommendation**

Consider proper management of enabling users to mint for free. Add a fully functional restriction (at least for a free mint), or at least add a function to take away the user's ability to freely mint.

**Re-audit comment**

Resolved.

From client:

Eternal free mint now removed with the introduction of a free mint count check.

### Not consolidated data
**Description**

SoftClay.sol, setPioneerLevel(), setLegend Level()

Consider carefully the situation with changing the levels of required softClay. For example, if quantity 100 is needed for the Pioneer level first, and the user has 105 softClay units -> the Pioneer rank is assigned. Then you change the limit, and set the required level bar to 200. Verify if the user should stay at the Pioneer level in this case. Issue is set as High-risk, since there are no methods to update all fields as required.

**Recommendation**

Verify the functionality of changing levels of required softClay

**Re-audit comment**

Verified.

From client:

The Metropolis team has confiremed that it is allowable for the update of passport rank to occur only when the user next redeems or adds to their softClay balance.

### Referral minter can pass his future token id to receive referral points.
**Description**

AccessTokenContract.sol: function referralMint(). Message sender can pre compute a newItemid, which will be minted to him and pass it as referrer Tokenld, thus receive referral points for himself. Also, having the opportunity to have several passports - the user can specify his referrerTokenld from his other passport.

**Recommendation**

Verify that referrerTokenld exists and is not equal to newitemid.

**Re-audit comment**

Resolved

## Medium Risk

### The ability to increase winChance without participating in the lottery.
**Description**

WinChancesContract.sol, updateAfterLoss()

If the user was not a lottery participant (for example, he joined later), he can still raise his winChance in the same way as if he lost in each of them.

**Recommendation**

Verify the functionality of late users joined to the lottery.

**Re-audit comment**

Resolved.

From client:

Added contingency functions for this. So the team can manually stop the increasemet when we feel enough time has passed.

Post-audit:

Note that the protocol runs the lottery at the city level - not the buildings. That is, if the protocol blocks the lottery claim, and holds some kind of draw after a long time again in this city, this new lottery will open access to winChance increasement for new users by the amount equal to the number of buildings already sold in this city.

### Updater without an admin role cannot add roles.
**Description**

AccessToken Contract.sol, WinChances.sol, WLContract.sol: function

addTheRoles ForContractCalls(). Function calls grantRoles() function from AcessControl, which checks that the message sender is the default admin. Thus, in case Updater doesn't have a default admin role, he won't be able to call addTheRoles ForContractCalls(). If you assign UPDATER_ROLE to anyone other than the creator of the contract, that user will not be able to call these methods.

**Recommendation**

Call internal function_grantRole() to avoid checking the default admin role.

**Re-audit comment**

Resolved

### Not profitable standard mint
**Description**

AccessTokenContract.sol

There are two methods - paidMint and referralMint. Both methods are public. But in the referral Mint method, the user sets himself more winChance (2 instead of 1), while you can specify any passportid, since this information is public.

**Recommendation**

Depending on the needs of your business logic, you may want to make winChance the same in both cases. Or put a restriction on the use of referral Mint, for example, by pre-mapping sponsor/referral. This approach will require more spending of users' money - but it will keep the business logic. Perhaps there are more options. Thus the verification is required.

**Re-audit comment**

Acknowledged.

From client:

The team is acknowledged of this and the concerns. From the business perspective Metropolis team allows the use case where a user chooses to use the referral mint instead of paid mint. The team can protect this a little on the front end and allows those who bypass the frontend to do this.

### Not equal softClay amount
**Description**

AccessTokenContract.sol

The updateRank method does not take into account cases where softClay is equal to any of the levels.

**Recommendation**

Add variant handling when softClay is equal to levels.

**Re-audit comment**

Resolved

### Not working method bulkMint.
**Description**

AccessTokenContract.sol function bulkMint()

The bulkMint method sequentially calls the_internalMint method, which in turn checks the

user's passport balance limit. That is, calling BulkMint with the number of -5, but with a limit of 1 - the transaction will be terminated on the second call to_internalMint.

**Recommendation**

Remove and redo the constraint (balanceOf(toWallet) <_maxAllowedPerWallet inside the _internalMint method

**Re-audit comment**

Resolved.

From client:

Added a check within bulk mint to ensure the number of mints will not break flow.

Post-audit:

function bulkMint

If you pass numberOfMints greater than mintLimit or maxAllowedPerWall then these checks will overflow.

require(balanceOf(msg.sender) <_maxAllowedPerWallet - numberOfMints, "address already owns max allowed");

require(_tokenlds.current() <_mintLimit-numberOfMints, "Will take you over max supply");

Thanks to the built-in overflow check in solidity ^8.0, it will throw an error, but since these are user-defined functions - it is recommended to add a check for (numberOfMints < maxAllowedPerWall), with the full text of the error so that users understand the problem.

So, the check for (_tokenlds.current() <_mintLimit-numberOfMints) can be replaced with (_tokenlds.current() + numberOfMints - 1 <= mintLimit)

## Low Risk

### Soft Clay limit
**Description**

There is a comment in the code indicating that softClay has a limit of 4 billion, but there are no checks for this. uint32 is a little bit more than 4 billion.

File:

AccessTokenContract.sol 72

uint32 softClay; // max is 4 billion

**Recommendation**

If this violates the business logic, add a limit check to the MetropolisWorldPassport.increaseSoftClay method

**Re-audit comment**

Verified.

From client:

Metropolis team don't have a business need to set the limit at 4 billion as it is expected to release far below this amount, the note was to remind of the rough upper limit of the variable type.

### Inaccurate access.
**Description**

The user can increase their winChance from both the Metropolis World Passport contract and the WinChances contract. In the case of the MetropolisWorldPassport contract, there is a check for possession of a passport - and in the other case, no.

**Recommendation**

Make access in one place, and mandatory verification.

**Re-audit comment**

Resolved

### Never used storage
**Description**

PropertyDraw._start

CityWL._avatars

MetropolisWorldPassport._wilds

Unused storage removal will decrease gas usage during the deployment and will increase the code quality.

**Recommendation**

remove unused storage.

**Re-audit comment**

Resolved

### Pragma version
**Description**

Use the latest (if possible) version of pragma. Avoid floating pragmas for non-library contracts and use the fixed version of the latest stable release. As for now the latest stable version is 0.8.15. Usage of the floating point version increases risk to get the unstable release (as it was for several versions in 0.8.x line).

**Recommendation**

use explicitly Solidity 0.8.15

**Re-audit comment**

Unresolved

## Informational

### Centralized solution and great influence of manual control
**Description**

Most of the functionality depends on the manual management of UPDATE_ROLE. Because of this, there is a high probability of making an error outside the contract part, which will get into the blockchain, and if it does not destroy the system, then at least it will lead to data inconsistency. Thus, the information about the role system should be present in the report.

**Recommendation**

Add the description of the roles system to the documentation, make it clearly visible for users in the documentation about the admin functions, consider usage of the DAO or decentralized keepers.

**Re-audit comment**

Acknowledged.

From client:

A series of user facing docs is in the works to cover this.

### Extra memory usage
**Description**

AccessTokenContract.sol, _internalMint()

You have:

string memory elm = elements[0];

uint $x=$ newltemld % 6;

elm = elements[x];

Better:

string memory elm = elements[newItemId % 6];

Also consider usage of the opimized code:

avatarWI: newItemId % $2==1$?_evenAvatar: _oddAvatar,

**Recommendation**

consider memory usage optimization

**Re-audit comment**

Resolved

### Unoptimized use of events.
**Description**

Check all events throughout contracts. Somewhere you have events like this:

event AvatarWLSpotAdded(string update, string avatar, uint256 tokenId);

Field "update" contains information that duplicates the event name or could be placed in the name of the event. Thus it creates extra gas spending for the string construction and extra- parameter passing. Though the event is already a self-describing object.

**Recommendation**

Consider revising the events system.

**Re-audit comment**

Resolved

### Constants usage:
**Description**

PropertyDraw._start (contracts/VRFLottery.sol#59)

Property Draw.callbackGasLimit (contracts/VRFLottery.sol#46)

Property Draw.keyHash (contracts/VRFLottery.sol#38)

Property Draw.numWords (contracts/VRFLottery.sol#53)

Property Draw.requestConfirmations (contracts/VRFLottery.sol#49)

Property Draw.vrfCoordinator (contracts/VRFLottery.sol#33)

Consider usage of constants, since these values are set only once

**Recommendation**

Consider constants usage

**Re-audit comment**

Resolved.

Post-audit:

Contract PropertyDraw is no longer used.

### External vs public
**Description**

Use public modificator only if function can be called inside and outside contract. if it's used only from outside - use external, it's cheaper from the gas usage perspective.

• MetropolisWorldPassport.tokenURI(uint256) (contracts/AccessToken Contract.sol#461-471)

• MetropolisWorldPassport.contractURI() (contracts/AccessTokenContract.sol#139-141)

• MetropolisWorldPassport.addTheRoles ForContractCalls(address, address, address,address) (contracts/AccessToken Contract.sol#167-172)

• MetropolisWorldPassport.freeMint(address) (contracts/AccessTokenContract.sol#214-216)

• MetropolisWorldPassport.userFreeMint() (contracts/AccessTokenContract.sol#221-225)

MetropolisWorldPassport.paidMint() (contracts/AccessTokenContract.sol#230-240)

MetropolisWorldPassport.bulkMint(uint16,address) (contracts/ AccessTokenContract.sol#247-257)

• MetropolisWorldPassport.referralMint(uint256) (contracts/ AccessTokenContract.sol#263-273)

• MetropolisWorldPassport.manualAddAvatarWL(uint256, string) (contracts/ AccessTokenContract.sol#316-323)

MetropolisWorldPassport.setAvatarWLNames(string, string) (contracts/ AccessTokenContract.sol#329-335)

• MetropolisWorldPassport.userUpdateAfterLoss(uint256, string, uint32) (contracts/ AccessTokenContract.sol#338-342)

• MetropolisWorldPassport.getWinChances(uint256) (contracts/ AccessTokenContract.sol#353-356)

MetropolisWorldPassport.setFreeMinters(address[]) (contracts/ AccessTokenContract.sol#399-405)

MetropolisWorldPassport.setPrice(uint256) (contracts/AccessTokenContract.sol#407-410)

• MetropolisWorldPassport.setMaxAllowed (uint16) (contracts/ AccessTokenContract.sol#412-415)

• MetropolisWorldPassport.getMaxAllowed() (contracts/AccessToken Contract.sol#417-420)

• MetropolisWorldPassport.getDefaultAccessTokens() (contracts/ AccessTokenContract.sol#422-429)

MetropolisWorldPassport.getCurrentTokenId() (contracts/ AccessTokenContract.sol#446-452)

• MetropolisWorldPassport.setMintLimit(uint32) (contracts/ AccessTokenContract.sol#454-457)

MetropolisWorldPassport.setContractURI(string, string, string, string, string) (contracts/ AccessTokenContract.sol#482-508)

MetropolisWorldPassport.withdraw() (contracts/AccessToken Contract.sol#511-514)

ImageDataContract.setNavImages(string[], string[]) (contracts/ImageData.sol#53-64)

ImageDataContract.setPioneerlmages(string[], string[]) (contracts/ImageData.sol#66-77)

• ImageDataContract.setLegendImages(string[], string[]) (contracts/ImageData.sol#78-89)

• ImageDataContract.setAnnimations(string[]) (contracts/ImageData.sol#90-100)

• PaymentSplitPassport.getBalance() (contracts/PaymentSplit.sol#67-69)

• PaymentSplitPassport.totalShares() (contracts/PaymentSplit.sol#74-76)

PaymentSplitPassport.shares(address) (contracts/PaymentSplit.sol#96-98)

• PaymentSplitPassport.payee(uint256) (contracts/PaymentSplit.sol#118-120)

PaymentSplitPassport.releaseAll() (contracts/PaymentSplit.sol#145-150)

• PaymentSplitPassport.release(IERC20, address) (contracts/PaymentSplit.sol#157-170)

• PaymentSplitPassport.addPayee (address, uint256) (contracts/PaymentSplit.sol#189-191)

• SoftClay.awardSoftClay(uint256, uint32) (contracts/SoftClayContract.sol#37-41)

• SoftClay.userClaimSoftClay (uint256) (contracts/SoftClayContract.sol#43-49)

SoftClay.addClaimableSoftClay(uint256[], uint32[]) (contracts/SoftClayContract.sol#51-55)

• SoftClay.getClaimableClay (uint256) (contracts/SoftClayContract.sol#57-59)

SoftClay.setPioneerLevel(uint32) (contracts/SoftClayContract.sol#71-74)

• SoftClay.setLegend Level(uint32) (contracts/SoftClayContract.sol#76-79)

• SoftClay.getPioneerLevel() (contracts/SoftClayContract.sol#81-84)

• SoftClay.getLegendLevel() (contracts/SoftClayContract.sol#86-89)

• PropertyDraw.addTheRolesForContractCalls(address, address) (contracts/ VRFLottery.sol#36-39)

• PropertyDraw.setPartnerContracts(address, address) (contracts/VRFLottery.sol#41-46)

• CityWL.addThe Roles For ContractCalls(address, address) (contracts/WLContract.sol#58-61)

• CityWL.removeCityWISpot(uint256, string, uint32) (contracts/WLContract.sol#78-83)

• WinChances.addTheRoles ForContractCalls(address, address) (contracts/ WinChances Contract.sol#53-56)

• WinChances.setLossIncrease(uint16) (contracts/WinChancesContract.sol#83-87)

• WinChances.setReferralIncrement (uint16) (contracts/WinChances Contract.sol#89-93)

• WinChances.setWinDecrease (uint16) (contracts/WinChances Contract.sol#95-99)

WinChances.closeCityForWin Loss Redemption(string) (contracts/ WinChances Contract.sol#113-115)

**Recommendation**

Change the modifier in the specified methods to external if these methods are not going to be called within this contract.

**Re-audit comment**

Unresolved.

Post-audit:

Left unchanged:

• MetropolisWorldPassport.tokenURI (uint256) (contracts/AccessTokenContract.sol#447-457)

• MetropolisWorldPassport.contractURI() (contracts/AccessToken Contract.sol#123-125)

• MetropolisWorldPassport.setMintLimit(uint32) (contracts/ AccessTokenContract.sol#440-443)

• PaymentSplitPassport.payee(uint256) (contracts/PaymentSplit.sol#119-121)

• PaymentSplitPassport.releaseAll() (contracts/PaymentSplit.sol#147-153)

• PaymentSplitPassport.addPayee(address, uint256) (contracts/PaymentSplit.sol#182-185)

Property Draw.addTheRolesForContractCalls(address, address) (contracts/ VRFLottery.sol#37-42)

Property Draw.setPartnerContracts(address,address)

### Missing zero-address checks.
**Description**

There are no checks for a zero address when initializing contracts.

MetropolisWorldPassport - constructor, setWLContractAddress

SoftClay - constructor

PropertyDraw - setPartnerContracts

WinChances - constructor

CityWL - constructor

**Recommendation**

You should add checks for the zero address when initializing the contracts and payment addresses.

**Re-audit comment**

Resolved

### Calldata vs memory
**Description**

Use calldata for all input parameters instead of memory. Use memory only if you change this parameter inside the function.

ImageDataContract

• setNavlmages (string[] memory image, string[] memory cdnimage)

• setPioneerImages(string[] memory ipfs, string[] memory cdn)

• setLegendImages (string[] memory ipfs, string[] memory cdn)

• setAnnimations (string[] memory animations)

**Recommendation**

Change the modifier in the specified methods to calldata.

**Re-audit comment**

Resolved

### Re-inheritance
**Description**

Don't need to inherit Metropolis World Passport from ERC721, because ERC721 Enumerable already inherits from ERC721.

**Recommendation**

Remove inheritance from ERC721 in MetropolisWorld Passport contract.

**Re-audit comment**

Resolved

### String alternative
**Description**

Consider using an enum instead of an array of strings for elements and avatar. Using string is always more vulnerable to management errors and takes up more memory.

Potential replacement variables:

File: AccessToken Contract.so

64: string private_oddAvatar = "nomad"; 65: string private_evenAvatar = "citizen";

could be: enum Avatars{ Nomad, Citizen }

• 84: string[] elements = ["Fire", "Water", "Air", "Space", "Pixel", "Earth"];

could be: enum Elements{ Fire, Water, Air, Space, Pixel, Earth }

ranks in MetropolisWorld Passport could be: enum Ranks{ Navigator, Pioneer, Legend, }

**Recommendation**

Replace parameters of type string with elements of structures of type enum.

**Re-audit comment**

Unresolved.

From client:

Avatars will change over time, so needs to remain updatable.

### Unnecessary private fields, duplicated getters
**Description**

You have private fields in your contacts for which there are public getters. If you do not plan to inherit from these contracts, then you can probably remove unnecessary getters and make the variables public. Solidity will automatically create a public getter for such public variable. There are also public variables for which you also wrote a direct getter. Basically, it's code duplication. Thus following the recommendation will reduce the contract size and increase readability.

Unnecessary private fields:

MetropolisWorldPassport._contractURI

MetropolisWorldPassport._maxAllowedPerWallet

PaymentSplitPassport._totalShares

PaymentSplitPassport._totalReleased

PaymentSplitPassport._erc20TotalReleased

PaymentSplitPassport._shares

PaymentSplitPassport._released

PaymentSplitPassport._erc20Released

PaymentSplitPassport._payees

SoftClay._pioneerLevel

SoftClay._legendLevel

PropertyDraw._drawsDone

WinChances._citiesDropped

Duplicated getters:

WinChances.getReferalIncrease

**Recommendation**

Reconsider whether you need a private modifier in all these fields. Put the public modifier where possible, and remove the manual getter.

Consider removing manual getters for public variables.

**Re-audit comment**

Unresolved

### Unreleased ERC20
**Description**

The PaymentSplitPassport contract contains methods for ERC20 tokens, but the general scheme for managing funds in other contracts (MetropolisWorldPassport) does not have the ability to buy for ERC20, as well as send ERC20 to the PaymentSplitPassport contract. If this is not your idea and the system will not use the ERC20 token in the future (without deploying a new PaymentSplitPassport contract), then you should remove unnecessary functions and variables.

File: PaymentSplitPassport.sol

_erc20TotalReleased

_erc20Released

function release (IERC20 token, address account) public virtual

function totalReleased (IERC20 token) public view returns (uint256)

function released (IERC20 token, address account) public view returns (uint256)

**Recommendation**

Consider removing the specified methods and variables or verify the correctness of ERC20 tokens usage.

**Re-audit comment**

Unresolved

### Check that the length of array parameters matches.
**Description**

SoftClayContract.sol: function addClaimableSoftClay(). The length of passportids and amounts should be verified to match each other. Such

validations are helpful for ease of error navigation.

ImageData.sol: function setNavlmages()

Length of image and cdnImage should be verified to match each other

ImageData.sol: function setPioneerlmages(), function setLegendImages()

Length of ipfs and cdn should be verified to match each other.

Raising these events will raise an error if the parameters are an empty array.

ImageData.sol:

emit NavlmagesUpdated (image[0], cdnImage[0]);

emit PionImagesUpdated(ipfs[0], cdn[0]);

emit LegImagesUpdated (ipfs[0], cdn[0]) ;

emit AnimationsUpdated(animations[0]);

**Recommendation**

Validate that length of specified arrays matches. Add a check for parameter equality to empty arrays.

**Re-audit comment**

Resolved

### Code optimization
**Description**

File: ImageData.sol

function getAnnimation ForElement(string calldata element, uint16 level)external view returns(string memory){

if ( $(level==1)${

return_navAnnimation [element];

}else if $(level==2)\{$

return_navAnnimation [element];

}else{

return_navAnnimation [element];

}

}

Better:

function getAnnimation ForElement(string calldata element, uint16 level)external view returns(string memory){

}

return_navAnnimation [element];

**Recommendation**

Change the code to be more economical and shorter.

**Re-audit comment**

Resolved
