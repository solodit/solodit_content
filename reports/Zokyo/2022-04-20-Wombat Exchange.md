**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### In contract TokenVesting.sol, in the function setBeneficiary there are no checks in order to see that the WOM token balance of the contract is greater than or equal to the _totalAllocationBalance + allocation. This can be a centralization issue because the owner might choose not to send the tokens for a certain beneficiary. Also, it can lead to undefined behavior as some users might not claim on time, leading to them not being able to claim after others already claimed their share. Alternatively you could add a require for the contract balance to be greater than or equal to the new _totalAllocationBalance. This should be added before line 117 (before storing beneficiary info and increasing _totalAllocationBalance). Also consider adding a transferFrom directly in the function, after the beneficiary data has been stored, so that the setting and transferring is done in a single atomic transaction.


### In contract Asset.sol, override the “approve” function from ERC20, to not be used. Beware that changing an allowance with this method brings the risk that someone may use both the old and the new allowance by unfortunate transaction ordering. Use IncreaseAllowance and DecreaseAllowance instead of Approve.


### In contract WombatERC20.sol, override the “approve” function from ERC20, to not be used. Beware that changing an allowance with this method brings the risk that someone may use both the old and the new allowance by unfortunate transaction ordering. Use IncreaseAllowance and DecreaseAllowance instead of Approve.


### In contract DSMath.sol, at line 43, function wdiv is not consistent with the message in the comment above the function. The function does not return zero if x*y<WAD. Unexpected results may also occur when y has very low values.

## Informational

### In contract Pool.sol, in function “setFeeTo”, error message that appears when “feeTo” is zero address is not the expected one.

**Recommendation**: Use “_checkAddress” function instead of checking through an if, at line 238.


### Redundant cast in contract TokenVesting.sol, at line 143 timestamp is being cast to uint256 although it is received as a parameter of type uin256.
**Recommendation**: Remove the cast at line 143.

### In contract TokenVesting.sol, redundant initialization of IERC20(vestedToken) at line 133 inside safeTransfer. The vestedToken is already an IERC20 initialized in the constructor.
**Recommendation**: Remove the initialization of IERC20 inside the safeTransfer call. Also, might consider dropping the initialization in the constructor and storing only the address and initializing only inside the safeTransfer call at line 133, as it reduces gas.


### In contract TokenVesting.sol in function _vestingSchedule at lines 156-175 there is no default return path for the function. Also, the condition at line 165 will always be  true as the parameter timestamp is passed from release function as block.timestamp.
**Recommendation**: Consider changing the order of conditions by calling _calculateInterval only once and then have the isUnlocked check and update the _unlockIntervalsCount. Attached below is a snippet of how this can be refactored.


 ![image](https://github.com/user-attachments/assets/ae0a95fd-72bc-4eaf-bb65-f8a13b06f827)




### In contract Pool.sol, the state mutability of the functions at lines 114-132 can be restricted to pure as they operate on private variables and act as modifiers.



### In contract Pool.sol at lines 197-200 there’s no event emitted after changing the dev address, similar to the Ownable approach which emits a OwnershipTransferred event.
**Recommendation**: Add an event for the setDev function.




### In contract Pool.sol the IMasterWombat is declared, line 63, as variable and set in storage through the setMasterWombat at lines 202-205. It is then used in function deposit, at lines 446 and 447.
**Recommendation**: Consider declaring/storing only the address of the MasterWombat, as masterWombatAddress and use in-place interface initialization such as IMasterWombat(masterWombaAddress).someFunc(..) as this can reduce gas cost.


### In contract Pool.sol at lines 769-775 in function globalEquilCovRatio variables equilCovRatio and invariant are shadowed, leading to redeclaration in the function body.
**Recommendation**: Either remove the named return types to avoid the shadowing and declare inside the function body or rename the invariant declaration at 770 and remove the uint256 redeclaration of equilCovRatio at 773.


### In contract Pool.sol at lines 348-350 function assetOf has a misleading naming. It receives a token address and returns the address of the IAsset.
**Recommendation**: Consider renaming to addressOfAsset.


