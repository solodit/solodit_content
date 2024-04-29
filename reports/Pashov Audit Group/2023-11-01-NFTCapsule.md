**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## High Risk

### [H-01] Protocol assets can be stolen through sandwich attacks

**Severity**

**Impact:**
High, as protocol assets will be lost

**Likelihood:**
High, as sandwich attacks are very common and easy to execute

**Description**

The `Capsule` contract is currently vulnerable to sandwich attack in many places. Every place which does a swap or provides/removes liquidity is done in a manner that is vulnerable. While the code has slippage checks, they are flawed. Take for example the `tradeCRVtoWETH` method, here is how a swap is done:

```solidity
uint256 expectedAmount = curvePoolWETHCRV.get_dy(2, 1, crvBal); // Estimate WETH received for CRV:WETH exchange
// Exchange CRV for WETH within the slippage limit
curvePoolWETHCRV.exchange(2, 1, crvBal, expectedAmount * LPslippageNum / LPslippageDen);
```

The problem is that if an attacker sees you calling `tradeCRVtoWETH` he can imbalance the pool with a very big front-run transaction, which will give you a much smaller `expectedAmount`, and then after you do the swap he will execute a back-run transaction putting the price back to normal, essentially sandwiching your bad trade and profiting your loss. This is a problem in all methods that have an underlying call to `add_liquidity`, `remove_liquidity_one_coin`, `calc_withdraw_one_coin` and `calc_token_amount`.

**Recommendations**

Instead of doing on-chain same transaction calculations to calculate slippage tolerance, add an argument to all methods that use slippage calculations called "minAmountReceived" - it has to be calculated off-chain so it is not prone to on-chain manipulation.

### [H-01] Incorrect `IPETHNFTVault::borrow` assumption breaks the protocol

**Severity**

**Impact:**
Medium, as the protocol will have to be redeployed

**Likelihood:**
High, as it it certain to happen

**Description**

The `depositNft` method in `Capsule` calls `IPETHNFTVault::borrow` with a `_borrowAmount` argument. Later, the code actually tries using the `_borrowAmount` value as the amount of `PETH` to provide as liquidity to a Curve pool. The problem is that the `borrow` method of those vaults always takes a fee, so `Capsule` will have received less than `_borrowAmount` of `PETH`. Quoted from `IPETHNFTVault::borrow`'s NatSpec:

> /// @param \_amount The amount of PUSD to be borrowed. Note that the user will receive less than the amount requested,
> /// the borrow fee and insurance automatically get removed from the amount borrowed

This means that the liquidity provision will always fail due to insufficient `PETH` balance, making the protocol unusable.

Even if the fee value is currently zero it can be changed and the protocol will be broken.

**Recommendations**

Use only the `PETH` received from borrowing for providing liquidity as well as for the `newPosition.amountBorrowed` value in `depositNft`. The issue is also present in `increaseBorrowAmount` and should be addressed there as well.

## Medium Risk

### [M-01] User supplying a fake vault can steal protocol assets as a grief attack

**Severity**

**Impact:**
High, as a malicious actor can drain protocol assets

**Likelihood:**
Low, as it is a common attack vector and it requires no preconditions

**Description**

The `_vault` argument in the `repay` method (and in all others) has no input validation, meaning a user can supply a fake vault which he himself created. By calling `repay` with a fake `_vault` argument for an NFT borrowing position he owns, when `amountBorrowed > repayment` he will go through this code:

```solidity
PETH.approve(address(_vault), tokensWithdrawn); // Approve JPEGD to take PETH
_vault.repay(_id, tokensWithdrawn); // Repay desired amount of loan based on desired repayment size
ePosition.lpSize -= LPcostToRepay; // No need to adjust profit index here, previous check requires they either withdraw or forfeit profits beforehande the users amount borrowed
ePosition.amountBorrowed -= tokensWithdrawn; // Decrease the users amount borrowed
emit DecreaseBorrowAmount(owner, _nft, _id, tokensWithdrawn);
```

this would send the protocol's `PETH` to the user's fake vault, which would be a steal of protocol assets, even though his NFT wouldn't be withdrawable anymore.

**Recommendations**

Use a whitelisting approach to the `_vault` argument in all methods of the contract so that users can't provide fake vaults to them.

## Low Risk

### [L-01] Convex extra reward tokens are not handled

The `claimCVXReward` method makes a call to `getReward` from the Convex reward pool contract. The `getReward` method actually can possibly send other token rewards to the caller as well, but those are not handled in the `Capsule` contract as only `CRV` rewards are expected. Make sure to have a way to handle different token rewards to not miss on potential yield.