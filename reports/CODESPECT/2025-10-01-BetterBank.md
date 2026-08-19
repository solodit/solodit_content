**Auditors**

[jecikPo](https://x.com/JecikPo), [namx05](https://x.com/namx05), [0xAdityaRaj](https://x.com/0xAdityaRaj)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/032_CODESPECT_BETTERBANK.pdf)

# Findings

## Medium Risk

### [M-01] Flash loan inflatable staking balance allows stealing rewards

**Files:** [`Staking.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Staking.sol#L191)

**Description:**

The `Staking` contract is used to distribute Favor token rewards to Esteem token stakers. The rewards are delivered to the `Staking` contract through the `allocateSeigniorage(...)` function which is called from `FavorTreasury`. Only approved addresses can call it, yet the `FavorTreasury.allocateSeigniorage(...)` (which calls `Staking.allocateSeigniorage(...)`) can be called by anyone, only once during epoch. The `Staking` contract doesn’t have stake lockup period, users are allowed to stake and withdraw Esteem anytime.

Since anyone can call `FavorTreasury.allocateSeigniorage(...)` it makes the `Staking` contract vulnerable to a flash loan attack that could inflate a malicious user’s stake by using the following sequence:

a. Malicious user takes large Esteem flash loan at the approaching end of the period;
b. The flash loan is staked at `Staking`;
c. The `FavorTreasury.allocateSeigniorage(...)` is called and smaller RPS is calculated because of the large stake;
d. User collects Favor rewards and repays the flash loan;

**Impact:** A malicious user could take large chunk of the Favor rewards at only the flash loan cost at the expense of the legitimate staker’s rewards.

**Recommendation:** This could be solved in few ways:

a. Prevent staking and withdraw during same block or timestamp;
b. Enforce a small fee on the withdrawals;
c. Make the `FavorTreasury.allocateSeigniorage(...)` function admin/owner controlled only;

**Status:** Fixed

**Client response:** Fixed in [e0c58f4eae15d1e5579e15a0a64431db5d409499](https://github.com/grape-finance/BB-Custom-contracts/commit/e0c58f4eae15d1e5579e15a0a64431db5d409499)

### [M-02] Missing slippage protection when interacting with UniswapV2 Pair

**Files:** [`Zapper.sol/L150`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Zapper.sol#L150), [`Zapper.sol/L183`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Zapper.sol#L183)

**Description:**

The `Zapper` contract allows handling of both base and Favor tokens - it manages adding liquidity on behalf of the user and can swap token to favor when needed. Both `_swap(...)` and `_addLiquidity(...)` rely on router functions but have hard-coded the minimum output parameters to `0`. In `_swap(...)`, the contract uses `router.swapExactTokensForTokensSupportingFeeOnTransferTokens(...)` with no slippage threshold, allowing execution at unfavorable exchange rates.

**Impact:** An attacker can front-run the swap with a sandwich attack. These omissions generally expose users to front-running and sandwich attacks, yet the presence of a transfer tax on Favor token reduces exploitability. Any attacker attempting to manipulate the reserves to force poor execution must pay the tax when moving Favor, which erodes profit margins and can make such attacks economically infeasible. Still, the lack of slippage protection means the system does not revert in adverse conditions, and users can unintentionally add liquidity or swap at unfavorable terms. The price could still be altered through adding liquidity which swaps half of the added tokens.

**Recommendation:** Enforce a configurable minimum token output when calling the router to ensure trades or liquidity adding reverts, if the expected rates cannot be met.

**Status:** Fixed

**Client response:** Fixed in [b9413cc025d7f7cc76daf28224b917923daf732f](https://github.com/grape-finance/BB-Custom-contracts/commit/b9413cc025d7f7cc76daf28224b917923daf732f)

**CODESPECT fix review:** Lack of slippage protection in `_zapToken(...)` acknowledged.

### [M-03] Pausing of FavorTreasury queues outstanding epochs to be executed

**Files:** [`FavorTreasury.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/FavorTreasury.sol#L312)

**Description:**

The `FavorTreasury` contract implements a custom epoch checkpoint mechanism instead of using the standard `Epoch.sol`. In this mechanism, functions with the `checkEpoch` modifier increment the epoch by **one** per call, rather than calculating the current epoch based on the elapsed time.

This design introduces unpredictable behaviour if the protocol is paused for an extended period. For example, if the protocol is paused for a week, calling `allocateSeigniorage(...)` after unpausing could process all outstanding epochs (e.g., 7 days × 24 = 168 epochs). This would mint `Favor` tokens for the entire paused duration. Additionally, an attacker could exploit this by taking a flash loan, staking via `Staking.sol`, and claiming rewards for all 168 epochs, potentially draining rewards intended for long-term stakers.

**Impact:**

- Unexpected minting of `Favor` tokens for paused periods;
- Potential manipulation of staking rewards;
- Risk of reward theft from long-term stakers;

**Recommendation:** Update the epoch logic to compute the current epoch based on the elapsed time rather than incrementing by one, similar to the approach in `Epoch.sol`.

**Status:** Fixed

**Client response:** Was fixed, epoch system was reworked ([1e7747aa89f79ff8ad55e8facc0a0e3a5cc3a699](https://github.com/grape-finance/BB-Custom-contracts/commit/1e7747aa89f79ff8ad55e8facc0a0e3a5cc3a699))

### [M-04] Staking user can inflate RPS on low total supply of shares

**Files:** [`Staking.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Staking.sol#L198)

**Description:**

On `Staking.sol` contract the `allocateSeigniorage(...)` function is used to add rewards. The RPS (Reward Per Share) value is calculated based on the amount deposited and total supply of shares:

```solidity
uint256 nextRPS = prevRPS + ((amount * 1e18) / totalSupply());
```

If the total supply falls below `1e18` the RPS value is inflated to multiple times deposited amount.

While it is not likely that such a condition might occur, it may happen shortly before launch if a malicious user deposits a small amount just before initial rewards are loaded in. This will cause his RPS to grow significantly at the expense of later users.

**Impact:** While the occurence is not likely, as it would require specific timing (e.g. front-running initial rewards adding event), the impact would be severe as it the early deposit would get a disproportionate amount of rewards accrued. Those rewards in form of favor token, if claimed, would be at the expense of other stakers.

**Recommendation:** The `totalSupply() > 0` condition of the `allocateSeigniorage(...)` could be changed so that the total supply cannot be lower than `1e18`, yet it will prevent loading of the seigniorage.

**Status:** Fixed

**Client response:** We implemeted minimal value as proposed. ([f0770e8c96c959f27d1cff74b46d66812ffc8253](https://github.com/grape-finance/BB-Custom-contracts/commit/f0770e8c96c959f27d1cff74b46d66812ffc8253))

### [M-05] UniswapV2 price TWAP accumulator underflow will lead to oracle update failure

**Files:** [`LPOracle.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/LPOracle.sol#L112)

**Description:**

The `LPOracle` contract contains `update()` function which is used to update the TWAP price accumulator in its storage. As the accumulator returned by the UniswapV2 Pair contract only grows it is necessary to handle the underflow using the `unchecked` clause which is not present:

```solidity
uint32 dt = blockTs - blockTimestampLast;
require(dt > 0, "Oracle: ZERO_TIME");

price0Average = FixedPoint.uq112x112(
    uint224((p0C - price0CumulativeLast) / dt)
);
price1Average = FixedPoint.uq112x112(
    uint224((p1C - price1CumulativeLast) / dt)
);

price0CumulativeLast = p0C;
price1CumulativeLast = p1C;
blockTimestampLast = blockTs;
```

The correct implementation is present at the `UniTWAPOracle` contract:

```solidity
unchecked {
    // Overflow desired wrapped in unchecked
    timeElapsed = blockTimestamp - blockTimestampLast;
}
// [...]
unchecked {
    // Overflow desired wrapped in unchecked
    price0Delta = price0Cumulative - price0CumulativeLast;
    price1Delta = price1Cumulative - price1CumulativeLast;
}
```

The underflow will render the `update()` function revert and hence the `consult()` will always returned stale TWAP as there is no check against it.

The problem exists also on the USD accumulator:

```solidity
uint256 newUsd0C = usd0CumulativeLast + u0 * dtU;
uint256 newUsd1C = usd1CumulativeLast + u1 * dtU;
```

**Impact:** Lack of correct up to date collateral prices combined with lack of staleness check will result in incorrect stale prices of the collateral tokens.

**Recommendation:** Add the nessary `unchecked` clause. Consider adding staleness check into the `consult` function.

**Status:** Fixed

**Client response:** [1e7747aa89f79ff8ad55e8facc0a0e3a5cc3a699](https://github.com/grape-finance/BB-Custom-contracts/commit/1e7747aa89f79ff8ad55e8facc0a0e3a5cc3a699) Wrapped into unchecked

### [M-06] Zapper can retain dust affecting depositor funds

**Files:** [`Zapper.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Zapper.sol)

**Description:**

The `Zapper` contract supports "zapping" a base token into a UniswapV2 LP by swapping half of the provided base token to the second pool token and then calling `router.addLiquidity(...)`, after which LP tokens are forwarded into the lending `POOL`. The function that perform this flow include `zapToken(...)`. The result of adding liquidity to the UniswapV2 pool is that not all tokens provided could be added. Those leftovers stay on the `Zapper` contract and the `_refundDust(...)` is called to transfer them back to the caller.

The `zapToken(...)` is not the only function adding liquidity, other are:

- `addLiquidity(...)`;
- `addLiquitityETH(...)`;
- `requestFlashLoan(...)` - through the callback `executeOperation(...)`;

The three above functions lack the `refundDust(...)`, hence leftover tokens will stay on the contract.

**Impact:** Users zapping through flows without `_refundDust(...)` lose residual token balances, causing silent value leakage.

**Recommendation:** Add consistent dust refunds after every code path that performs `_addLiquidity(...)`, ensuring leftover token balances are returned to the intended recipient.

**Status:** Fixed

**Client response:** Fixed with [95d6d306b0de5aecae0d1f435421c96394c3c162](https://github.com/grape-finance/BB-Custom-contracts/commit/95d6d306b0de5aecae0d1f435421c96394c3c162)

### [M-07] Flash loan functionality in LPZapper always reverts

**Files:** [`LPZapper.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/9ccb11be9c4e21a63042ea4fcc56a0c1b654a4f0/contracts/LPZapper.sol)

**Description:**

In the final commit, the `LPZapper` contract introduced a reentrancy guard to mitigate potential attack vectors. However, the `nonReentrant` modifier was applied to both `requestFlashLoan(...)` and `executeOperation(...)`. Since flash loans necessarily re-enter the contract through `executeOperation(...)`, this setup causes every flash loan attempt to revert, making the functionality unusable.

**Impact:** Flash loan functionality is permanently broken and cannot be executed.

**Recommendation:** Remove the `nonReentrant` modifier from `executeOperation(...)`.

**Status:** Fixed

**Client response:** Resolved in [f484e550e31e8d9775bfb49db423c04bb49bbb5a](https://github.com/grape-finance/BB-Custom-contracts/tree/f484e550e31e8d9775bfb49db423c04bb49bbb5a)

## Low Risk

### [L-01] Lack of whenNotPaused modifier on claimReward

**Files:** [`Staking.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Staking.sol#L182)

**Description:**

The `Staking.sol` contract allows stakers to claim their accrued rewards through the `claimReward()` function. While functions like `stake(...)`, `withdraw(...)` (which also calls `claimReward()` inside) are protected by the `whenNotPaused` modifier, the `claimReward()` isn’t.

**Impact:** Lack of consistent pausing protection mechanism in the contract.

**Recommendation:** It is recommended to place also the modifier on the `claimReward()`. To make it work it is advised to move the claiming code into an internal `_claimReward()` function and call it within all other public functions.

**Status:** Fixed

**Client response:** Fixed in [20c8b1fd26d655ad834738ae01af71187cce1edd](https://github.com/grape-finance/BB-Custom-contracts/commit/20c8b1fd26d655ad834738ae01af71187cce1edd)

### [L-02] Normalization function can underflow with tokens having more than 18 decimals

**Files:** [`PulseMinter.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/PulseMinter.sol#L117-L129)

**Description:**

The helper function `_normalizeTokenAmount(...)` converts a token’s amount into a standardized 18-decimals representation. It checks the token’s decimals using `IERC20Metadata(token).decimals()` and, if the token uses fewer than 18 decimals, multiplies the amount by `10 ** (18 - tokenDecimals)`. However, the function assumes that `tokenDecimals <= 18`. If a token reports more than 18 decimals, the subtraction (`18 - tokenDecimals`) will underflow, causing the transaction to revert. The root cause is the absence of a defensive check on tokens with higher decimal counts.

**Impact:** If a token with more than 18 decimals is ever added, this function would always revert, breaking minting of Esteem flows that rely on normalization.

**Recommendation:** Update `_normalizeTokenAmount(...)` to handle tokens with more than 18 decimals.

**Status:** Acknowledged

## Informational

### [I-01] CREATE2 vulnerability in favor token allows sell tax bypass

**Files:** [`Favor.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Favor.sol#L103)

**Description:**

In the `Favor.sol` contract, the `_update()` function applies a sell tax when tokens are transferred to a contract address. This mechanism identifies contract destinations by checking if the receiving address has code deployed.

```solidity
bool destinationIsContract = _to.code.length != 0;
// [...]
if (destinationIsContract) {
    taxAmount = (_value * sellTax) / MULTIPLIER;
}
```

However, this implementation is vulnerable to a CREATE2 bypass attack. An attacker can:

a. Pre-compute a contract address using CREATE2 opcode (without deploying any code);
b. Transfer tokens to this computed address (which passes the `code.length == 0` check and avoids the sell tax);
c. Deploy a contract to that same address using CREATE2;
d. The contract now controls tokens that were transferred without paying the intended sell tax;

**Impact:** This vulnerability allows users to, bypass the 50

**Recommendation:** There is no real prevention mechanism against this, yet the impact is minimal.

**Status:** Acknowledged

### [I-02] Protocol not prepared to handle base tokens with fee-on-transfer

**Files:** [`PulseMinter.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/PulseMinter.sol#L83-L101)

**Description:**

The protocol currently supports PLS, PLSx and pDAI as base tokens, and in the future more base tokens are planned to be supported. Currently the way how the protocol handles transfers does not support the fee-on-transfer tokens and hence all calculations which calculate Esteem or Favor token amounts accrued to users will not be valid.

**Impact:** Currently there is no impact. This issue is just to raise awareness that special care needs to be undertaken when introducing a new base token with fee-on-transfer feature.

**Recommendation:** If fee-on-transfer tokens are added in the future, calculate the actual received amount by checking the contract’s balance before and after transfers every time the token is transferred from the user and to other accounts, like the treasury.

**Status:** Acknowledged

### [I-03] UniTWAPOracle and LPOracle state can become stale if it is not updated

**Files:** [`PulseMinter.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/PulseMinter.sol#L195-L202), [`UniTWAPOracle.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/UniTWAPOracle.sol#L54-L82), [`LPOracle.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/LPOracle.sol#L104)

**Description:**

The `PulseMinter.updateEsteemRate()` function is intended to increment `esteemRate` by `dailyRateIncrease` (0.25 ether per hour), but the increase only occurs when the function is explicitly called. If no call is made, both the rate and epoch remain unchanged.

Similarly, in `UniTWAPOracle.update()` and `LPOracle.update()`, only approved users can trigger updates to refresh cumulative price data and compute new averages for `price0Average` and `price1Average`. If these updates are not invoked, the averages remain stale and `consult` may return outdated pricing.

**Impact:** If approved users fail to trigger updates, `esteemRate` and oracle price averages may become stale, leading to outdated state or inaccurate values being returned.

**Recommendation:** To ensure `esteemRate` and oracle values remain up to date without depending solely on manual calls, implement one of the following approaches:

a. Epoch-Based Auto-Update;

Modify the functions so that whenever they are called, they first check whether the epoch time has elapsed. If so, the update logic is executed automatically before continuing with the normal function logic.

2. External Automation (Cron Job / Keeper) Integrate with an off-chain automation system (e.g., Chainlink Keepers, Gelato, or server-based cron jobs) to ensure `updateEsteemRate()` and oracle `update()` functions are called once per hour. This guarantees consistent updates without relying on users.

**Status:** Acknowledged

**Client response:** Keeper job is planned. There is already a keeper job for seignorage allocation in place

### [I-04] Usage of two-step ownership transfer is recommended

**Original severity:** Best Practices

**Files:** [`Favor.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Favor.sol), [`Zapper.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Zapper.sol), [`MinterOracle.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/MinterOracle.sol), [`FavorTreasury.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/FavorTreasury.sol), [`Staking.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Staking.sol), [`Epoch.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Epoch.sol), [`Esteem.sol`](https://github.com/grape-finance/BB-Custom-contracts/blob/43fba5ed6a5f47d6ae997660a7fe656c57f44b30/contracts/Esteem.sol)

**Description:**

The `Ownable2Step` pattern is an improvement over the traditional `Ownable` pattern, designed to enhance the security of ownership transfer functionality in a smart contract. Unlike the original `Ownable` pattern, where ownership can be transferred directly to a specified address, the `Ownable2Step` pattern introduces an additional step in the ownership transfer process. Ownership transfer only completes when the proposed new owner explicitly accepts the ownership, mitigating the risk of accidental or unintended ownership transfers to mistyped addresses.

**Impact:** Without the `Ownable2Step` pattern, the contract owner might inadvertently transfer ownership to an unintended or mistyped address, potentially leading to a loss of control over the contract.

**Recommendation:** It is recommended to use either `Ownable2Step` or `Ownable2StepUpgradeable` depending on the smart contract.

**Status:** Fixed

**Client response:** Was implemented as proposed [9c5239aecb5d519b91df85ccccbd196fb5314f13](https://github.com/grape-finance/BB-Custom-contracts/commit/9c5239aecb5d519b91df85ccccbd196fb5314f13)
