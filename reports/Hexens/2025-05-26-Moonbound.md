**Auditors**

[Hexens](https://hexens.io/)

---

# Findings
## High Risk

### [MNBD1-5] Early Pair Creation Breaks Fair Launch of New Moon Tokens And Gives An Attacker The Possibility To Steal All KAS from BondingCurvePool

**Severity:** Critical

**Path:** contracts/BondingCurvePool.sol#L234-L265 

**Description:** A bonding curve is designed to create fair and manipulation-resistant pricing for new tokens in their early stages. However, if someone can create a Zealous pair and control the reserves or pricing, they can influence how the token is valued after its graduation.

For example, an attacker could exploit this by purchasing tokens from MoonBound and then creating a liquidity pair for the token using `ZealousSwapRouter` by calling the `addLiquidityKAS` function. They could add a large amount of KAS (e.g., 1e10 tokens) and just 1 wei of the newly created moon token. This results in an extremely low price for buying KAS with moon tokens.

The pricing formula used when adding liquidity is:
```
amountB = (amountA * reserveB) / reserveA;
```
In simple terms:
```
kasAmount = (moonTokenAmount * kasReserves) / moonTokenReserves;
```
After a token is launched and reaches the "graduated" state, it typically calls `addLiquidityKAS` on the `ZealousSwapRouter`, supplying a certain amount of KAS (X) and a portion of the token supply (Y), usually around 25% of the total supply.

However, due to the severe imbalance in reserves caused by the attacker, the router ends up transferring nearly all of the KAS and only a negligible amount of moon tokens.

A malicious user can then swap their moon tokens and extract most or all of the newly added KAS, effectively draining the pool.
```
ERC20(token).approve(zealousSwapRouter, tokenForLiquidity);
IZealousSwapRouter02(zealousSwapRouter).addLiquidityKAS{ value: kasCollected }(
  token,
  tokenForLiquidity,
  0,
  0,
  address(this),
  block.timestamp + 15 minutes
);
```


**Remediation:**  The `ZealousSwapPair` contract should only be deployed after the MoonBound token has completed its `graduate()` process. In other words, the protocol should implement a modified version of the `ZealousSwapFactory` contract that includes a check to ensure the token has graduated before allowing the `createPair()` function to execute successfully.

It’s essential to ensure that the token is marked as graduated before any liquidity is added to the pair. In the `graduateToken()` function, move the `tokenManager.graduateToken(token)` call to occur before adding liquidity:
```
function graduateToken() internal {
    // -- snip -- 
    
++  tokenManager.graduateToken(token);
    
    ERC20(token).approve(zealousSwapRouter, tokenForLiquidity);
    IZealousSwapRouter02(zealousSwapRouter).addLiquidityKAS{ value: kasCollected }(
      token,
      tokenForLiquidity,
      0,
      0,
      address(this),
      block.timestamp + 15 minutes
    );

    // Mark the token as graduated in the TokenManager
    // This will disable trading on the bonding curve
--  tokenManager.graduateToken(token);
    
    // -- snip -- 
}

```

**Status:**  Fixed


- - -

## Medium Risk

### [MNBD1-12] Insufficient Check of Approval Delay in BondingCurvePool
 
**Severity:** Medium

**Path:** contracts/BondingCurvePool.sol#L95-L100 

**Description:** In the `BondingCurvePool.buyTokens()` function, lines 95–100 attempt to enforce a delay between a user’s approval and their first token purchase:
```
if (firstApprovalBlock[to] > 0 && lastBuyBlock[to] == 0) {
  require(
    block.number > firstApprovalBlock[to] + APPROVE_DELAY,
    'Please wait ~2.5s after approving before buying'
  );
}
```
However, this check can be bypassed if the user never calls the `registerApproval()` function before purchasing. In that case, `firstApprovalBlock[to]` remains at its default value of zero, making the condition `block.number > firstApprovalBlock[to] + APPROVE_DELAY` trivially true.

As a result, users can skip the intended delay and buy tokens immediately, defeating the purpose of the `APPROVE_DELAY` restriction.

**Remediation:**  Consider adding the following requirement:
```
++  require(firstApprovalBlock[to] != 0, 'User must be approved first'); 

    if (firstApprovalBlock[to] > 0 && lastBuyBlock[to] == 0) {
      require(
        block.number > firstApprovalBlock[to] + APPROVE_DELAY,
        'Please wait ~2.5s after approving before buying'
      );
    }
```


**Status:**  Fixed

- - -

### [MNBD1-4] Tokens cannot graduate if an attacker transfers KAS to the BondingCurvePool contract

**Severity:** Medium

**Path:** contracts/BondingCurvePool.sol#L247-L260 

**Description:** The `BondingCurvePool::graduateToken()` function is responsible for creating a new `ZealousSwapPair` between the MoonBound token and KAS. It does this by using the KAS collected during the bonding curve phase and a reserve of MoonBound tokens (25% of `maxSupply`):
```
function graduateToken() internal {
    ...
    // Add liquidity to DEX
    uint256 currentPrice = getPriceAtTokens(totalTokensSold);
    uint256 kasCollected = address(this).balance;

    uint256 tokenForLiquidity = (kasCollected * SCALING_FACTOR) / currentPrice;
    ...
}
```
The function calculates `kasCollected` using `address(this).balance`, and determines the number of MoonBound tokens needed for liquidity by dividing `kasCollected` by `currentPrice`.

Under normal conditions, `tokenForLiquidity` should always be less than the reserved token amount, because:

- When all curve tokens (75% of maxSupply) are sold:

    - `currentPrice` equals `graduationPriceKAS`

    - `kasCollected` is expected to be less than
```(curveTokens / 3) * graduationPriceKAS
= reservedTokens * graduationPriceKAS
```
(due to the deduction of `graduationFeeKAS`)

The issue arises when an attacker can artificially inflate `kasCollected` by forcefully transferring extra KAS to the contract. This causes `tokenForLiquidity` to exceed `reservedTokens`, which leads to a failure in the call to `IZealousSwapRouter02(zealousSwapRouter).addLiquidityKAS()` because of not enough token to transfer.

This can be exploited via a self-destruct technique, allowing the attacker to send `graduationFeeKAS + ε` KAS to the `BondingCurvePool` even though the contract lacks a `receive()` function.

**Remediation:**  If `tokenForLiquidity` exceeds `reservedTokens`, cap `tokenForLiquidity` to `reservedTokens`.
```
   uint256 tokenForLiquidity = (kasCollected * SCALING_FACTOR) / currentPrice;
++ if (tokenForLiquidity > reservedTokens) {
++    tokenForLiquidity = reservedTokens;
++ }
```

**Status:**  Fixed


- - -

### [MNBD1-2] Lack of Slippage Protection During Graduation in BondingCurvePool

**Severity:** Medium

**Path:** contracts/BondingCurvePool.sol#L253-L260 

**Description:** When the `BondingCurvePool` has sold all its tokens, it calls `graduateToken` and sends all collected Ether to the Zealous Swap Router to convert it into liquidity. However, it lacks any checks to ensure a fair amount of liquidity is received.
```
IZealousSwapRouter02(zealousSwapRouter).addLiquidityKAS{ value: kasCollected }(
  token,
  tokenForLiquidity,
  0,
  0,
  address(this),
  block.timestamp + 15 minutes
);
```
This makes it vulnerable to sandwich attacks from other users. A user could watch the BondingCurvePool, buy the last tokens to trigger graduation, and then front-run this transaction with another transaction to exploit the lack of slippage (eg: create a pool with server imbalance reserves).

**Remediation:**  To enhance safety, consider setting minimum thresholds for `amountTokenMin` and `amountKASMin`. If the protocol enforces that the `ZealousSwapPair` can only be created after the token has graduated, these minimums can be set to `tokenForLiquidity` and `kasCollected`, respectively.

**Status:**   Fixed


- - -
## Low Risk


### [MNBD1-6] An attacker can prevent the graduation of the BondingCurvePool by front-running to sell a very small amount of curve tokens

**Severity:** Low

**Description:** In the BondingCurvePool contract, `graduateToken()` is only triggered when all curve tokens have been sold, as indicated by a condition in the `buyTokens()` function:
```
// Graduate if all tokens have been sold
if (totalTokensSold == curveTokens) {
  graduateToken();
}
```
`totalTokensSold` increases by the amount of curve tokens bought by users in the `buyTokens()` function. However, it also decreases in the `sellTokens()` function whenever curve tokens are sold.

The problem is that there is no minimum threshold for the amount of curve tokens that can be sold in the `sellTokens()` function. This allows an attacker to sell just 1 wei of curve tokens to reduce `totalTokensSold`, thereby preventing graduation from occurring. The attacker can front-run any `buyTokens()` transaction by selling 1 wei of curve tokens to block the graduation.
```
function sellTokens(
  uint256 amount,
  address to
) external override nonReentrant onlyMoonBound returns (uint256) {
  bool graduated = tokenManager.getTokenGraduationStatus(token);
  require(!graduated, 'Token has already graduated.'); // Check graduation status
  require(amount > 0, 'Amount must be greater than 0');
  require(ERC20(token).balanceOf(to) >= amount, 'Not enough tokens to sell');
  require(ERC20(token).allowance(to, address(this)) >= amount, 'Insufficient allowance');

  uint256 refundAmount = calculateCost(totalTokensSold - amount, totalTokensSold);
  require(address(this).balance >= refundAmount, 'Not enough KAS in the pool.');

  ERC20(token).transferFrom(to, address(this), amount); // Transfer tokens to the contract

  // Update the total tokens sold
  totalTokensSold -= amount;

  uint256 tax = (refundAmount * TAX_BPS) / 10000; // 1% tax
  if (tax > 0) {
    refundAmount -= tax; // Deduct tax from refund

    // Transfer tax to moonbound vault
    (bool taxSuccess, ) = moonBound.call{ value: tax }('');
    require(taxSuccess, 'Failed to send tax to moonBound vault');
  }

  // Transfer KAS to the seller
  (bool success, ) = to.call{ value: refundAmount }('');
  require(success, 'Failed to send refund to the sender');

  emit TokensSold(to, amount, refundAmount);

  return refundAmount;
}

```


**Remediation:**  A minimum threshold for the amount of tokens sold in the `sellTokens()` function should be added.

**Status:** Fixed

- - -

### [MNBD1-14] 10 KAS Permanently Locked in MoonBound Contract

**Severity:** Low

**Path:** contracts/MoonBound.sol#L186-L189 

**Description:** The `distributeRevenue()` function handles the distribution of revenue from MoonBound tokens to relevant parties (e.g., projects, stakers, etc.). It distributes an amount of KAS `kasToDistribute` equal to the contract's total KAS balance minus 10 KAS. As a result, 10 KAS will always remain in the contract, with no available function to claim or withdraw it.
```
uint256 totalKasBalance = address(this).balance;
require(totalKasBalance > 10e18, 'Insufficient balance for distribution');

uint256 kasToDistribute = totalKasBalance - 10e18;
```

**Remediation:**  Consider adding a function that allows the owner to claim the remaining 10 KAS, or alternatively, modify the `distributeRevenue()` function to distribute the entire KAS balance held by the contract.

**Status:** Fixed


- - -

### [MNBD1-10] Incorrect Assumption on totalVolume in distributeToProjects() May Lead to Misallocation or Revert

**Severity:** Low

**Path:** contracts/MoonBound.sol#L224-L250 

**Description:** The `distributeToProjects()` function calculates each creator’s share using `totalVolume` as the denominator:
```
uint256 creatorShare = (kasToProjects * tokenVolumes[i]) / totalVolume;
```
However, it assumes that `totalVolume == sum(tokenVolumes)`, without enforcing this constraint. If the sum of `tokenVolumes` is not equal to totalVolume, the distribution becomes inaccurate—potentially underpaying creators or causing a revert due to rounding or insufficient balance.

To ensure correct behavior, the function should either validate that `totalVolume` matches the sum of `tokenVolumes`, or compute `totalVolume` internally to eliminate this dependency.

The same issue applies to the function `distributeToNFTStakers()`.
```
function distributeToProjects(
  uint256 kasToDistribute,
  address[] calldata tokenCreators,
  uint256[] calldata tokenVolumes,
  uint256 totalVolume
) internal {
  uint256 kasToProjects = (kasToDistribute * 10) / 100;
  for (uint256 i = 0; i < tokenCreators.length; i++) {
    uint256 creatorShare = (kasToProjects * tokenVolumes[i]) / totalVolume;
    (bool projectSuccess, ) = tokenCreators[i].call{ value: creatorShare }('');
    require(projectSuccess, 'Failed to transfer to token creator');
  }
}

function distributeToNFTStakers(
  uint256 kasToDistribute,
  address[] calldata nftStakers,
  uint256[] calldata stakerPowers,
  uint256 totalPowerStaked
) internal {
  uint256 kasToNFTStakers = (kasToDistribute * 10) / 100;
  for (uint256 i = 0; i < nftStakers.length; i++) {
    uint256 stakerShare = (kasToNFTStakers * stakerPowers[i]) / totalPowerStaked;
    (bool stakerSuccess, ) = nftStakers[i].call{ value: stakerShare }('');
    require(stakerSuccess, 'Failed to transfer to NFT staker');
  }
}
```

**Remediation:**  Consider adding a check to ensure that the sum of all elements in the `tokenVolumes[]` array matches the `totalVolume` value. A similar validation should also be applied in the `distributeToNFTStakers()` function.

**Status:**  Fixed

- - -