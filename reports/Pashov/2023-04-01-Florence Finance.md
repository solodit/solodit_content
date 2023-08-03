**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## High Risk

### [C-01] Anyone can move `EURS` tokens that the user allowed `FlorinTreasury` to spend

**Impact:**
High, as funds will be moved from a user's wallet unwillingly

**Likelihood:**
High, as it requires no preconditions and is a common attack vector

**Description**

The `depositEUR` method in `FlorinTreasury` looks like this:

```solidity
function depositEUR(address from, uint256 eurTokens) external whenNotPaused {
    eurTokens = Util.convertDecimals(eurTokens, 18, Util.getERC20Decimals(eurToken));
    SafeERC20Upgradeable.safeTransferFrom(florinToken, from, address(this), eurTokens);
    emit DepositEUR(_msgSender(), from, eurTokens);
}
```

The problem is that the `from` argument is user controlled, so anyone can check who has allowed the `FlorinTreasury` contract to spend his tokens and then pass that address as the `from` argument of the method. This will move `eurTokens` amount of `EURS` tokens from the exploited user to the `FlorinTreasury` contract, even though the user did not do this himself. The `depositEUR` method is expected to be called by `LoanVault::repayLoan` or `LoanVault::depositRewards`, where the user should first approve the `FlorinTreasury` contract to spend his `EURS` tokens. This is especially problematic if the user set `type(uint256).max` as the allowance of the contract, because in such case all of his `EURS` balance can be drained.

**Recommendations**

Use `msg.sender` instead of a user-supplied `from` argument, so tokens can only be moved from the caller's account.

## High Risk

### [H-01] Stakers/vault depositors can be front-run and lose their funds

**Impact:**
High, as it results in a theft of user assets

**Likelihood:**
Medium, as it works only if the attacker is the first staker

**Description**

Let's look at the following example:

1. The `FlorinStaking` contract has been deployed, unpaused and has 0 staked $FLR in it
2. Alice sends a transaction calling the `stake` method with `florinTokens == 10e18`
3. Bob is a malicious user/bot and sees the transaction in the mempool and front-runs it by depositing 1 wei of $FLR, receiving 1 wei of shares
4. Bob also front-runs Alice's transaction with a direct `ERC20::transfer` of 10e18 $FLR to the `FlorinStaking` contract
5. Now in Alice's transaction, the code calculates Alice's shares as `shares = florinTokens.mulDiv(totalShares_, getTotalStakedFlorinTokens(), MathUpgradeable.Rounding.Down);`, where `getTotalStakedFlorinTokens` returns `florinToken.balanceOf(address(this))`, so now `shares` rounds down to 0
6. Alice gets minted 0 shares, even though she deposited 10e18 worth of $FLR
7. Now Bob back-runs Alice's transaction with a call to `unstake` where `requestedFlorinTokens` is the contract's balance of $FLR, allowing him to burn his 1 share and withdraw his deposit + Alice's whole deposit

This can be replayed multiple times until the depositors notice the problem.

**Note:** This absolute same problem is present with the ERC4626 logic in `LoanVault`, as it is a common vulnerability related to vault shares calculations. OpenZeppelin has introduced a way for mitigation in version 4.8.0 which is the used version by this protocol.

**Recommendations**

UniswapV2 fixed this with two types of protection:

[First](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L119-L121), on the first `mint` it actually mints the first 1000 shares to the zero-address

[Second](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L125), it requires that the minted shares are not 0

Implementing them both will resolve this vulnerability.

## Medium Risk

### [M-01] The Chainlink price feed's input is not properly validated

**Impact:**
High, as it can result in the application working with an incorrect asset price

**Likelihood:**
Low, as Chainlink oracles are mostly reliable, but there has been occurrences of this issue before

**Description**

The code in `LoanVault::getFundingTokenExchangeRate` uses a Chainlink price oracle in the following way:

```solidity
(, int256 exchangeRate, , , ) = fundingTokenChainLinkFeeds[fundingToken].latestRoundData();

if (exchangeRate == 0) {
    revert Errors.ZeroExchangeRate();
}
```

This has some validation but it does not check if the answer (or price) received was actually a stale one. Reasons for a price feed to stop updating are listed [here](https://ethereum.stackexchange.com/questions/133242/how-future-resilient-is-a-chainlink-price-feed/133843#133843). Using a stale price in the application can result in wrong calculations in the vault shares math which can lead to an exploit from a bad actor.

**Recommendations**

Change the code in the following way:

```diff
- (, int256 exchangeRate, , , ) = fundingTokenChainLinkFeeds[fundingToken].latestRoundData();
+ (, int256 exchangeRate, , uint256 updatedAt , ) = fundingTokenChainLinkFeeds[fundingToken].latestRoundData();

- if (exchangeRate == 0) {
+ if (exchangeRate <= 0) {
    revert Errors.ZeroExchangeRate();
}

+ if (updatedAt < block.timestamp - 60 * 60 /* 1 hour */) {
+   pause();
+}
```

This way you will also check for negative price (as it is of type `int256`) and also for stale price. To implement the pausing mechanism I proposed some other changes will be needed as well, another option is to just revert there.

### [M-02] The ERC4626 standard is not followed correctly

**Impact:**
Medium, as functionality is not working as expected but without a value loss

**Likelihood:**
Medium, as multiple methods are not compliant with the standard

**Description**

As per EIP-4626, the `maxDeposit` method "MUST factor in both global and user-specific limits, like if deposits are entirely disabled (even temporarily) it MUST return 0.". This is not the case currently, as even if the contract is paused, the `maxDeposit` method will still return what it usually does.

When it comes to the `decimals` method, the EIP says: "Although the convertTo functions should eliminate the need for any use of an EIP-4626 Vault’s decimals variable, it is still strongly recommended to mirror the underlying token’s decimals if at all possible, to eliminate possible sources of confusion and simplify integration across front-ends and for other off-chain users."
The `LoanVault` contract has hardcoded the value of 18 to be returned when `decimals` are called, but it should be the decimals of the underlying token (it might not be 18 in some case maybe).

**Recommendations**

Go through [the standard](https://eips.ethereum.org/EIPS/eip-4626) and follow it for all methods that `override` methods from the inherited ERC4626 implementation.

### [M-03] User exit/claim methods should not have a `whenNotPaused` modifier

**Impact:**
High, as user funds can be left stuck in the contract

**Likelihood:**
Low, as it requires a malicious or a compromised owner

**Description**

The `unstake` and `claim` methods in `FlorinStaking` have a `whenNotPaused` modifier and the same is true for the `redeem` and `_withdraw` methods in `LoanVault`. This opens up an attack vector, where the protocol owner can decide if the users are able to withdraw/claim any funds from it. There is also the possibility that an admin pauses the contracts and renounces ownership, which will leave the funds stuck in the contract forever.

**Recommendations**

Remove the `whenNotPaused` modifier from user exit/claim methods in the protocol or reconsider the `Pausable` integration in the protocol altogether.

### [M-04] The `apr` and `fundingFee` percentage values are not constrained

**Impact:**
High, as this can result in the contract being in a state of DoS or in 0 rewards for users

**Likelihood:**
Low, as it requires a malicious or a compromised owner, or a big mistake on the owner side

**Description**

Neither the `setApr` nor the `setFundingFee` methods have input validations, checking if the percentage value arguments are too big or too small. A malicious/compromised owner, or one that does a "fat-finger", can input a huge number as those methods' argument, which will result in a state of DoS for the contract. Also the values of 0 or 100 (percentage) are valid as well, but shouldn't be - they will result in either 0 rewards for users or high fees (100% fees are not possible because of the slippage check in `approveFundingAttempt`).

**Recommendations**

Add a min and max value checks in both the `setApr` and `setFundingFee` methods in `LoanVault`.

### [M-05] The protocol uses `_msgSender()` extensively, but not everywhere

**Impact:**
Low, because protocol will still function normally, but an expectedly desired types of transactions won't work

**Likelihood:**
High, because it is certain that he issue will occur as code is

**Description**

The code is using OpenZeppelin's `Context` contract which is intended to allow meta-transactions. It works by using doing a call to `_msgSender()` instead of querying `msg.sender` directly, because the method allows those special transactions. The problem is that the `onlyDelegate` and `onlyFundApprover` modifiers in `LoanVault` use `msg.sender` directly instead of `_msgSender()`, which breaks this intent and will not allow meta-transactions at all in the methods that have those modifiers, which are one of the important ones in the `LoanVault` contract.

**Recommendations**

Change the code in the `onlyDelegate` and `onlyFundApprover` modifiers to use `_msgSender()` instead of `msg.sender`.

### [M-06] Multiple centralization attack vectors are present in the protocol

**Impact:**
High, as it can result in a rug from the protocol owner

**Likelihood:**
Low, as it requires a compromised or a malicious owner

**Description**

The protocol owner has privileges to control the funds in the protocol or the flow of them.

The `mint` function in `FlorinToken` is callable by the contract owner, which is `FlorinTreasury`, but `FloriNTreasury` has the `transferFlorinTokenOwnership` method. This makes it possible that the `FlorinTreasury` deployer to mint as many `FlorinToken` tokens to himself as he wants, on demand.

The `withdraw` method in `FlorinStaking` works so that the owner can move all of the staked `florinToken` tokens to any wallet, including his.

The `setMDCperFLRperSecond` method in `FlorinStaking` works so that the owner can stop the rewards at any time or unintentionally distribute them in an instant.

The method `setFundingTokenChainLinkFeed` allows the owner to set any address as the new Chainlink feed, so he can use an address that he controls and returns different prices based on rules he decided.

**Recommendations**

Consider removing some owner privileges or put them behind a Timelock contract or governance.

## Low Risk

### [L-01] If too many funding tokens are whitelisted then removal might become impossible

The `setFundingToken` method in `LoanVault` allows the owner to whitelist as many funding tokens as he wants, pushing them to an unbounded array. The problem is that if he whitelists too many tokens, then the array will grow too big and removing a value (for example the last one) from the array might cost too much gas, even more than the block gas limit, resulting in impossibility of removing some funding tokens from the whitelist. To fix this you should put an upper limit on the `_fundingTokens` array size, for example 50.

### [L-02] Number of registered loan vaults should be capped

The owner of the `LoanVaultRegistry` might call `registerLoanVault` too many times, which will make the `loanVaultIds` array very big. Calling `getLoanVaultIds` on-chain might result in a revert if the gas required for the call is too much (more than the block gas limit for example). It is recommended that you limit the number of loan vaults that can be registered, for example a maximum of 200.

### [L-03] Allowance check gives a false sense of security

The `LoanVault::createFundingAttempt` method contains the following allowance check:

```solidity
if (fundingToken.allowance(_msgSender(), address(this)) < fillableFundingTokenAmount) {
    revert Errors.InsufficientAllowance();
}
```

But it does not do a `transferFrom` for the `fundingToken` by itself. This means that the call to the method can be back-ran with an allowance revoke transaction. This will invalidate the check and later revert when the funding attempt is executed, so you are better off removing the check.
