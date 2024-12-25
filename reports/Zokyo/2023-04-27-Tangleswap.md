**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

###  Lack of appropriate access control

**Severity**: High

**Status**: Resolved

**Description**

The initAuctionLauncher() function lacks appropriate access controls. Anyone can call this initAuctionLauncher() function and initialize himself as an admin. 

**Recommendation**: 

It is advised that initAuctionLauncher is called only by the address that has deployed the PostAuctionLauncher rather than relying on quickly calling this init function after deployment of the contract. This will add more security to the initialization step and will reduce any chances of unwanted errors.


## Medium Risk

### Lack of safety measures in depositETH() function

**Severity**: Medium

**Status**: Resolved

**Description**

Part A) depositETH() allows anyone to deposit ETH(i.e. token1 or token2 if it's a WETH) at any point of time. But it can lead to bypass of checks in _deposit() function for the same token1 or token2. This can result in the bypass of some important require checks, such as - 
        require(!launcherInfo.launched, "Must first launch liquidity");
        require(launcherInfo.liquidityAdded == 0, "Liquidity already added");

- specifically, in case when depositETH() is used instead of _deposit() (i.e. depositToken1() or depositToken2()). This can lead to incorrect assumptions being made, which may lead to unintended issues.

**Recommendation**:  

It is advised to review the business and operational logic and add the needed to require checks accordingly.


Part B) Moreover, A malicious admin can withdraw the all ETH(i.e. WETH that was received to the contract via depositETH() function), especially in case when finalize() has been already been called before. 

**Recommendation**: 

To fix this, disallow calling depositETH() function once finalize() has been called or refund the deposited ETH of finalize() has been already called(i.e. when launcherInfo.launched is true).

### Centralization risk and possibility of a Denial of Service

**Severity**: Medium

**Status**: Resolved

**Description**

A malicious admin can call cancelLauncher() and drain all the deposited tokens from the contract. 

**Recommendation**: 

It is advised to add a multisig or a governance mechanism to prevent this from happening.

### Possibility of incorrect minting of liquidity via finalize() 

**Severity**: Medium

**Status**: Resolved

**Description**

Depending upon the logic of TangleswapCallingParams.mintParams(which is out of scope of this audit) if the parameters-
        uint160 sqrtPriceX96,
        int24 tickLower,
        int24 tickUpper
- to the finalize() function are provided incorrect values, which are not corresponding to the getTokenAmounts() values (i.e. lesser than the expected amount), it may lead to lesser liquidity getting minted or added in the liquidity pool. The rest of the liquidity would remain in the contract and which will require an admin to intervene by removing the remaining liquidity via withdrawDeposits(), and then add it manually to the Tangleswap pool, which introduces centralization risk of a malicious admin draining and removing all the remaining liquidity. 

**Recommendation**: 

Therefore, it is advised to check if the following scenarios can arise and introduce specific measures such as an access control mechanism to finalize function so that this issue is prevented.


## Low Risk

### Missing zero value check for locktime

**Severity**: Low

**Status**: Resolved

**Description**

Missing zero value, check for _locktime. It is possible that the _locktime is accidentally set to zero. This will result in the unlocktime same as the time when the pool was launched using finalize(). 

**Recommendation**: 

To avoid this issue, it is advised to add a non-zero requirement check as well for locktime in the initAuctionLauncher() function.

## Informational

### Increase test coverage

**Severity**: Informational

**Status**: Acknowledged

**Description**

It is advised to have more test coverage and integration tests for the contract. This is because the contracts with which the PostLauncher is interacting (which are out of the scope of this audit) might have been modified and can be different from Miso's original contracts. 

**Recommendation**: 

Thus, more thorough testing is advised to check whether the same PostLauncher logic holds true for the contracts with which PostLauncher is interacting.
