**Auditors**

[Trust Security](https://twitter.com/trust__90)


---


# Findings

## High Risk
### TRST-H-1 Attacker can freeze profit withdrawals from V3 vaults
**Description:**
Users of Ninja can use Vault's `withdrawProfit()` to withdraw profits. It starts with the 
following check:
 
```solidity 
   if (block.timestamp <= lastProfitTime) {
      revert NYProfitTakingVault__ProfitTimeOutOfBounds();
      }

```
If attacker can front-run user's `withdrawProfit()` TX and set **lastProfitTime** to 
block.timestamp, they would effectively freeze the user's yield. That is indeed possible using 
the Vault paired strategy's `harvest()` function. It is permissionless and calls `_harvestCore()`. 
The attack path is shown in **bold**.

```solidity 
      function harvest() external override whenNotPaused returns (uint256 callerFee) {
            require(lastHarvestTimestamp != block.timestamp);
                uint256 harvestSeconds = lastHarvestTimestamp > 0 ? block.timestamp 
            - lastHarvestTimestamp : 0;
      lastHarvestTimestamp = block.timestamp;
                uint256 sentToVault;
          uint256 underlyingTokenCount;
     (callerFee, underlyingTokenCount, sentToVault) = _harvestCore();
            emit StrategyHarvest(msg.sender, underlyingTokenCount, 
                   harvestSeconds, sentToVault);
                }
```

```solidity
      function _harvestCore() internal override returns (uint256 callerFee, uint256 underlyingTokenCount,      uint256 sentToVault)
            {
        IMasterChef(SPOOKY_SWAP_FARM_V2).deposit(POOL_ID, 0);
            _swapFarmEmissionTokens();
                callerFee = _chargeFees();
                    underlyingTokenCount = balanceOf();
                       sentToVault = _sendYieldToVault();
            } 
```
```solidity
      function _sendYieldToVault() internal returns (uint256 sentToVault) {
         sentToVault = IERC20Upgradeable(USDC).balanceOf(address(this));
            if (sentToVault > 0) {
               IERC20Upgradeable(USDC).approve(vault, sentToVault);
            IVault(vault).depositProfitTokenForUsers(sentToVault);
                }
                  }
```
```solidity
      function depositProfitTokenForUsers(uint256 _amount) external nonReentrant {
         if (_amount == 0) {
            revert NYProfitTakingVault__ZeroAmount();
         }
        if (block.timestamp <= lastProfitTime) {
            revert NYProfitTakingVault__ProfitTimeOutOfBounds();
         }
        if (msg.sender != strategy) {
            revert NYProfitTakingVault__OnlyStrategy();
        }
            uint256 totalShares = totalSupply();
        if (totalShares == 0) {
            lastProfitTime = block.timestamp;
            return;
          }
            accProfitTokenPerShare += ((_amount * PROFIT_TOKEN_PER_SHARE_PRECISION) / totalShares);
               lastProfitTime = block.timestamp;
            // Now pull in the tokens (Should have permission)
            // We only want to pull the tokens with accounting
               profitToken.transferFrom(strategy, address(this), _amount);
            emit ProfitReceivedFromStrategy(_amount);
                }
```
**Recommended Mitigation:**
Do not prevent profit withdrawals during lastProfitTime block.

**Team response:**
Accepted and removed.



### TRST-H-2 Lack of child rewarder reserves could lead to freeze of funds
**Description:**
In ComplexRewarder.sol, `onReward()` is used to distribute rewards for previous time period, 
using the complex rewarder and any child rewarders. If the complex rewarder does not have 
enough tokens to hand out the reward, it correctly stores the rewards owed in storage. 
However, child rewarded will attempt to hand out the reward and may revert:

```solidity 
   function onReward(uint _pid, address _user, address _to, uint, uint _amt) external override onlyParent nonReentrant {
      PoolInfo memory pool = updatePool(_pid);
         if (pool.lastRewardTime == 0) return;
            UserInfo storage user = userInfo[_pid][_user];
            uint pending;
         if (user.amount > 0) {
              pending = ((user.amount * pool.accRewardPerShare) / ACC_TOKEN_PRECISION) - user.rewardDebt;
      rewardToken.safeTransfer(_to, pending);
         }
         user.amount = _amt;
         user.rewardDebt = (_amt * pool.accRewardPerShare) / 
            ACC_TOKEN_PRECISION;
      emit LogOnReward(_user, _pid, pending, _to);
      }
```
 Importantly, if the child rewarder fails, the parent's `onReward()` reverts too:
```solidity
      uint len = childrenRewarders.length();
         for (uint i = 0; i < len; ) {
      IRewarder(childrenRewarders.at(i)).onReward(_pid, _user, _to, 0, 
         _amt);
      unchecked {
         ++i;
         }
      }
```
In the worst-case scenario, this will lead the user's `withdraw()` call to V3 Vault, to revert.

**Recommended Mitigation:**
Introduce sufficient exception handling in the CompexRewarder.sol contract, so that 
`onReward()` would never fail.

**Team Response:**
Rejected. Child rewarders are not being used in the protocol and are out of the scope. We 
have kept the ability for them if needed, and they will be included in a future audit before 
use. We appreciate this being pointed out and will take care of the issue in future updates.


### TRST-H-3 Wrong accounting of user's holdings allows theft of reward

**Description:**
In `deposit()`, `withdraw()` and `withdrawProfit()`, `rewarder.onReward()` is called for reward 
bookkeeping. It will transfer previous eligible rewards and update the current amount user 
has:

```solidity
      user.amount = _amt;
      user.rewardDebt = (_amt * pool.accRewardPerShare) / ACC_TOKEN_PRECISION;
      user.rewardsOwed = rewardsOwed;
```

In `withdraw()`, there is a critical issue where `onReward()` is called too early:

```solidity
      // Update rewarder for this user
          if (address(rewarder) != address(0)) {
      rewarder.onReward(0, msg.sender, msg.sender, pending, user.amount);
      }
      // Burn baby burn
            _burn(msg.sender, _shares);
      // User accounting
                uint256 userAmount = balanceOf(msg.sender);
      // - Underlying (Frontend ONLY)
            if (userAmount == 0) {
               user.amount = 0;
         } else {
            user.amount -= r;
         }
```
The new **_amt** which will be stored in reward contract's **user.amount** is vault's **user.amount**, 
before decrementing the withdrawn amount. Therefore, the withdrawn amount is still 
gaining rewards even though it's no longer in the contract. Effectively it is stealing the 
rewards of others, leading to reward insolvency.
In order to exploit this flaw, attacker will deposit a larger amount and immediately withdraw 
it, except for one wei. When they would like to receive the rewards accrued for others, they 
will withdraw the remaining wei, which will trigger `onReward()`, which will calculate and 
send pending awards for the previously withdrawn amount. 


**Recommended Mitigation:**
Move the `onReward()` call to after user.amount is updated.

**Team response:**
Accepted and updated.


## Medium Risk
### TRST-M-1 Unsafe transferFrom breaks compatibility with 100s of ERC20 tokens
**Description:**
In Ninja vaults, the delegated strategy sends profit tokens to the vault using 
`depositProfitTokenForUsers()`. The vault transfers the tokens in using:
```solidity 
         // Now pull in the tokens (Should have permission)
          // We only want to pull the tokens with accounting
                profitToken.transferFrom(strategy, address(this), _amount);
          emit ProfitReceivedFromStrategy(_amount);

```
The issue is that the code doesn't use the `safeTransferFrom()` utility from SafeERC20. 
Therefore, profitTokens that don't return a bool in `transferFrom()` will cause a revert which 
means they are stuck in the strategy. 
Examples of such tokens are USDT, BNB, among hundreds of other tokens.

**Recommended Mitigation:**
Use `safeTransferFrom()` from SafeERC20.sol

**Team Response:**
Accepted. Excellent find. I can't believe we missed this.


### TRST-M-2 Attacker can force partial withdrawals to fail

**Description:**
In Ninja vaults, users call `withdraw()` to take back their deposited tokens. There is 
bookkeeping on remaining amount:

```solidity
      uint256 userAmount = balanceOf(msg.sender);
         // - Underlying (Frontend ONLY)
            if (userAmount == 0) {
            user.amount = 0;
         } else {
         user.amount -= r;
      }
```
If the withdraw is partial (some tokens are left), user.amount is decremented by r.

```solidity
      uint256 r = (balance() * _shares) / totalSupply();
```
Above, r is calculated as the relative share of the user's _shares of the total balance kept in 
the vault.

We can see that user.amount is incremented in deposit().

```solidity
      function deposit(uint256 _amount) public nonReentrant {
      …
            user.amount += _amount;
      …
         }
```
The issue is that the calculated r can be more than _amount , causing an overflow in 
`withdraw()` and freezing the withdrawal. All attacker needs to do is send a tiny amount of 
underlying token directly to the contract, to make the shares go out of sync.

**Recommended Mitigation:**
Redesign **user** structure, taking into account that balance of underlying can be externally 
manipulated

**Team Response:**
Accepted after further investigation. we agreed to remove the double accounting 
(user.amount) and to dynamically calculate the value from the users share balance * price 
per share. We added the public view function getUserUnderlyingBalance to assist (which 
also allows dynamic underlying decimals).



### TRST-M-3 Rewards may be stuck due to unchangeable slippage parameter
**Description:**
In NyPtvFantomWftmBooSpookyV2StrategyToUsdc.sol, MAX_SLIPPAGE is used to limit 
slippage in trades of BOO tokens to USDC, for yield:
```solidity
      function _swapFarmEmissionTokens() internal { IERC20Upgradeable boo = IERC20Upgradeable(BOO);
            uint256 booBalance = boo.balanceOf(address(this));
      if (booToUsdcPath.length < 2 || booBalance == 0) {
         return;
      }
         boo.safeIncreaseAllowance(SPOOKY_ROUTER, booBalance);
             uint256[] memory amounts = 
      IUniswapV2Router02(SPOOKY_ROUTER).getAmountsOut(booBalance, booToUsdcPath);
          uint256 amountOutMin = (amounts[amounts.length - 1] * MAX_SLIPPAGE) / PERCENT_DIVISOR;
            IUniswapV2Router02(SPOOKY_ROUTER).swapExactTokensForTokensSupportingFeeOnTransferTokens( booBalance, amountOutMin, booToUsdcPath, address(this), block.timestamp );
                }
```
If slippage is not satisfied the entire transaction reverts. Since **MAX_SLIPPAGE** is constant, it 
is possible that harvesting of the strategy will be stuck, due to operations leading to too high 
of a slippage. For example, strategy might accumulate a large amount of BOO, or `harvest()` 
can be sandwich-attacked.

**Recommended Mitigation:**
Allow admin to set slippage after some timelock period.

**Team Response:**
Accepted. We converted MAX_SLIPPAGE to maxSlippage, a uint256 that the ADMIN Multisig 
role can update. We decided against a timelock, as we may need to change it once, unlock 
an individual harvest issue and put it back before the next harvest.


### TRST-M-4 truncation in reward calculation could cause leakage of rewards
**Description:**
In ComplexRewarder.sol, `updatePool()` call updates values in the specified pool.
```solidity
      uint lpSupply = IVault(VAULT).balance();
          if (lpSupply > 0) {
      uint time = block.timestamp - pool.lastRewardTime;
                uint reward = totalAllocPoint == 0 ? 0 : ((time * rewardPerSecond * 
                  pool.allocPoint) / totalAllocPoint);
               pool.accRewardPerShare = pool.accRewardPerShare + uint128((reward * 
          ACC_TOKEN_PRECISION) / lpSupply);
      }
```
**reward** could be a fairly large number. The decimals of **reward * ACC_TOKEN_PRECISION** is 
10**30, because of how ACC_TOKEN_PRECISION is defined:

```solidity
         ACC_TOKEN_PRECISION = 10 ** (30 - decimalsRewardToken);
```

**lpSupply** is given in LP's decimals, but could be as small as 1. If **lpSupply** is 1, **reward** of 
2**28 = 268435456 will be enough to cause uint128 overflow of the product (10 * * 30 is 100 
bits). Therefore, it is shown that this calculation is not safe under 128-bit math. The impact
would be **pool.accRewardPerShare** increasing by a tiny amount in relation to the correct 
amount.

**Recommendation:**
Use a larger int type to handle the above calculation. 

**Team response**
Accepted. We updated to uint192 and can move to uint256 if you prefer

### TRST-M-5 potential overflow in reward accumulator may freeze functionality

**Description:** 
Note the above description of `updatePool()` functionality. We can see that **accRewardPerShare** is only allocated 128 bits in **PoolInfo:**
```solidity
      struct PoolInfo {
          uint128 accRewardPerShare;
            uint64 lastRewardTime;
               uint64 allocPoint;
```
Therefore, even if truncation issues do not occur, it is likely that continuous incrementation 
of the counter would cause **accRewardPerShare** to overflow, which would freeze vault 
functionalities such as withdrawal.
 
**Recommended Mitigation:**
Steal 32 bits from lastRewardTime and 32 bits from allocPoint to make the accumulator have 
192 bits, which should be enough for safe calculations.


**Team response:**
Accepted. We updated PoolInfo.accRewardPerShare to uint256.

## Low Risk
### TRST-L-1 when using fee-on-transfer tokens in VaultV3, capacity is limited below underlyingCap

**Description:**
Vault V3 documentation states it accounts properly for fee-on-transfer tokens. It calculates 
actual transferred amount as below:
```solidity
      uint256 _pool = balance();
           if (_pool + _amount > underlyingCap) {
      revert NYProfitTakingVault__UnderlyingCapReached(underlyingCap);
            }
      uint256 _before = underlying.balanceOf(address(this));
            underlying.safeTransferFrom(msg.sender, address(this), _amount);
               uint256 _after = underlying.balanceOf(address(this));
                   _amount = _after - _before;
```
A small issue is that underlyingCap is compared to the _amount before correction for actual 
transferred amount. Therefore, it cannot actually be reached, and limits the maximum 
capacity of the vault to underlyingCap minus a factor of the fee %.

**Recommended Mitigation:**
Move the underlyingCap check to below the effective _amount calculation

**Team Response:**
Accepted and updated.




### TRST-L-2 Redundant checks in Vault V3
**Description:**
`depositProfitTokenForUsers()` and `withdrawProfit()` contain the following check:
```solidity
    if (block.timestamp <= lastProfitTime) {
       revert NYProfitTakingVault__ProfitTimeOutOfBounds();
          }
```
However, lastProfitTime is only ever set to block.timestamp. Therefore, it can never be 
larger than block.timestamp.

**Recommended Mitigation:**
It would be best in terms of gas costs and logical clarity to change the comparison to !=

**Team Response:**
Rejected. While valid, this change introduced significant errors during hardhat tests. As it 
can not impact production, we have left it as is.


### TRST-L-3 Strategy may be initialized by attacker
**Description:**
In NyPtvFantomWftmBooSpookyV2StrategyToUsdc.sol, `initialize()` is used to bootstrap the 
strategy. However, there is no caller check, which means it is only safe if proxy is upgraded 
to the strategy and immediately calls `initialize()`, using `upgradeToAndCall()` proxy 
functionality. Otherwise, attacker may call it themself and pass malicious values for treasury 
and other important parameters.

**Recommended Mitigation:**
Consider adding initialization protection. For example, during construction set an immutable 
to be the deployer address, which is the only one that is allowed to call `initialize()`.


**Team response:**
Rejected.


### TRST-L-4 Hard-coding Uniswap path assumptions
**Description:**
In NyPtvFantomWftmBooSpookyV2StrategyToUsdc.sol, estimateHarvest() is used to check if 
harvesting is profitable.
```solidity
      function estimateHarvest() external view override returns (uint256 profit, uint256 callFeeToUser) {
         uint256 pendingReward = 
         IMasterChef(SPOOKY_SWAP_FARM_V2).pendingBOO(POOL_ID, address(this));
            uint256 totalRewards = pendingReward + 
      IERC20Upgradeable(BOO).balanceOf(address(this));
      if (totalRewards != 0) {
         profit += 
      IUniswapV2Router02(SPOOKY_ROUTER).getAmountsOut(totalRewards, booToUsdcPath)[1];
      }
               profit += IERC20Upgradeable(USDC).balanceOf(address(this));
            uint256 usdcFee = (profit * totalFee) / PERCENT_DIVISOR;
         callFeeToUser = (usdcFee * callFee) / PERCENT_DIVISOR;
      profit -= usdcFee;
      }
```
Note that the code assumes `getAmountsOut()` will return USDC amount at index [1]. That is 
indeed the case right now, as **booToUsdcPath = [BOO, USDC];** However, it is an unnecessary 
coupling in code. `_swapFarmEmissionTokens()` handles the path correctly:

```solidity
            uint256[] memory amounts = 
         IUniswapV2Router02(SPOOKY_ROUTER).getAmountsOut(booBalance, booToUsdcPath);
      uint256 amountOutMin = (amounts[amounts.length - 1] * MAX_SLIPPAGE) / PERCENT_DIVISOR;
```

**Recommended mitigation**
Make the code more futureproof by refactoring `estimateHarvest()` to act similarly to 
`_swapFarmEmissionTokens()`.

**Team response**
Accepted, updated


### TRST-L-5 Strategy deployer has privileges intended only for multisig addresses.
**Description:** 
When V3 Strategy is initialized, different roles are given to privileged addresses:
```solidity
         for (uint256 i = 0; i < _strategists.length; i++) {
            _grantRole(STRATEGIST, _strategists[i]);
               }
               _grantRole(DEFAULT_ADMIN_ROLE, msg.sender); // @audit - Is this a security risk? Default admin roles is REALLY powerful! Consider removing
            _grantRole(DEFAULT_ADMIN_ROLE, _multisigRoles[0]);
         _grantRole(ADMIN, _multisigRoles[1]);
         _grantRole(GUARDIAN, _multisigRoles[2]);
 ```
As the previous @audit note says, it is really important to not permit msg.sender to be 
**DEFAULT_ADMIN_ROLE**. In fact, this role gives arbitrary power to the EOA, to remove the 
multisig, to assign itself any of the roles, and so on. In a way, it makes the multisig be just for 
show. Since Strategy is effectively responsible to hold the vault's funds, and can be 
upgraded, this represents a worrying rug pull potential.

**Team response:**
Reduced deployer access to GUARDIAN means they do not have any upgrade capability. 
They can pause contracts and perform STRATEGIST/KEEPER roles. We have also added a 
revokeDeployer function which GUARDIANS and above can use to remove the deployers 
access.


### TRST-L-6 Strategy upgrade cooldown is set too low
 **Description:** 
**UPGRADE_TIMELOCK** defines how long admin must wait before upgrading the contract. 
```solidity
      function _authorizeUpgrade(address) internal override {
            _atLeastRole(DEFAULT_ADMIN_ROLE);
      require(upgradeProposalTime + UPGRADE_TIMELOCK < block.timestamp);
          clearUpgradeCooldown();
             }
```
It is currently set to 1 hour, which is far too low in relation to its importance and funds at 
risk. We recommend a value of 24 hours at minimum to give the community a chance to 
respond to a malicious upgrade risk.

**Team response:**
Accepted, changed to 36 hours.


## Informational
### Make greater use of immutable variables
Immutable variables are guaranteed to be read only during a contract's lifetime. In V3 vault, 
several variables are not immutable but should never change: **profitToken**, 
**constructionTime**, **underlying**. 

**Team response**
Accepted & done.

### Emission of events during state changes
To satisfy the objective of transparency, it is recommended to emit events when any change 
of importance takes place. In V3 Strategies, `updateSecurityFee()`, `updateTreasury()` and the 
important upgrade related functions `clearUpgradeCooldown()` and 
`initiateUpgradeCooldown()` do not emit events.

**Team response**
Accepted & done.

### Lack of zero-address checks
It is a standard practice to validate important addresses for zero value. `updateTreasury()` 
does not conform to this best practice.

**Team response**
Accepted & done