**Auditors**

[Guardian Audits](https://twitter.com/guardianaudits)

# Findings

## High Risk

### NTE-1 | Incorrect Fee Structure

**Description**

The `fees` variable is incremented whenever `_redeemEth` or `_redeemWstEth` is called. However, the `withdrawFees` function treats `fees` as if it were entirely in Ether. Therefore, when `fees` is incremented in the `_redeemWstEth` function it is wrongly attributed as Ether, rather than `wstEth`.
Because of the misattribution, once `withdrawFees` is called the contract will no longer have enough Ether to back all of the regular Ether ethernotes, rendering some holders of regular Ether ethernotes unable to redeem and causing complete loss of funds for those users.
Notably, a malicious actor could front-run a call to `withdrawFees` to mint and redeem `wstEth` ethernotes, causing this imbalance and complete loss of funds for some regular Ether ethernote holders.

**Recommendation**

Track Ether fees and `wstEth` fees separately or find some fee structure that maintains the backing value of all ethernotes.

**Resolution**

Ethernote Team:

- Add a new variable `feesWstEth`
- Add a new function `withdrawWstEthFees`

## Medium Risk

### GLOBAL-1 | Centralization Risk

**Description**

The `owner` for `ethernote2` and `ethernotePresale` has authority over many functions that may be used to negatively disrupt the project.
Some possible attack vectors include:

- Ceasing the available notes to prevent minting
- Updating an edition’s validator status to prevent minting
- Calling `approveWstEth` and withdrawing all available wstETH
- Reentering on `withdrawFees` until the contract’s Ether balance is drained
- Updating a note’s `redeemFee` to 100% as it is not capped
- Updating a note’s edition index which would prevent minting due to an out-of-bounds error
- Setting the `ethernote` address in the `EthernotePresale` to siphon user funds

**Recommendation**

Ensure that the `owner` is a multi-sig and/or introduce a timelock for improved community oversight. Optionally introduce `require` statements to limit the scope of the exploits that can be carried out by the `owner`.

**Resolution**

Ethernote Team:

- Add reentrancy guard on `withdrawFees` function
- Remove `denomination` and edition being changeable from `UpdateNote` function
- Add constant MAX_FEE variable - Set currently to 501 (5%) , Add require check in `updateNote` and `createNote` - See NTE-3
- Updating index is not possible - the ID is just used to change that ID's attributes not the actual ID
- `owner` will also be a multi-sig after deployment with function `transferOwnership` function via OpenZeppelin's `Ownable.sol` contract
- Although the contract has an ACTIVE OWNERSHIP, once a note is minted, there is no ability for us to pause redemption, only new minting

### NTE-2 | Dangerous Edition Update

**Description**

The `owner` can update the `ethereum` field for an edition that has already been minted. This can lead to a state where users are shifted to a different redemption method but there are not enough funds for all users to redeem their ethernote, causing complete loss of funds for some ethernote holders.
Notably, a transaction to update the `ethereum` field can be sandwich attacked by a malicious actor to mint notes of the edition with one token before the `ethereum` value is updated and redeem their notes after for a different token. A malicious sandwich attack would leave some ethernotes unbacked by their respective token.

**Recommendation**

Only update the `ethereum` field when minting is paused and properly shift funds between Ether and `wstEth` to ensure all users can still redeem.

**Resolution**

Ethernote Team:

- Remove `_ethereum` Boolean in `updateEdition` to prevent changing type of note.

## Low Risk

### NTE-3 | Setting Default Values

**Description**

The variables `circulatingNotes` and `fees` are assigned to 0. This is unnecessary because the default value for the `uint256` type is 0.

**Recommendation**

Remove the assignment.

**Resolution**

Ethernote Team:

- Remove the 0 assignment as `uint256` defaults to 0

### NTE-4 | Uncapped Values

**Description**

In the functions `createNote` and `updateNote`, the parameter `_redeemFee` is not capped. `_redeemFee` can be be arbitrarily set to siphon user funds and even exceed 100% to entirely prevent redemption.

**Recommendation**

A `require` statement can be added to cap the note’s `redeemFee` at a reasonable rate.

**Resolution**

Ethernote Team:

- `createNote` added `require(_redeemFee < MAX_FEE, "[x] Max fee exceeded");`
- `updateNote` added `require(_redeemFee < MAX_FEE, "[x] Max fee exceeded");`
- `ceaseNote` has no effect on `_redeemFee`

### NTE-5 | Visibility Modifiers

**Description**

The functions `getTokenEditionName`, `getTokenEditionId`, `getTokenNoteId`, `getTokenBalance`, `getEditionsLength`, `getNotesLength`, `totalPrinted`, and `notesCirculating` are marked as `public`, but are never called from inside the contract.

**Recommendation**

Modify their visibility from `public` to `external`.

**Resolution**

Ethernote Team:
Modified visibility from public to external for the following functions:

- `getTokenEditionName`
- `getTokenEditionId`
- `getTokenNoteId`
- `getTokenBalance`
- `getEditionsLength`
- `getNotesLength`
- `totalPrinted`
- `notesCirculating`

### NTE-6 | Memory Modifiers

**Description**

The functions `onERC721Received`, `updateEdition`, and `createEdition` have arguments marked as `memory` when they could be declared `calldata`.
Changing the memory modifiers from `memory` to `calldata` will demand more gas upon deployment but less gas for each call to `onERC721Received`, `updateEdition`, and `createEdition`.

**Recommendation**

If it is desired to reduce gas for these functions at the expense of deployment costs, change the memory modifiers from `memory` to `calldata`.

**Resolution**

Ethernote Team:

- Changed from `memory` to `calldata`

### PSL-1 | Pull Over Push Fees

**Description**

In the `mintSilver` and `mintGold` functions a minting fee is charged and immediately sent to the `payout` address. In the event that the `payout` address was set to a malicious contract it could stop minting by reverting upon call, resulting in a denial of service for minting.

**Recommendation**

Adopt a pull-over-push pattern for charging fees and rely on the `withdrawAll` function to collect fees.

**Resolution**

Ethernote Team:

- No Change

### PSL-2 | Boolean Redundancy

**Description**

Modifiers `hasWhitelistGold` and `hasWhitelistSilver` use `whitelistedGold[msg.sender] == true` and `whitelistedSilver[msg.sender] == true` respectively. This is redundant since the mapping values are booleans themselves.

**Recommendation**

Remove the unnecessary boolean assertion.

**Resolution**

Ethernote Team:

- Remove unnecessary Boolean assertion

### PSL-3 | Unchecked Return Value

**Description**

The `withdraw` function uses `transfer` which provides a return value that should be checked. Not all ERC20 implementations revert in case of failure, so it is important to have some logic in the event these executions fail.

**Recommendation**

Check the return value, or opt for a `safeTransfer` alternative.

**Resolution**

Ethernote Team:

- Not expecting any ERC20 tokens in this contract, only recovering ERC20 tokens to this address

### PSL-4 | Immutability Modifiers

**Description**

The `ETH_01`, `ETH_05`, `ETH_10`, `ETH_100`, and `ETH_FEE` variables are never mutated, and should therefore be declared `constant`.

**Recommendation**

Declare them as `constant`.

**Resolution**

Ethernote Team:

- Changed to recommended
- No function exists to `change` these variables

### PSL-5 | Visibility Modifiers

**Description**

The `withdrawAll` function is marked as `public`, but is never called from inside the contract.

**Recommendation**

Modify the visibility of `withdrawAll` from `public` to `external`.

**Resolution**

Ethernote Team:

- Changed `withdrawAll` function to `external`

### PSL-6 | Comparison Inefficiency

**Description**

The `require` statements in `addMultipleWhitelistSilver` and `addMultipleWhitelistGold` use a `<=` operator, while a `<` operator combined with incrementing the right hand side of the comparison is a more gas efficient alternative.

**Recommendation**

Utilize a `<` operator and increment the right hand side of the comparison.

**Resolution**

Ethernote Team:

- Change `<=` to `<`
