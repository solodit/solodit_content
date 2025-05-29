**Auditors**

[zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Lost assets if `collateralWallet` is zero address in SEuroOffering.
**Description**

SEuroOffering.sol - body of transferCollateral(): As this method is triggered by an external call of swap and swapETH. An amount of an asset token is meant to be transferred to collateralWallet in exchange for SEuro.
if (collateralWallet != address(0))
token.transfer (collateralWallet, amount);
In case collateralWallet is zero-address token is not transferred to collateral wallet, hence token amount will be kept in balance of SEuroOffering contract. It is worth noting that the amount of token kept will be inaccessible. Hence, we end up having lost inaccessible assets. In biref: this leads to a scenario in which SEuroOffering can receive assets which are not going to be transferable. Assets shall not be accessible or controlled in this case, hence will be lost.

**Recommendation**

replace if by require.

**Re-audit comment**

Resolved

### Missing minimum output amount in SEuroOffering swap functions.
**Description**

SEuroOffering.sol - swap() and swapETH () do not enable caller to determine the minimum quantity of seuros to receive, which shall expose him/her to attacks. In this scenario, the caller might receive an amount of seuros less than s/he expects.

**Recommendation**

add argument to functions representing the minimum quantity user is willing to receive and a require statement to revert the transaction if that minimum quantity is not met.

**Re-audit comment**

Unresolved.

fix-1:
These functions are exposed to losing funds due to attacks or even accidental price change. Mainly it is getSeuros which is used to calculate price of the asset and determining how much to mint in return (i.e. it also includes oracle call). An unlikely and undesired can be calculated during this transaction, a check needed to be added so that the user determine the minimum amount of token he would like to receive in return.

### Unsafe ERC20 transfer in SEuroOffering `transferCollateral`.
**Description**

SEuroOffering.sol transferCollateral - function is transferring assets (i.e. tokens) without safeguarding that transfer and not validating return.

**Recommendation**

It is preferred to use safeTransfer while transferring ERC20.

**Re-audit comment**

Unresolved

## Low Risk

### Duplicate token addition allowed in TokenManager.
**Description**

TokenManager.sol
addAccepted Token function allows the owner to add the same token multiple times

**Recommendation**

Allow only unique tokens to be added to the accepted token symbols.

**Re-audit comment**

Resolved

### Incomplete removal of multiply-added tokens in TokenManager.
**Description**

TokenManager.sol
Removing an accepted token that is added by the owner multiple times is possibly not removed completely in a single transaction.
Example flow:
Initialize contract
- Add a token e.g, "usdt" 3 times.
Call getAcceptedTokens. Result: ['weth', 'usdt', 'usdt', 'usdt']
Remove the token e.g. "usdt" 1 time
Call getAcceptedTokens Result: ['weth', 'usdt']

**Recommendation**

Allow only unique tokens to be added to the accepted token symbols.

**Re-audit comment**

Resolved

### Missing zero address validation for `setCollateralWallet` in SEuroOffering.
**Description**

SEuroOffering.sol, in setCollateralWallet(address): input address is not asserted to be non-zero address.

**Recommendation**

require statement

**Re-audit comment**

Unresolved

### Missing non-zero amount check for swap functions in SEuroOffering.
**Description**

SEuroOffering.sol body of swap(...) & swapETH(): input amount and msg.value are not required here to be non-zero leading to unnecessary computation.

**Recommendation**

Add checks to ensure input amount and msg.value are non-zero in `swap()` and `swapETH()` functions to prevent unnecessary computation.

**Re-audit comment**

Resolved

## Informational

### Gas optimization opportunity in TokenManager `deleteToken` function.
**Description**

TokenManager.sol
Possible Gas optimization in the deleteToken function.
Current code:
for (uint256 $i=$ index; i < tokenSymbols.length $-1$; i++)
Suggestion:
uint256 len = tokenSymbols.length;
for (uint256 $i=$ index; i < len;) {
unchecked {
++i;
}
}

**Recommendation**

Storing array length in 'len' saves gas rather than reading the length every time. Also, `++i` saves gas rather than 'i++`. The unchecked keyword can be used while incrementing if the number of tokens to be added is intended to exceed the maximum value of uint256 type.

**Re-audit comment**

Unresolved

### Potential DOS from block gas limit in TokenManager.
**Description**

TokenManager.sol
DOS - Block Gas Limit: For large array lengths, the gas usage will be significantly higher and can lead to the failure of a transaction to be included in the blockchain due to Block Gas Limit
https://swcregistry.io/docs/SWC-128

**Recommendation**

Consider mechanisms to handle large arrays to prevent exceeding block gas limit, such as pagination or limiting array size.

**Re-audit comment**

Unresolved

### Deployer responsibility for correct Oracle address in TokenManager.
**Description**

TokenManager.sol
The deployer can put any address as the Oracle address. This address is used by other contracts in the project for making calculations. It is the responsibility of the deployer to put the correct oracle address.

**Recommendation**

Ensure a trusted and correct Oracle address is set during deployment, as this is critical for calculations in other project contracts.

**Re-audit comment**

Unresolved

### Excessive admin authority across contracts.
**Description**

All contracts - Admin enjoys too much authority. The general theme of the code is that admin has power to call several functions like set BondingCurve, setCalculator, setTokenManager and more importantly granting roles in all the contracts which leads to enable him to manipulate funds. Some functions can be more highly severe to be left out controlled by one wallet more than other functions.

**Recommendation**

Apply governance / use multisig wallets.

**Re-audit comment**

Unresolved

### Floating pragma versions.
**Description**

Lock the pragma to a specific version, since not all the EVM compiler versions support all the features, especially the latest ones which are kind of beta versions, So the intended behavior written in code might not be executed as expected. Locking the pragma helps ensure that contracts do not accidentally get deployed using, for example, the latest compiler, which may have higher risks of undiscovered bugs.

**Recommendation**

fix version to 0.8.15

**Re-audit comment**

Resolved

### Direct contract referencing instead of interfaces in SEuroCalculator.
**Description**

SEuroCalculator.
SEuroCalculator.sol - Reference of BondingCurve, TokenManager
Generally referencing the contracts as is rather than implementing interfaces.

**Recommendation**

Use interfaces for interacting with external contracts (BondingCurve, TokenManager) instead of direct contract referencing for better upgradability and modularity.

**Re-audit comment**

Unresolved

### Discrepancy in `getBucketMidpoint` logic in BondingCurve.sol.
**Description**

Bonding Curve.sol - in body of getBucketPrice(): getBucketMidpoint(bucketIndex) is supposed to represent x in this formula: $y=k^{*}(x/m)\wedge j+i,$ where $x=$ current total supply of sEURO by Bonding Curve according to doc provided in code which is ibcoTotalSupply. But we have getBucketMidpoint (bucketIndex) = (bucket Index bucketSize) + (bucketSize / 2) which is kind of a quantized quantity of ibcoTotalSupply and not the same.

**Recommendation**

Align the implementation of `getBucketMidpoint` with the documented formula or update the documentation to reflect the current quantized approach.

**Re-audit comment**

Unresolved.

fix-1:
No change done by devs here to address the issue. The developing team might tell us that this does not harm the tokenomics of the project. Based upon their knowledge about the tokenomics the issue shall be considered irrelevant from our side.
