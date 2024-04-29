**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## High Risk

### [H-01] Last NFT from the supply can't be minted

**Severity**

**Impact:**
Medium, as only one NFT won't be available for minting, but this is value loss to the protocol

**Likelihood:**
High, as it's impossible to mint the last NFT

**Description**

Currently both the `mint` and `mintPhysical` methods have the following check:

```solidity
if (nextId + amount >= MAX_SUPPLY) revert ExceedsMaxSupply();
```

This is incorrect, as even when the `nextId` is `MAX_SUPPLY - 1` then an `amount` of 1 should be allowed but with the current check the code will revert. This is due to the `equal` sign in the check, which shouldn't be there. Here is a Proof of Concept unit test demonstrating the issue (add it to `MuseumOfMahomes.t.sol`):

```solidity
    function testNotAllNFTsCanBeMinted() public {
        museum.setPrice(PRICE);
        uint256 allButOneNFTSupply = 3089;

        // mint all but one from the NFT `MAX_SUPPLY` (3090)
        museum.mint{value: allButOneNFTSupply * PRICE}(address(this), allButOneNFTSupply);
        require(allButOneNFTSupply == museum.balanceOf(address(this)), "Mint did not work");

        // try to mint the last NFT from the supply, but it doesn't work
        vm.expectRevert(MuseumOfMahomes.ExceedsMaxSupply.selector);
        museum.mint{value: PRICE}(address(this), 1);
    }
```

**Recommendations**

Do the following change in both `mint` and `mintPhysical`:

```diff
- if (nextId + amount >= MAX_SUPPLY) revert ExceedsMaxSupply();
+ if (nextId + amount > MAX_SUPPLY) revert ExceedsMaxSupply();
```

## Low Risk

### [L-01] Reveal and Redeem should only be set to `true`

Currently the `setRevealOpen` and `setRedeemOpen` methods allow setting the values to both `true` and `false` as many times as the `metadataOwner` decides to. This shouldn't be the case, as both should only be available to set to `true` just once, and never to `false` after this. Change the setters to methods that only set the values to `true`, removing the parameters from the methods.

### [L-02] All state-changing methods should emit events

Currently most of the state-changing methods in the `MuseumOfMahomes` contract do not emit an event. An example is the `setPrice` method, which might be important for users or front-end/UI clients that wish to monitor and track the current price of the NFTs. Add proper event emissions in all state-changing methods.

### [L-03] A `treasury` account can mint all NFTs

Currently an account that is in the `treasury` mapping can mint all NFTs for free. While it is desired that such an account does not pay for minting a token, consider adding a `MAX_TREASURY_MINTS` upper bound to limit the count of NFTs minted by `treasury` accounts. You can also make sure that when a `treasury` account is minting, the `msg.value` is 0.

### [L-04] Contract is not working as a state machine

Currently it is possible for the `metadataOwner` to set the `redeemOpen` value to `true` while the `revealOpen` hasn't been set to `true` yet. There should be a sequence/flow of how the contract works - first minting, then revealing, then redeem (or redeem right after reveal). Allow setting `redeemOpen` to `true` only if `revealOpen == true`, and also allow setting `revealOpen` to `true` only when mint is completed (`totalSupply == MAX_SUPPLY`).

### [L-05] Use a two-step access control transfer pattern

The `MuseumOfMahomes` contract uses a single-step access control transfer pattern in `setOwner` and `setMetadataOwner`. This means that if the current `owner` or `metadataOwner` accounts call the methods with an incorrect address, then those roles will be lost forever along with all the functionality that depends on them. Follow the pattern from OpenZeppelin's [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol) and implement a two-step transfer pattern for the actions.