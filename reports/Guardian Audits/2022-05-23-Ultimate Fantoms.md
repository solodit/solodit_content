**Auditors**

[Guardian Audits](https://twitter.com/guardianaudits)

# Findings

## Medium Risk

### UF-1 | Centralization Risk

**Description**

The `owner` address, `0x3e522051a9b1958aa1e828ac24afba4a551df37d`, is not a multi-sig and has potentially dangerous permissions for `renounceOwnership`, `transferOwnership`, `setRoyaltyAddress`, `setSpiritRouter`, `updatePaintRouter`, `setBaseURI`, `setMintSize`, `sweepEthToAddress`.

**Recommendation**

Make the `owner` a multi-sig and/or introduce a timelock for improved community oversight.

**Resolution**

Ultimate Fantoms: Acknowledged, contract ownership will be changed to the multisig at
`0x87f385d152944689f92Ed523e9e5E9Bd58Ea62ef`.

### UF-2 | Denial-of-Service With Failed Call

**Description**

`publicMint` relies on multiple external calls which can fail accidentally or deliberately. If just one consistently fails, users will not be able to mint.

**Recommendation**

Isolate external calls to another transaction(s). `wFTM` allocations could be distributed with a pull-over-push pattern.

**Resolution**

Ultimate Fantoms: Acknowledged, failed transactions can be resubmitted + the chance of a failed call is low.

### UF-3 | Random Manipulation

**Description**

The `random` function relies on weak sources of pseudo-randomness from only on-chain attributes. A
validator node can manipulate the `block.timestamp` and therefore the random number. Therefore, the
`_sendTo` address can be manipulated in favor of the validator.

**Recommendation**

Utilize the Randomness pattern to obtain on-chain randomness and avoid validator manipulation or
obtain random numbers off-chain through an oracle.

**Resolution**

Ultimate Fantoms: Acknowledged in source code.

### UF-4 | Mint Failure

**Description**

In `publicMint`, when performing `_earnTo = random() % (_tokenIdCounter.current() +1)`, there is a
possibility `_earnTo` is equivalent to `_tokenIdCounter.current()` which yields a `tokenID` for a token that
does not exist yet. Therefore, the subsequent call to `ownerOf` will fail and the mint will revert.

**Recommendation**

Perform `random() % _tokenIdCounter.current()`.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion.

### UF-5 | Price Inconsistency

**Description**

In the `getPrice` function the stepwise price jumps do not account for the following `tokenIds`: 101,
301, 601, 1001, 1501, and 2301.
This is because each if statement utilizes > instead of >= when referring to these tokenIds.Therefore
a mint for one of these `tokenIds` will mistakenly go to the else branch and charge 6 `FTM`.

**Recommendation**

Use `>=` or decrement the lower boundaries by one.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion.

## Low Risk

### UF-6 | Using .transfer

**Description**

`transfer()` comes with a fixed amount of gas.

**Recommendation**

Utilize `call()` with a success check.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion.

### UF-7 | Inaccurate Comments

**Description**

On line 1712: `// random number between 0 to 4` is inaccurate as `_toEarn` takes on a random value of 0 or 1.
Additionally, on line 1672: `// 10%` is inaccurate as `_rndmAlloc` is calculated to be 15%.

**Recommendation**

Refactor comments to accurately reflect the code.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion.

### UF-8 | Unnecessary Code

**Description**

Several functions such as `setMintFees`, `enableMinting`, `disableMinting`, `setTeamMinting`, `minterTeamMintsRemaining`, and `minterTeamMintsCount` serve no purpose.
In addition, variables such as `enableMinter`, `_earnAmount`, `_mintFees`, `_teamMintSize`, `RNDM_TOKEN`, `bePATH1`, `bePATH2`, `BEETS_TOKEN`, `bPath1`, `bPath2`, `BRUSH_TOKEN`, `SPIRITSWAP_TOKEN`, `wCYBERs`, `wFTMOPRs`, `OPR`, `_beetsAlloc`, `_treasuryAlloc`, `_dfyAlloc`, and `_teamMintCounter` go unused.

**Recommendation**

Remove unused code.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion

### UF-9 | Repetitive Function Calls

**Description**

In `publicMint` the `random` function is called several times, even though the random value is constant during each tx.

**Recommendation**

Compute the random value once.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion.

### UF-10 | Mint Fee Manipulation

**Description**

Because the `getPrice` function is stepwise, anyone can mint 10 tokens as if they were all in a lower cost bracket while potentially only 1 was.

**Recommendation**

Compute the mint fee for each token, account for mints that traverse the fee increase, or accept the manipulation.

**Resolution**

Ultimate Fantoms: Acknowledged, protocol loss will be minimal if exploited.

### UF-11 | Arbitrary Max Supply

**Description**

The `owner` address has permissions to arbitrarily `setMintSize` which can drastically affect the tokenomics of the project.

**Recommendation**

Remove the `setMintSize` function or appropriately timelock it for community trust and safety.

**Resolution**

Ultimate Fantoms: Acknowledged, contract ownership will be changed to the multisig at `0x87f385d152944689f92Ed523e9e5E9Bd58Ea62ef`.

### UF-12 | Unequal Minting Rewards

**Description**

In `publicMint` the rewards distribution to `CYBERs` holders can only occur on the first mint and never
after.

**Recommendation**

Ensure this is the expected behavior. If it isn’t, refactor the reward logic to more fairly include `CYBERs` holders.

**Resolution**

Ultimate Fantoms: Resolved.

### UF-13 | Mutability Modifiers

**Description**

The `_beetsAlloc`, `_dfyAlloc`, `_treasuryAlloc`, mintFees, and `_earnAmount` variables are never modified, and should therefore be declared `constant`.

**Recommendation**

Declare them as `constant`.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion where appropriate.

### UF-14 | Function Visibility Modifiers

**Description**

The functions `setRoyaltyAddress`, `updateSpiritRouter`, `updatePaintRouter`, `publicMint`, `setMintFees`, `enableMinting`, `disableMinting`, `setBaseURI`, `setTeamMinting`, `setMintSize`, and `sweepEthToAddress` are marked as public, but are never called from inside the contract.

**Recommendation**

These functions can be marked `external` for gas optimization.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion where appropriate.

### RS-1 | Unnecessary Code

**Description**

The `_earnAmount` variable goes unused.

**Recommendation**

Remove unused code.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion.

### RS-2 | Unequal Royalty Rewards

**Description**

In the `receive` and `fallback` functions, the rewards distribution to `CYBERs` holders can only occur before the second mint and never after.

**Recommendation**

Ensure this is the expected behavior. If it isn’t, refactor the reward logic to more fairly include `CYBERs` holders.

**Resolution**

Ultimate Fantoms: Resolved.

### RS-3 | Repetitive Function Calls

**Description**

In the `receive` and `fallback` functions the `random` function is called several times, even though the random value is constant during each tx.

**Recommendation**

Compute the random value once.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion.

### RS-4 | Mutability Modifiers

**Description**

The `SPIRITSWAP_ROUTER` and `_earnAmount` variables are never modified, and should therefore be declared `constant`.

**Recommendation**

Declare them as `constant`.

**Resolution**

Ultimate Fantoms: Resolved, applied suggestion where appropriate.
