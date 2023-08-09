**Auditors**

[Trust Security](https://twitter.com/trust__90)


---


# Findings

## High Risk
### TRST-H-1 More than one hat of the same hatId can be assigned to a user
**Description:**
Hats are minted internally using `_mintHat()`.
```solidity
        /// @notice Internal call to mint a Hat token to a wearer
        /// @dev Unsafe if called when `_wearer` has a non-zero balance of `_hatId`
        /// @param _wearer The wearer of the Hat and the recipient of the  newly minted token
        /// @param _hatId The id of the Hat to mint
        function _mintHat(address _wearer, uint256 _hatId) internal {
            unchecked {
        // should not overflow since `mintHat` enforces max balance of 1
            _balanceOf[_wearer][_hatId] = 1;
        // increment Hat supply counter
        // should not overflow given AllHatsWorn check in `mintHat` ++_hats[_hatId].supply;
        }
        emit TransferSingle(msg.sender, address(0), _wearer, _hatId, 1);
        }
```
As documentation states, it is unsafe if **_wearer** already has the **hatId**. However, this could 
easily be the case when called from `mintHat()`. 
```solidity
        function mintHat(uint256 _hatId, address _wearer) public returns (bool) {
        Hat memory hat = _hats[_hatId];
            if (hat.maxSupply == 0) revert HatDoesNotExist(_hatId);
        // only the wearer of a hat's admin Hat can mint it
             _checkAdmin(_hatId);
            if (hat.supply >= hat.maxSupply) {
                 revert AllHatsWorn(_hatId);
                     }
        if (isWearerOfHat(_wearer, _hatId)) {
                revert AlreadyWearingHat(_wearer, _hatId);
                      }
        _mintHat(_wearer, _hatId);
             return true;
                  }
```
The function validates **_wearer** doesn't currently wear the hat, but its balance could still be 
over 0, if the hat is currently toggled off or the wearer is not eligible.
The impact is that the hat supply is forever spent, while nobody actually received the hat. 
This could be used maliciously or occur by accident. When the hat is immutable, the max 
supply can never be corrected for this leak. It could be used to guarantee no additional, 
unfriendly hats can be minted to maintain permanent power.

**Recommended Mitigation:**
Instead of checking if user currently wears the hat, check if its balance is over 0.


**Team response:**
Accepted.

**Mitigation review:**
Fixed by checking the static hat balance of wearer.



### TRST-H-2 TXs can be executed by less than the minimum required signatures
**Description:**
In HatsSignerGateBase, `checkTransaction()` is the function called by the Gnosis safe to 
approve the transaction. Several checks are in place.
```solidity
        uint256 safeOwnerCount = safe.getOwners().length;
             if (safeOwnerCount < minThreshold) {
                 revert BelowMinThreshold(minThreshold, safeOwnerCount);
        }
```
```solidity
           uint256 validSigCount = countValidSignatures(txHash, signatures, signatures.length / 65);
        // revert if there aren't enough valid signatures
             if (validSigCount < safe.getThreshold()) {
              revert InvalidSigners();
                  }
 ```
The first check is that the number of owners registered on the safe is at least **minThreshold**. 
The second check is that the number of valid signatures (wearers of relevant hats) is not 
below the safe's threshold. However, it turns out these requirements are not sufficient. A 
possible situation is that there are plenty of owners registered, but currently most do not
wear a hat. `reconcileSignerCount()` could be called to reduce the safe's threshold to the 
current validSigCount, which can be below **minThreshold**. That would make both the first 
and second check succeed. However, **minThreshold** is defined to be the smallest number of 
signers that must come together to make a TX. The result is that a single signer could 
execute a TX on the safe, if the other signers are not wearers of hats (for example, their 
toggle has been temporarily set off in the case of multi-hat signer gate.

**Recommended Mitigation:**
Add another check in `checkTransaction()`, which states that **validSigCount >= minThreshold**.


**Team Response:**
Accepted.

**Mitigation review:**
Fixed


### TRST-H-3 Target signature threshold can be bypassed leading to minority TXs
**Description:**
`checkTransaction()` is the enforcer of the HSG logic, making sure signers are wearers of hats 
and so on. The check below makes sure sufficient hat wearers signed the TX:
```solidity
        uint256 validSigCount = countValidSignatures(txHash, signatures, signatures.length / 65);
                // revert if there aren't enough valid signatures
        if (validSigCount < safe.getThreshold()) {
                     revert InvalidSigners();
         }
```
The issue is that the safe's threshold is not guaranteed to be up to date. For example, 
initially there were 5 delegated signers. At some point, three lost eligibility. 
`reconcileSignerCount()` is called to update the safe's threshold to now have 2 signers. At a 
later point, the three signers which lost eligibility regained it. At this point, the threshold is 
still two, but there are 5 valid signers, so if **targetThreshold** is not below 5, they should all 
sign for a TX to be executed. That is not the case, as the old threshold is used. There are 
various scenarios which surface the lack of synchronization between the wearer status and 
safe's stored threshold.

**Recommended Mitigation:**
Call `reconcileSignerCount()` before the validation code in `checkTransaction()`.
 
**Team response:**
Fixed

### TRST-H-4 maxSigners can be bypassed
**Description:**
**maxSigners** is specified when creating an HSG and is left constant. It is enforced in two ways 
**–targetThreshold** may never be set above it, and new signers cannot register to the HSG 
when the signer count reached **maxSigners**. Below is the implementation code in 
HatsSignerGate.
```solidity
        function claimSigner() public virtual {
             if (signerCount == maxSigners) {
                revert MaxSignersReached();
        }
        if (safe.isOwner(msg.sender)) {
                revert SignerAlreadyClaimed(msg.sender);
            }
        if (!isValidSigner(msg.sender)) {
             revert NotSignerHatWearer(msg.sender);
         }
         _grantSigner(msg.sender);
           }
```
An issue that arises is that this doesn't actually limit the number of registered signers. 
Indeed, **signerCount** is a variable that can fluctuate when wearers lose eligibility or a hat is 
inactive. At this point, `reconcileSignerCount()` can be called to update the signerCount to the 
current valid wearer count. A simple attack which achieves unlimited claims is as follows:
1. Assume **maxSigners** = 10
2. 10 signers claim their spot, so **signerCount** is maxed out
3. A signer misbehaves, loses eligibility and the hat. 
4. reconcile() is called, so **signerCount** is updated to 9
5. A new signer claims, making **signerCount** = 10
6. The malicious signer behaves nicely and regains the hat.
7. reconcile() is called again, making **signerCount** = 11
8. At this point, any eligible hat wearer can claim their hat, easily overrunning the 
**maxSigners** restriction.

**Recommended Mitigation:**
The root cause is that users which registered but lose their hat are still stored in the safe's 
owners array, meaning they can always get re-introduced and bump the **signerCount**.
Instead of checking the **signerCount**, a better idea would be to compare with the list of 
owners saved on the safe. If there are owners that are no longer holders, `removeSigner()` can 
be called to vacate space for new signers.

**Team response:**
Accepted; added a `swapSigner()` flow to `claimSigner()`.

**Mitigation review:**
Fixed but introduced a new issue. The new code will swap the new signer with an invalid old 
signer.
```solidity
    address[] memory owners = safe.getOwners();
         uint256 ownerCount = owners.length;
    if (ownerCount >= maxSigs) {
        _swapSigner(owners, ownerCount, maxSigs, currentSignerCount, msg.sender);
    } else {
        _grantSigner(owners, currentSignerCount, msg.sender);
        }
```
However, it's possible that all current owners are valid signers, in this case `_swapSigner()` will 
complete the loop and return gracefully. A user will think they have claimed signer 
successfully, but nothing has changed.

### TRST-H-5 Minority may be able to call safe operations 
**Description:**
Users can update the HSG's view of signers using reconcileSignerCount()
```solidity
        function reconcileSignerCount() public {
            address[] memory owners = safe.getOwners();
                 uint256 validSignerCount = _countValidSigners(owners);
        // update the signer count accordingly
        signerCount = validSignerCount;
        if (validSignerCount <= targetThreshold && validSignerCount != safe.getThreshold())
             {
        bytes memory data =  abi.encodeWithSignature("changeThreshold(uint256)", validSignerCount);
        bool success = safe.execTransactionFromModule(
        address(safe), // to 0, 
        // value data, // data
        Enum.Operation.Call // operation
        );
        if (!success) {
                   revert FailedExecChangeThreshold();
                }
             }
         }
```
Notice that the safe's registered threshold is only updated if the new **validSignerCount** is 
lower than the **targetThreshold**. Actually, that is not desired behavior, because if signers
have reactivated or have become eligible again, it's possible this condition doesn't hold, and 
the previous threshold could be lower than **targetThreshold**. In this scenario, a small 
minority could still execute TXs when **targetThreshold** signatures are needed

**Recommended mitigation:**
Add an else clause, stating that if the new **validSignerCount > targetThreshold** and 
**safe.getThreshold() < targetThreshold**, the threshold changes to **targetThreshold**.

**Team response:**
Accepted.

**Mitigation review:**
Fixed by restructuring conditions in `reconcileSignerCount()`



## Medium Risk
### TRST-M-1 Hats token breaks ERC1155 specifications
**Description:**
The Hats token implements ERC1155 (https://eips.ethereum.org/EIPS/eip-1155). It implements safeTransferFrom() and
batchSafeTransferFrom() as revert-only functions, so tokens cannot be transferred using 
standard ERC1155 means. However, hats can still be transferred using `mintHat()`, 
`mintTopHat()` and `transferHat()`. Whenever there is a transfer, the standard requires 
checking the receiver accepts the transfer:
"If an implementation specific API function is used to transfer ERC-1155 
token(s) to a contract, the `safeTransferFrom` or `safeBatchTransferFrom` (as 
appropriate) rules MUST still be followed if the receiver implements 
the `ERC1155TokenReceiver` interface. If it does not the non-standard 
implementation SHOULD revert but MAY proceed."
By not checking a contract receiver accepts the transfer, Hats token does not adhere to 
ERC1155.

**Recommended Mitigation:**
If the recipient implements ERC1155TokenReceiver, require that it accepts the transfer. If 
the recipient is a contract that does not implement a receiver, reject the operation.

**Team Response:**
Acknowledged; changed documentation to ERC1155-similar and to explicitly clarify that Hats 
implements the ERC1155 interface but does not conform to the full standard.

### TRST-M-2 Attacker can DOS minting of new top hats in low-fee chains
**Description:**
In Hats protocol, anyone can be assigned a top hat via the `mintTopHat()` function. The top 
hats are structured with top 32 bits acting as a domain ID, and the lower 224 bits are 
cleared. There are therefore up to 2^32 = ~ 4 billion top hats. Once they are all consumed, 
`mintTopHat()` will always fail:
```solidity
          // uint32 lastTopHatId will overflow in brackets
             topHatId = uint256(++lastTopHatId) << 224;
```     
This behavior exposes the project to a DOS vector, where an attacker can mint 4 billion top 
hats in a loop and make the function unusable, forcing a redeploy of Hats protocol. This is 
unrealistic on ETH mainnet due to gas consumption, but definitely achievable on the 
cheaper L2 networks. As the project will be deployed on a large variety of EVM blockchains, 
this poses a significant risk.

**Recommended Mitigation:**
Require a non-refundable deposit fee (paid in native token) when minting a top hat. Price it 
so that consuming the 32-bit space will be impossible. This can also serve as a revenue 
stream for the Hats project.

**Team Response:**
Acknowledged; electing not to address in v1 for several reasons:
1. Additional requirement to set & manage authorization for withdrawal
2. Challenge of setting a consistently reasonable fee on chains without stablecoin based native tokens (i.e. all except for Gnosis Chain) / the added complexity and 
centralization risk of making the fee adjustable
3. Contract size constraints


### TRST-M-3 Linking of hat trees can freeze hat operations
**Description:**
Hats support tree-linking, where hats from one node link to the first level of a different 
domain. This way, the amount of levels for the linked-to tree increases by the linked-from 
level count. This is generally fine, however lack of checking of the new total level introduces 
severe risks. 
```solidity
        /// @notice Identifies the level a given hat in its hat tree
        /// @param _hatId the id of the hat in question
        /// @return level (0 to type(uint8).max)
     function getHatLevel(uint256 _hatId) public view returns (uint8) {
```
The `getHatLevel()` function can only return up to level 255. It is used by the `checkAdmin()` call 
used in many of the critical functions in the Hats contract. Therefore, if for example, 17 hat
domains are joined together in the most stretched way possible, It would result in a correct 
hat level of 271, making this calculation revert:
```solidity
        if (treeAdmin != 0) {
                 return 1 + uint8(i) + getHatLevel(treeAdmin);
            }
```
The impact is that intentional or accidental linking that creates too many levels would freeze 
the higher hat levels from any interaction with the contract.

**Recommended Mitigation:**
It is recommended to add a check in `_linkTopHatToTree()`, that the new accumulated level 
can fit in uint8. Another option would be to change the maximum level type to uint32.

**Team Response:**
Accepted; increased max level type to uint32.

**Mitigation review:**
Fixed.


### TRST-M-4 Admin can transfer hat to a non-eligible target, potentially burning the hat
**Description:**
Hat admins may transfer child hats using `transferHat()`. It checks the hat receiver does not 
currently have a balance for this **hatId**.
```solidity
        // Check if recipient is already wearing hat; also checks storage to  maintain balance == 1 invariant
        if (_balanceOf[_to][_hatId] > 0) {
            revert AlreadyWearingHat(_to, _hatId);
         }
 ```
The issue is that it does not also check that the recipient is eligible for the **hatId**. Therefore, 
an admin could transfer a hat and then it could be immediately burnt by anyone using the 
`checkHatWearerStatus()` call.

**Recommended mitigation:**
Verify the recipient is eligible for the **hatId** before transferring it

**Team response:**
Accepted.

**Mitigation review:**
Fixed


### TRST-M-5 Attacker can make a signer gate creation fail
**Description:** 
DAOs can deploy a HSG using `deployHatsSignerGateAndSafe()` or 
`deployMultiHatsSignerGateAndSafe()`.The parameters are encoded and passed to 
`moduleProxyFactory.deployModule()`:
```solidity
    bytes memory initializeParams = abi.encode(_ownerHatId, _signersHatId, _safe, hatsAddress, _minThreshold, 
    _targetThreshold, _maxSigners, version );
        hsg = moduleProxyFactory.deployModule(hatsSignerGateSingleton, abi.encodeWithSignature("setUp(bytes)", 
    initializeParams), _saltNonce );
```
This function will call `createProxy()`:
```solidity
    proxy = createProxy( masterCopy, keccak256(abi.encodePacked(keccak256(initializer), saltNonce)) );
```
The second parameter is the generated salt, which is created from the initializer and passed 
saltNonce. Finally `createProxy()` will use CREATE2 to create the contract:
```solidity
        function createProxy(address target, bytes32 salt)  internal  returns (address result)
        {
            if (address(target) == address(0)) revert ZeroAddress(target);
            if (address(target).code.length == 0) revert 
        TargetHasNoCode(target);
                bytes memory deployment = abi.encodePacked(
                  hex"602d8060093d393df3363d3d373d3d3d363d73", target, hex"5af43d82803e903d91602b57fd5bf3" );
            // solhint-disable-next-line no-inline-assembly
                assembly {
                     result := create2(0, add(deployment, 0x20), 
        mload(deployment), salt)
              }
                  if (result == address(0)) revert TakenAddress(result);
             }
```
An issue could be that an attacker can frontrun the creation TX with their own creation 
request, with the same parameters. This would create the exact address created by the 
CREATE2 call, since the parameters and therefore the final salt will be the same. When the 
victim's transaction would be executed, the address is non-empty so the EVM would reject 
its creation. This would result in a bad UX for a user, who thinks the creation did not 
succeed. The result contract would still be usable, but would be hard to track as it was 
created in another TX.

**Recommended Mitigation:**
Use an ever-increasing nonce counter to guarantee unique contract addresses.

**Team response:**
Accepted.


### TRST-M-6 Signers can backdoor the safe to execute any transaction in the future without consensus
**Description:** 
The function `checkAfterExecution()` is called by the safe after signer's request TX was 
executed (and authorized). It mainly checks that the linkage between the safe and the HSG 
has not been compromised.
```solidity
        function checkAfterExecution(bytes32, bool) external override {
            if (abi.decode(StorageAccessible(address(safe)).getStorageAt(uint256(GUARD_STORAGE_SLOT), 1), (address))
                    != address(this)) 
                    {
                revert CannotDisableThisGuard(address(this));
            }
            if (!IAvatar(address(safe)).isModuleEnabled(address(this))) {
                    revert CannotDisableProtectedModules(address(this));
            }
            if (safe.getThreshold() != _correctThreshold()) {
                     revert SignersCannotChangeThreshold();
            }
            // leave checked to catch underflows triggered by re-erntry
        attempts
            --guardEntries;
        }
```
However, it is missing a check that no new modules have been introduced to the safe. When 
modules execute TXs on a Gnosis safe, the guard safety callbacks do not get called. As a 
result, any new module introduced is free to execute whatever it wishes on the safe. It 
constitutes a serious backdoor threat and undermines the HSG security model.

**Recommended Mitigation:**
Check that no new modules have been introduced to the safe, using the 
`getModulesPaginated()` utility.

**Team response:**
Accepted; added a method for HSG owner to add modules, and an enabled modules counter 
to check against in `checkAfterTransaction()`

**Mitigation review:**
Fix is not bulletproof. A malicious transaction can remove an existing module and replace it 
with their own malicious module. In addition to a length check on the modules array, it is 
necessary to do a full comparison before and after the TX execution.


### TRST-M-7 Hats can't be renounced when not worn, leading to abuse concerns
**Description:**
Hats can be renounced by the owner using `renounceHat()` call. They can only be renounced 
when currently worn, regardless if they have a positive balance. Issues can arise from abuse 
by **toggle or eligibility delegates**. They can temporarily disable or sanction the wearer so 
that it cannot be renounced. At a later point, when the wearer is to be made accountable for 
their responsibilities, they could be toggled back on and penalize an innocent hat wearer.


**Recommended mitigation:**
Allow hats to be renounced even when they are not worn right now. A different event 
parameter can be used to display if they were renounced while worn or not

**Team response:**
Accepted.

**Mitigation review:**
Fixed by applying the suggested mitigation.


### TRST-M-8 Safety checks compare safe's threshold with a stale value
**Description:**
In HatsSignerGateBase, `_correctThreshold()` calculates what the safe's threshold should be. 
However, it uses **signerCount** without updating it by calling `reconcileSignerCount()`. 
Therefore, in `checkAfterExecution()`, the safe's current threshold will be compared to potentially the wrong value. This may trip valid transactions or allow malicious ones to go 
through, where the threshold should end up being higher.

**Recommended mitigation:**
Call `reconcileSignerCount()` before making use of the signerCount value.

**Team response:**
Accepted; added `_countValidSigners()` rather than `reconcileSignerCount()`.

**Mitigation review:**
Fixed.


## Low Risk
### TRST-L-1 createHat does not detect MAX_LEVEL admin correctly
**Description:**
In `createHat()`, the contract checks user is not minting hats for the lowest hat tier:
```solidity
        function createHat( uint256 _admin, string memory _details, uint32 _maxSupply, address _eligibility,
            address _toggle, bool _mutable,  string memory _imageURI)       
                 public returns (uint256 newHatId) {
        if (uint8(_admin) > 0) {
                    revert MaxLevelsReached();
                 }
             ….
        }
```

The issue is that it does not check for max level correctly, as it looks only at the lowest 8 bits. 
Each level is composed of 16 bits, so ID xx00 would pass this check. 
Fortunately, although the check is passed, the function will revert later. The call to 
`getNextId(_admin)` will return 0 for max-level admin, and _checkAdmin(0) is guaranteed to 
fail. However, the check should still be fixed as it is not exploitable only by chance.

**Recommended Mitigation:**
Change the conversion to uint16.

**Team Response:**
Accepted.

**Mitigation review:**
Fixed.


### TRST-L-2 Incorrect imageURI is returned for hats in certain cases
**Description:**
Function `getImageURIForHat()` should return the most relevant imageURI for the requested 
hatId. It will iterate backwards from the current level down to level 0, and return an image if 
it exists for that level.
```solidity
            function getImageURIForHat(uint256 _hatId) public view returns (string memory) {
                // check _hatId first to potentially avoid the `getHatLevel` call
                      Hat memory hat = _hats[_hatId];
                        string memory imageURI = hat.imageURI; // save 1 SLOAD
                // if _hatId has an imageURI, we return it
                            if (bytes(imageURI).length > 0) {
                return imageURI;
                }
                // otherwise, we check its branch of admins
                        uint256 level = getHatLevel(_hatId);
                // but first we check if _hatId is a tophat, in which case we fall back to the global image uri
                if (level == 0) return baseImageURI;
                // otherwise, we check each of its admins for a valid imageURI
                     uint256 id;
                // already checked at `level` above, so we start the loop at `level - 1`
                for (uint256 i = level - 1; i > 0;) {
                     id = getAdminAtLevel(_hatId, uint8(i));
                        hat = _hats[id];
                            imageURI = hat.imageURI;
                if (bytes(imageURI).length > 0) {
                      return imageURI;
                 }
                // should not underflow given stopping condition is > 0
                    unchecked {
                         --i;
                    }
             }
                // if none of _hatId's admins has an imageURI of its own, we 
            again fall back to the global image uri
                return baseImageURI;
            }
```
It can be observed that the loop body will not run for level 0. When the loop is finished, the 
code just returns the baseImageURI, which is a Hats-level fallback, rather than top hat level fallback. As a result, the image displayed will not be correct when querying for a level above 
0, when all levels except level 0 have no registered image.

**Recommended Mitigation:**
Before returning the **baseImageURI**, check if level 0 admin has a registered image.

**Team Response:**
Fixed using an additional check.



### TRST-L-3 Fetching of hat status may fail due to lack of input sanitization
**Description:**
The functions `_isActive()` and `_isEligible()` are used by `balanceOf()` and other functions, so 
they should not ever revert. However, they perform ABI decoding from external inputs.
```solidity
        function _isActive(Hat memory _hat, uint256 _hatId) internal view  returns (bool) {
        bytes memory data = 
             abi.encodeWithSignature("getHatStatus(uint256)", _hatId);
                (bool success, bytes memory returndata) = 
        _hat.toggle.staticcall(data);
        if (success && returndata.length > 0) {
            return abi.decode(returndata, (bool));
                } else {
        return _getHatStatus(_hat);
                }
         }
```
If **toggle** returns invalid return data (whether malicious or by accident), `abi.decode()` would 
revert causing the entire function to revert.

**Recommended Mitigation:**
Wrap the decoding operation for both affected functions in a try/catch statement. Fall back 
to the `_getHatStatus()` result if necessary. Checking that **returndata** size is correct is not 
enough as bool encoding must be 64-bit encoded 0 or 1.

**Team response:**
Accepted.

**Mitigation review:**
Fixed by performing safe decoding of input data, and falling back to the static hat status or 
standing.


### TRST-L-4 Admin check is overly gas-intensive
**Description:**
Hats checks for most operations that the caller is an authorized hat admin. The function 
implementing this check is `isAdminOfHat()`. The function loops and checks if the sender is a 
wearer of a lower-level hat.
```solidity
        while (adminHatLevel > 0) {
             if (isWearerOfHat(_user, getAdminAtLevel(_hatId, adminHatLevel))) 
        {
                return true;
            }
        // should not underflow given stopping condition > 0
            unchecked {
                --adminHatLevel;
            }
        }
```
The issue is that calling `getAdminAtLevel()` for every level is very gas intensive. The vast 
majority of the cost of the function is resolving the hat level using `getHatLevel()`, but it is 
implemented recursively. Most of the gas can be saved by refactoring this code, which will 
be used by almost every interaction with Hats.

**Recommended mitigation:**
Refactor some part of the described code flow so that it will be less gas intensive for hats 
with linked trees

**Team response:**
Accepted; refactored to increase efficiency

**Mitigation review:**
Fixed, but introduced new issue. The new implementation checks at the end of 
`isAdminOfHat()` if the hat is linked to another tree.
```solidity
        // if we get here, we're at the top of _hatId's local tree
             linkedTreeAdmin = linkedTreeAdmins[getTophatDomain(_hatId)];
        if (linkedTreeAdmin == 0) {
            // tree is not linked
                return isWearerOfHat(_user, getLocalAdminAtLevel(_hatId, 0));
        } else {
            if (isWearerOfHat(_user, linkedTreeAdmin)) return true; // user  wears the linkedTreeAdmin
            else return isAdminOfHat(_user, linkedTreeAdmin); 
                // check if  user is admin of linkedTreeAdmin (recursion)
        }
```
Importantly, it is never checked if user is wearer of the admin hat at level 0, in the event that 
the tree is linked. The impact is lack of adminship at the top of the tree.


### TRST-L-5 Lack of zero-address check for important parameters
**Description:** 
Hats allows admin hats to change the **toggle** and **eligibility** set for a given hat. For example:
```solidity
    function changeHatToggle(uint256 _hatId, address _newToggle) external 
    {
        _checkAdmin(_hatId);
             Hat storage hat = _hats[_hatId];
    if (!_isMutable(hat)) {
            revert Immutable();
        }
         hat.toggle = _newToggle;
             emit HatToggleChanged(_hatId, _newToggle);
     }
```
The code lacks a zero-address check for _newToggle. There is no good reason why this value 
might need to be set to zero, and it's a standard best practice for fault-detection.
**Recommended mitigation:**
Add a zero-address check for both address parameters.

**Team response:**
Accepted

**Mitigation review:**
Fixed.


### TRST-L-6 Safe's registered threshold could be below minThreshold
**Description:** 
The functions `setTargetThreshold()` and `reconcileSignerCount()` change the safe's registered 
threshold. Both functions do not check the new value is above or equal to **minThreshold**. 
This is not a serious issue if the HSG is defined to be the enforcer of the **minThreshold**. 
However, this was not done correctly as was described in H-2, so it would be best to never 
set the safe's version of the threshold to below **minThreshold**

**Recommended mitigation:**
Recommended mitigation steps

**Team response:**
Acknowledged; not making explicit changes because the Safe contract does not allow 
thresholds below number of owners, so we can't remove all scenarios where threshold is 
below **minThreshold**. Risks associated with this finding should be mitigated by changes done 
applied in other findings


### TRST-L-7 Reentrancy guards can be easily bypassed
**Description:**
In `checkTransaction()`, **guardEntries** is incremented while in `checkAfterExecution()`, it is 
decremented, implementing a basic reentrancy guard.
The issue is that both functions lack a msg.sender check. Therefore, if there was a way to 
exploit the contract using a reentrancy, this defense would be futile. Attacker could simply 
call `checkTransaction()` an unlimited amount of times, with valid arguments. Then, 
`checkAfterExecution()` would never overflow from the reentrancy abuse.

**Recommended mitigation:**
Check that the msg.sender is the safe for the two functions above.

**Team response:**
Accepted.

**Team response:**
Fixed



## Informational
### Documentation issues
• Remove irrelevant comment in `buildHatId()`:
```solidity///
        @dev Check hats[_admin].lastHatId for the previous hat created underneath _admin
 ```
• Correct comment in checkHatStatus():
```solidity
        // then we know the contract exists and has the getWearerStatus function
```
• Remove comment in MultiHatsSignerGate's claimSigner():
```solidity
        /// @dev overloads HatsSignerGateBase.claimSigner()
```
• Change comment in approveLinkTopHatToTree() – can be approved by wearer or
admin
```solidity
        /// @dev Requests can only be approved by an admin of the `_newAdminHat`, and there
```

### Cleaner code
• Consider using the utility function `getTophatDomain()` here:
```solidity
        uint256 treeAdmin = linkedTreeAdmins[uint32(_hatId >> (256 - TOPHAT_ADDRESS_SPACE))];
```
• **SAFE_TX_TYPEHASH** is not used throughout the code, yet it is declared. Consider 
omitting it.
• Several variables which cannot be changed in the lifetime of the contract are not 
declared immutable (e.g. **maxSigners**). It wastes gas as well as potentially hurting 
the clarity of the code.

### Emit events without state change
In `changeHatMaxSupply()`, if the new **maxSupply** equals the previous **maxSupply**, the 
function doesn't actually change anything yet emits a supply changed event. It is 
recommended to validate the next supply is greater than the current one.

### Improving display of information
In _constructURI(), the printed JSON string is composed of some properties.
```solidity
        // split into two objects to avoid stack too deep error
                string memory idProperties = string.concat('"domain": "',
                     LibString.toString(getTophatDomain(_hatId)), '", "id": "',
                        LibString.toString(_hatId), '", "pretty id": "', "{id}", '",'
        );
```
It is recommended to change the {id} placeholder to a more informative value, such as 
`LibString.toHexString(_hatId, 32)`