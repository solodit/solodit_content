**Auditors**

Talfao

Kalogerone

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/013_CODESPECT_TOKENTABLE_UNLOCKERV2_EVM.pdf)

# Findings

## Medium Risk

### [M-01] Fixed fees allow users to transfer all the project tokens from the Unlocker to the protocol owner

**Files:** [TokenTableUnlockerV2.sol](https://github.com/EthSign/tokentable-v2-evm/tree/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/core/TokenTableUnlockerV2.sol)

**Description:**

In the `TTUFeeCollector` contract, protocol can set fixed fees for selected unlockers. These fees will get charged and transferred every time users call the `claim(...)` function. However, there is no minimum claim amount required for this function to complete. Users can call this function indefinitely until all the `ProjectTokens` in the `Unlocker` get transferred to the `TTUFeeCollector.owner()` as fees.

**Impact:** All the project tokens that have been deposited by the team to cover the users’ claims will be transferred to the Token Table protocol as fees.

**Recommendation:** For fixed fees there should be a reasonable minimum claiming amount.

**Status:** Fixed

**Update from TokenTable:** Fixed in [b119c645215fec35ae08aa2c431433ca52a3dce6](https://github.com/EthSign/tokentable-v2-evm/pull/11/commits/b119c645215fec35ae08aa2c431433ca52a3dce6)

**Update from CODESPECT:** The scenario involving a zero amount has been taken into account; however, in certain situations, the Fixed Fee may still exceed the amount being claimed.

**Update from TokenTable:** Acknowledged.

## Low Risk

### [L-01] Tracker token’s balanceOf will revert if address owns cancelled actuals

**Files:** [TTTrackerTokenV2](https://github.com/EthSign/tokentable-v2-evm/tree/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/core/TTTrackerTokenV2.sol)

**Description:**

Tracker token is supposed to track an Unlcoker’s claimable project tokens. Specifically, the `balanceOf(...)` function should display the amount of currently claimable tokens of the given address. However, the call will always revert if the address owns a cancelled actual. This happens because the `cancel(...)` function deletes the `actuals[]` mapping of the actual.

```solidity
function cancel(
    uint256[] calldata actualIds,
    bool[] calldata shouldWipeClaimableBalance,
    uint256 batchId,
    bytes calldata
) external virtual override onlyOwner returns (uint256[] memory pendingAmountClaimables) {
    ..
    delete $.actuals[actualId];
}
_callHook(_msgData());
}
```

This mapping is later used by the `calculateAmountClaimable(...)` function which Tracker Token’s `balanceOf(...)` function calls.

```solidity
function calculateAmountClaimable(uint256 actualId)
    public
    view
    virtual
    override
    returns (uint256 deltaAmountClaimable, uint256 updatedAmountClaimed)
{
    (deltaAmountClaimable, updatedAmountClaimed) = simulateAmountClaimable(actualId, block.timestamp);
}

function simulateAmountClaimable(uint256 actualId, uint256 claimTimestampAbsolute)
    public
    view
    virtual
    override
    returns (uint256 deltaAmountClaimable, uint256 updatedAmountClaimed)
{
    TokenTableUnlockerV2Storage storage $ = _getTokenTableUnlockerV2Storage();
    Actual memory actual = $.actuals[actualId];
    if (actual.presetId == 0) revert ActualDoesNotExist();
    ..
}
```

**Impact:** If a user owns a cancelled actual, calling `balanceOf(...)` on that address will always revert.

**Recommendation:** Consider making the `simulateAmountClaimable(...)` function return 0 if the actual doesn’t exist instead of reverting.

**Status:** Fixed

**Update from TokenTable:** [4652bc40f9300ebd99a5fc2c26ff20a8cc94fb69](https://github.com/EthSign/tokentable-v2-evm/pull/11/commits/4652bc40f9300ebd99a5fc2c26ff20a8cc94fb69)

### [L-02] Tracker token’s totalSupply is always incorrect

**Files:** [TTTrackerTokenV2](https://github.com/EthSign/tokentable-v2-evm/tree/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/core/TTTrackerTokenV2.sol)

**Description:**

Tracker token is supposed to track an Unlcoker’s claimable project tokens. Specifically, the `totalSupply()` function should track the total number of tokens deposited into the Unlocker awaiting claim.

```solidity
/**
 * @dev Total number of tokens deposited into the unlocker awaiting claim.
 */
function totalSupply() external view returns (uint256) {
    return IERC20Metadata(ttuInstance.getProjectToken()).balanceOf(address(this));
}
```

However, the function is returning the `balanceOf(address(this))`, instead of the balance of the `Unlocker` which holds the project tokens.

**Impact:** Tracker token will return the wrong number of tokens that are deposited in the `Unlocker` as it is tracking the wrong address’ balance.

**Recommendation:** Make the function return the `balanceOf(address(ttuInstance))`.

**Status:** Fixed

**Update from TokenTable:** [0bff495890939f24db93a77b5c03979419b0f2ad](https://github.com/EthSign/tokentable-v2-evm/pull/11/commits/0bff495890939f24db93a77b5c03979419b0f2ad)

### [L-03] Unauthorized claims allowed when externalDelegateRegistry is not configured

**Files:** [TokenTableUnlockerV2.sol](https://github.com/EthSign/tokentable-v2-evm/blob/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/core/TokenTableUnlockerV2.sol#L153-L154)

**Description:**

The `TokenTableUnlockerV2` contract allows designated users to claim tokens on behalf of others via the `delegateClaim(...)` function. A caller can perform this action only if they are:

1. Included in the internal `claimingDelegates` set, or;
2. Authorised via an external delegate registry (if one is configured);

Only the contract owner can manage the `claimingDelegates` list. If an external registry is configured, it will be used to verify whether the `msg.sender` has permission to make a claim on behalf of the original token owner.

```solidity
function delegateClaim(uint256[] calldata actualIds, ...)
    ...
{
    TokenTableUnlockerV2Storage storage $ = _getTokenTableUnlockerV2Storage();
    bool callerIsClaimingDelegate = $.claimingDelegates.contains(_msgSender());
    for (uint256 i = 0; i < actualIds.length; i++) {
        if (
            // @audit statement incorrect
            !callerIsClaimingDelegate && $.currentChainSupportsExternalDelegateRegistry
                && !externalDelegateRegistry.checkDelegateForContract(
                    _msgSender(), $.futureToken.ownerOf(actualIds[i]), address(this), this.delegateClaim.selector
                )
        ) {
            revert NotPermissioned();
        }
        _claim(actualIds[i], address(0), batchId);
    }
    _callHook(_msgData());
}
```

The logic inside the `if` condition is flawed. The intention is to *revert* if the caller is **not** a claiming delegate **and not** authorised via the registry. However, due to the current structure, this is not correctly enforced when the external registry is **not configured**.

Example case where the registry is not configured:

- `callerIsClaimingDelegate = false;`
- `currentChainSupportsExternalDelegateRegistry = false;`

The condition evaluates to:

```text
!false && false => true && false => false
```

This incorrectly allows execution to continue, letting unauthorised users claim on behalf of others if the registry is not enabled.

**Impact:** This breaks the core invariant of the `delegateClaim` logic. Unauthorised users can claim tokens for others when the external registry is not set, potentially compromising any contracts that integrate with `TokenTableUnlockerV2`.

**Recommendation:** Refactor the conditional logic to clearly enforce that only a permitted claiming delegate or a registry-authorized user can claim on behalf of others.

**Status:** Fixed

**Update from TokenTable:** [661c34353311d27ad03f35f9066a7a772d5be6d0](https://github.com/EthSign/tokentable-v2-evm/pull/11/commits/661c34353311d27ad03f35f9066a7a772d5be6d0)

### [L-04] defaultFee can be huge or insignificant depending on the project token’s decimals

**Files:** [TTUFeeCollector.sol](https://github.com/EthSign/tokentable-v2-evm/tree/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/core/TTUFeeCollector.sol)

**Description:**

Projects can deploy an Unlocker for their project token in a permissionless manner through the deployer. If there is no communication or intervention from the protocol team to set up custom fees for the Unlocker, then the `defaultFee` is charged every time users call the `claim(...)` function. However, this `defaultFee` is a fixed fee amount and not a percentage.

Projects can have tokens with varying decimals, ranging from as low as 2 to as high as 24. It is impossible for a `defaultFee` to exist that can fairly cover all these decimals.

**Impact:** Depending on the project token decimals, the `defaultFee` can be huge or very small, requiring the protocol to step in and set up a custom fee. This removes the permissionless nature of the process for the projects.

**Recommendation:** Instead of having the `defaultFee` as a fixed amount, make it a percentage in BIPS.

**Status:** Acknowledged

**Update from TokenTable:** acknowledged

## Informational

### [I-01] Deploying an Unlocker with Multicall may revert for 0 deposit amounts

**Files:** [TTUMulticallDeployer.sol](https://github.com/EthSign/tokentable-v2-evm/tree/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/proxy/TTUMulticallDeployer.sol)

**Description:**

The `TTUMulticallDeployer` contracts allows projects to deploy an Unlocker, create presets, create actuals and deposit their project token in one call. A project may not want to deposit tokens into the Unlocker yet and desire to do it at a later stage (possibly after custom fees have been set up). However, if the project token reverts at 0 amount transfers, then the multicall will also revert:

```solidity
function multicallDeploy(
    ITTUDeployer deployer,
    bytes memory encodedDeployerParams,
    bytes calldata encodedCreatePresetsParams,
    bytes calldata encodedCreateActualsParams
) external {
    ITokenTableUnlockerV2 unlocker;
    {
        (
            address projectToken,
            address existingFutureToken,
            string memory projectId,
            bool isUpgradeable,
            bool isTransferable,
            bool isCancelable,
            bool isHookable,
            bool isWithdrawable,
            uint256 depositAmount
        ) = abi.decode(encodedDeployerParams, (address, address, string, bool, bool, bool, bool, bool, uint256));
        (unlocker,) = deployer.deployTTSuite(
            projectToken,
            existingFutureToken,
            projectId,
            isUpgradeable,
            isTransferable,
            isCancelable,
            isHookable,
            isWithdrawable
        );

        IERC20(projectToken).transferFrom(msg.sender, address(unlocker), depositAmount);
    }
    ...
}
```

**Impact:** Deploying with multicall will revert for 0 deposit amounts if the project token reverts for such transfers.

**Recommendation:** Implement a check that only call `transferFrom(...)` if the `depositAmount` is greater than 0.

**Status:** Acknowledged

**Update from TokenTable:** acknowledged

### [I-02] Future token’s getClaimInfo function can return inaccurate information about token’s cancel status

**Files:** [TTFutureTokenV2.sol](https://github.com/EthSign/tokentable-v2-evm/blob/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/core/TTFutureTokenV2.sol)

**Description:**

The `getClaimInfo(...)` function is supposed to return 3 different variables for the `tokenId` provided:

1. The amount of tokens claimable as of now;
2. The amount of tokens claimed as of now;
3. The cancellability of the token’s unlocker;

However, the cancellability of the unlocker is not enough to determine if a token is cancelled, as the token could get cancelled first and then by calling the `disableCancel(...)` function it will show that it’s not cancellable, which is contradictory to the state of the `tokenId`.

**Status:** Fixed

**Update from TokenTable:** Replaced this function in [15637996d75331063f50ecbc2a5b660f33268f4b](https://github.com/EthSign/tokentable-v2-evm/pull/11/commits/15637996d75331063f50ecbc2a5b660f33268f4b)

### [I-03] Implementation of CustomERC2771Context based on the vulnerable version of OZ contract

**Files:** [CustomERC2771Context.sol](https://github.com/EthSign/tokentable-v2-evm/blob/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/libraries/CustomERC2771Context.sol)

**Description:**

`ERC2771/*.sol` extensions inherit from `CustomERC2771Context.sol`, which facilitates gasless transaction execution. This abstract contract is based on OpenZeppelin’s `ERC2771Context` implementation, version 4.7.0.

However, this version of the OpenZeppelin contract contains a known issue disclosed in the following security advisory: [[ref]](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-g4vp-m682-qqmp). As cited: *“Contracts using ERC2771Context along with a custom trusted forwarder may see `_msgSender` return `address(0)` in calls that originate from the forwarder with calldata shorter than 20 bytes.”*

To prevent this, the contract should be updated to align with the latest OpenZeppelin implementation, which addresses this issue.

**Impact:** If a custom trusted forwarder is used and a transaction includes calldata shorter than 20 bytes, `_msgSender()` will return `address(0)`. This can lead to unexpected behaviour.

**Recommendation:** Upgrade `CustomERC2771Context.sol` to reflect the latest secure implementation provided by OpenZeppelin.

**Status:** Fixed

**Update from TokenTable:** Fixed in [5faa20f8fe1c7937ff72b00bb3579d39980a792b](https://github.com/EthSign/tokentable-v2-evm/pull/11/commits/5faa20f8fe1c7937ff72b00bb3579d39980a792b)

### [I-04] Tracker token’s balanceOf doesn’t account for cancelled actuals

**Files:** [TTTrackerTokenV2.sol](https://github.com/EthSign/tokentable-v2-evm/tree/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/core/TTTrackerTokenV2.sol)

**Description:**

Tracker token is supposed to track an Unlcoker’s claimable project tokens. Specifically, the `balanceOf(...)` function should display the amount of currently claimable tokens of the given address.

```solidity
/**
 * @dev Number of currently claimable tokens of the given address.
 */
function balanceOf(address account) external view returns (uint256) {
    uint256 amountClaimable;
    ITTFutureTokenV2 nftInstance = ttuInstance.futureToken();
    uint256[] memory tokenIdsOfOwner = nftInstance.tokensOfOwner(account);
    for (uint256 i = 0; i < tokenIdsOfOwner.length; i++) {
        (uint256 deltaAmountClaimable,) = ttuInstance.calculateAmountClaimable(tokenIdsOfOwner[i]);
        amountClaimable += deltaAmountClaimable;
    }
    return amountClaimable;
}
```

However, this function doesn’t account for cancelled actuals. A cancelled actual may still have a claimable amount ready to be claimed which is stored in the `pendingAmountClaimableForCancelledActuals` mapping. The `balanceOf(...)` function only returns the claimable amount of active actuals.

**Impact:** The `balanceOf(...)` will return incorrect claimable amount for addresses that have cancelled actuals with claimable balance.

**Recommendation:** Also call the `pendingAmountClaimableForCancelledActuals(...)` function to retrieve this amount for cancelled actuals.

**Status:** Fixed

**Update from TokenTable:** [881e65e4ff8fa35daf4c6d2d04c6a1b2f65fd026](https://github.com/EthSign/tokentable-v2-evm/pull/11/commits/881e65e4ff8fa35daf4c6d2d04c6a1b2f65fd026)

### [I-05] Order of calculation in simulateAmountClaimable function

**Original severity:** Best Practices

**Files:** [TokenTableUnlockerV2.sol](https://github.com/EthSign/tokentable-v2-evm/blob/e27192f627ea849f88e8a4b68382c5ac8808e3a5/contracts/core/TokenTableUnlockerV2.sol#L442)

**Description:**

The calculation order in the `simulateAmountClaimable(...)` function can be optimised to reduce precision loss due to integer division. More specifically, the following line:

```solidity
updatedAmountClaimed = (updatedAmountClaimed * actual.totalAmount) / BIPS_PRECISION / TOKEN_PRECISION;
```

Could be changed to this:

```solidity
updatedAmountClaimed = (updatedAmountClaimed * actual.totalAmount) / (BIPS_PRECISION * TOKEN_PRECISION);
```

**Status:** Fixed

**Update from TokenTable:** Fixed in [f2155e0b440808928330ec90aa6003a4c630eb9d](https://github.com/EthSign/tokentable-v2-evm/pull/11/commits/f2155e0b440808928330ec90aa6003a4c630eb9d)
