** Auditors**

[Shieldify Security](https://x.com/ShieldifySec)

---

# Findings

## High Risk

### [H-01] Anyone Can Burn Any NFTs of Other Users in the `Material`, `UGC` and `Craft` Objects

**Severity**

High Risk

**Description**

The `MaterialObject` and `CraftObject` contracts represent distinct digital assets and the corresponding materials required for their creation, respectively. These assets can take various forms, such as in-game items or characters.

The same principle applies to the `UGCObject` contract, which enables users to generate their unique assets known as User Generated Content (UGC). These UGCObjects serve as materials and catalysts in `CraftLogic` recipes and are integrated into the Phi ecosystem.

The vulnerability permits arbitrary users to destroy NFTs from any account, potentially resulting in the unauthorized loss of these digital assets.

Based on the provided code, it appears that anyone can invoke the `burnObject` and `burnBatchObject` functions while specifying any account address from which to burn NFTs (digital assets). In a secure system, only the account holder should possess the ability to burn their own tokens. This creates a vulnerability whereby malicious users can burn NFTs belonging to other users without their consent.

**Location of Affected Code**

```solidity
// Function to burn an object
function burnObject(address from, uint256 tokenid, uint256 amount) external override {
    super._burn(from, tokenid, amount);
}

// Function to burn a batch of objects
function burnBatchObject(address from, uint256[] memory tokenids, uint256[] memory amounts) external override {
    super._burnBatch(from, tokenids, amounts);
}
```

**Recommendation**

To address this vulnerability, it is crucial to implement a check in the code to verify that the caller is the rightful owner of the NFT intended for burning. This verification ensures that only the NFT holder has the authority to burn their own NFTs, effectively preventing unauthorized users from burning NFTs they do not possess.

- If the function `burnObject` is called only external. Remove the `address from` parameter from `burnObject` function and put `msg.sender` as a parameter of `super._burn` function. The same goes for the `burnBatchObject` function.

`Example:`

```solidity
function burnObject(uint256 tokenid, uint256 amount) external override {
    super._burn(msg.sender, tokenid, amount);
}
```

- If the function `burnObject` is called only from other contracts like function `MaterialObject.burnObject` is called from `CraftLogic/UGCCraftLogic` contract. Add a modifier like `onlyCraftLogic` on `MaterialObject.burnObject` function and then in `CraftLogic/UGCCraftLogic` contracts call `MaterialObject.burnObject` with the parameter `from ` as `msg.sender`. The same goes for the `burnBatchObject` function.

`Example:`

File: `src/object/MaterialObject.sol`

```solidity
function burnObject(address from, uint256 tokenid, uint256 amount) external override onlyCraftLogic {
    super._burn(from, tokenid, amount);
}
```

File: `src/CraftLogic.sol`

```solidity
IMaterialObject(material.tokenAddress).burnObject(_msgSender(), material.tokenid, material.amount);
```

Note:

If you implement our second suggestion, consider calling the `burnBatchObject` function in the `CraftLogic/UGCCraftLogic` contracts as it is currently only called externally because if a modifier is added to call this function only from the `CraftLogic/UGCCraftLogic` contracts there will be no other option to call it.

Or remove `burnBatchObject` function from the three contracts `CraftObject.sol`, `MaterialObject.sol` and `UGCObject.sol`, if you are not going to call this function in `CraftLogic/UGCCraftLogic` contracts.

**Team Response**

Acknowledged and fixed by creating a whitelist functionality for craft logic, restricting calling burn functions and removed `burnBatchObject` function from the three contracts `CraftObject.sol`, `MaterialObject.sol` and `UGCObject.sol`.

## Medium Risk

### [M-01] Signature Does Not Contain a Deadline, Making It Reusable

**Severity**

Medium Risk

**Description**

The `PhiDaily.sol` contract has the `claimMaterialObject` function that allow a user to claim a material object directly. These then call the private `_processClaim()` function to check that the coupon has been signed by the admin signer.
The `digest` variable in `_processClaim()` accepts an `eventId`, `logicId` and `msgSender()`, but it does not accept any variable that defines a timeframe in which this signature is valid and does not check if such variable has passed a certain amount of time. Without such a variable and a corresponding check for it, the issued signature essentially becomes timeless and reusable again. This is also applicabe in the other functions that call the `_processClaim()` function, namely - `batchClaimMaterialObject`, `claimMaterialObjectByRelayer` and `batchClaimMaterialObjectByRelayer` .This can negatively impact the business logic of the protocol, as one material object can be claimed more than once.

**Location of Affected Code**

File: [`src/PhiDaily.sol#L248-L258`](https://github.com/PHI-LABS-INC/PHIMaterial/blob/355376812ba1e2eeed97d5447c2afea83a3ca8f1/src/PhiDaily.sol#L248-L258)

```solidity
// Function to claim a material object.
function claimMaterialObject(
    uint32 eventid,
    uint16 logicid,
    Coupon memory coupon
)
    external
    onlyIfAlreadyClaimed(eventid, logicid)
    nonReentrant
{

function _processClaim(uint32 eventid, uint16 logicid, Coupon memory coupon) private {
    // Check that the coupon sent was signed by the admin signer
    bytes32 digest = keccak256(abi.encode(eventid, logicid, _msgSender()));
```

**Recommendation**

Consider adding a deadline check in the `claimMaterialObject()` function and an `expiresIn`/`deadline` variable either as an argument that is passed to the `claimMaterialObject()` function or in the `Coupon` struct. The `claimMaterialObject()` function should also contain a check if the `expiresIn` / `deadline` variable is valid:

`Example:`

```diff
    function claimMaterialObject(
        uint32 eventid,
        uint16 logicid,
++      uint256 expiresIn
        Coupon memory coupon
    )
        external
        onlyIfAlreadyClaimed(eventid, logicid)
        nonReentrant
    {
++     if (expiresIn <= block.timestamp){
++        revert SignatureExpired()
++     }
        _processClaim(eventid, logicid, coupon, expiresIn);
    }

    function _processClaim(uint32 eventid, uint16 logicid, Coupon memory coupon, expiresIn) private {
    // Check that the coupon sent was signed by the admin signer
++   bytes32 digest = keccak256(abi.encode(eventid, logicid, _msgSender(), expiresIn));
```

**Team Response**

Acknowledged and fixed by setting expiration period `expiresIn` to `Coupon` and added/modified tests for it.


### [M-02] Insecure Generation of Randomness Used for Token Determination Logic

**Severity**

Medium Risk

**Description**

`EmissionLogic.sol` contains the `determineTokenByLogic()` function that determines the rarity and `tokenId` of a `MaterialObject`.
It generates a `uint256 random` value that relies on variables like `block.timestamp` and `tx.origin` as a source of randomness is a common vulnerability, as the outcome can be predicted by calling contracts or validators. In the context of blockchains, the best and most secure source of randomness is that which is generated off-chain in a verified manner.

The function also uses `block.prevrandao` whose random seed calculation is by epoch basis, which means that entropy within 2 epochs is low and sometimes [`even predictable`](https://github.com/ethereum/annotated-spec/blob/master/phase0/beacon-chain.md#aside-randao-seeds-and-committee-generation). Users of PREVRANDAO would need to check that a validator has provided a block since the last time they called PREVRANDAO. Otherwise, they won't necessarily be drawing statistically independent random outputs on successive calls to PREVRANDAO.

In the context of Phi's business logic, improper insecure randomness generation could allow malicious actors to mint more rare/exclusive `MaterialObject` items.

**Location of Affected Code**

File: [`src/EmissionLogic.sol#L49`](https://github.com/PHI-LABS-INC/DailyMaterial/blob/355376812ba1e2eeed97d5447c2afea83a3ca8f1/src/EmissionLogic.sol#L49)

```solidity
uint256 random = uint256(keccak256(abi.encodePacked(block.prevrandao, block.timestamp, tx.origin)));
```

**Recommendation**

Consider using a decentralized oracle for the generation of random numbers, such as `Chainlink's VRF`. It is important to take into account the `requestConfirmations` variable that will be used in the `VRFv2Consumer` contract when implementing VRF. The purpose of this value is to specify the minimum number of blocks you wish to wait before receiving randomness from the Chainlink VRF service. The inclusion of this value is motivated by the occurrence of chain reorganizations, which result in the alteration of blocks and transactions. Addressing this concern is crucial for the successful implementation of this application on the Polygon network because it is prone to block reorgs and they happen almost on a daily basis.

Shieldify recommends setting the `requestConfirmations` value to at least 5, so that the larger portion of the reorgs that happen are properly taken into account and won't impact the randomness generation.

**Team Response**

Acknowledged, but currently will not be mitigated as the team does not have plans to implement Chainlink functionality yet.


### [M-03] Centralization Risk in Multiple Places

**Severity**

Medium Risk

**Description**

For `setEmissionLogic` and `setMaterialObject` in `PhiDaily.sol`, it's documented that `emissionLogic` and `materialObject` should be contracts but in reality, the admin can set any address.

With `setTreasuryAddress`, `setMaxClaimed`, `setRoyalityFee` and `setSecondaryRoyalityFee` in `BaseObject.sol` owner can set not validated values which can result in loss of funds for users.

**Location of Affected Code**

File: [`src/PhiDaily.sol`](https://github.com/PHI-LABS-INC/DailyMaterial/blob/355376812ba1e2eeed97d5447c2afea83a3ca8f1/src/PhiDaily.sol)

```solidity
function setEmissionLogic(address _emissionLogic) external onlyOwner {
function setMaterialObject(address _materialObject) external onlyOwner {
```

File: [`src/utils/BaseObject.sol`](https://github.com/PHI-LABS-INC/DailyMaterial/blob/355376812ba1e2eeed97d5447c2afea83a3ca8f1/src/utils/BaseObject.sol)

```solidity
function setTreasuryAddress(address payable newTreasuryAddress) external onlyOwner {
function setMaxClaimed(uint256 tokenId, uint256 newMaxClaimed) public virtual onlyOwner {
function setRoyalityFee(uint256 newRoyalityFee) external onlyOwner {
function setSecondaryRoyalityFee(uint256 newSecondaryRoyalty) external onlyOwner {
```

**Recommendation**

Add proper address validation and upper-bound checks for the fees setter functions despite the fact all these functions are callable only by the owner.

**Team Response**

Acknowledged, Timelock and Multisig will implemented.

## Low Risk

### [L-01] Usage of `ecrecover` Should Be Replaced with Usage of OpenZeppelin's `ECDSA` Library

**Severity**

Low Risk

**Description**

[Signature malleability](https://swcregistry.io/docs/SWC-117) is one of the potential issues with `ecrecover`. The `_isVerifiedCoupon` function calls the Solidity `ecrecover()` function directly to verify the given signatures. However, the `ecrecover()` retrieves the address of the signer via a provided `v`, `r` and `s` signatures. However, due to the nature of the elliptic curve digital signing algorithm (ECDSA), there are two valid points (i.e. two `s` values) on the elliptic curve with the exact same `r` value. An attacker can pretty easily compute the other valid `s` value on the elliptic curve that will return the same original signer, making the signature malleable.

`ecrecover()` allows malleable (non-unique) signatures and thus is susceptible to replay attacks. This means that multiple signatures will be considered valid, which will lead to wrong claim logic.

**Location of Affected Code**

File: [`src/PhiDaily.sol#L161`](https://github.com/PHI-LABS-INC/DailyMaterial/blob/355376812ba1e2eeed97d5447c2afea83a3ca8f1/src/PhiDaily.sol#L161)

```solidity
address signer = ecrecover(digest, coupon.v, coupon.r, coupon.s);
```

**Recommendation**

Use the `recover()` function from OpenZeppelin's ECDSA library to verify the uniqueness of the signature. Using this library implements a check on the value of the `s` variable to ensure that only one of its possible inputs is valid. Ensure that you are using a version `> 4.7.3` for there was a critical bug `>= 4.1.0` `< 4.7.3`.

**Team Response**

Acknowledged and fixed by using `ECDSA` library.


### [L-02] Lack of Consistency and Misleading Naming in Checks

**Severity**

Low Risk

**Description**

There is a different check implementation in `onlyIfAlreadyClaimed` and `onlyIfAlreadyClaimedMultiple` modifiers for the same check. Additionally, both modifier names ensure that the execution will revert if the user hasn't claimed already but actually, it is the opposite, it reverts if the user has claimed.

**Location of Affected Code**

File: [`src/PhiDaily.sol#L121`](https://github.com/PHI-LABS-INC/DailyMaterial/blob/355376812ba1e2eeed97d5447c2afea83a3ca8f1/src/PhiDaily.sol#L121)

```solidity
if (dailyClaimedStatus[_msgSender()][eventid][logicid] > 0) {
```

File: [`src/PhiDaily.sol#L134`](https://github.com/PHI-LABS-INC/DailyMaterial/blob/355376812ba1e2eeed97d5447c2afea83a3ca8f1/src/PhiDaily.sol#L134)

```solidity
if (dailyClaimedStatus[_msgSender()][eventids[i]][logicids[i]] == _CLAIMED) {
```

**Recommendation**

Implement the comparison with the claimed status in both modifiers and correct the naming for sure.

**Team Response**

Acknowledged and fixed as proposed.
