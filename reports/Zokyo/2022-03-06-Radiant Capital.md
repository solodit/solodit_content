**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Wrong comparison condition. 

**Description**

MultiFee Distribution.sol: _withdraw Expired LocksFor(), line 1147. 
The "if " statement validates that in case relock is not disabled by the user during the relock action transaction, withdrawn funds should be restaked. However, the contract compares variable 'isRelockAction' to "false" instead of "true", which doesn't correspond to the logic of the contract. For example, if_withdrawExpired LocksFor() is called within function withdrawExpiredLocks For(), it will have parameter 'isRelockAction' set to "false" which means that funds should be transferred to the user, not restaked. However, due to the wrong condition, funds will be restaked. The issue is marked as critical since it is currently impossible for users to withdraw expired locks to their balances. 

**Recommendation**: 

Verify the comparison logic OR compare 'isRelockAction' to true instead. 

**Post-audit**. 
Comparison condition wasn't changed. Thus, function withdraw ExpiredLocksFor() still performs a relock action, however another function was added to perform withdrawing without relock.

### Functions are not restricted. 

**Description**

1) RadiantOFT.sol: setLpMintComplete(), setMinter(). 
Functions are not restricted, meaning anyone can call and stop the minting or set a minter address. While function setMinter() can be called only once, there are no deployment scripts to verify that the team will call this function right after the deployment. Thus, anyone can call setMinter() before the deployer and become a minter, and anyone can call setLpMintComplete() and stop minting at any time. 
2) ChefIncentivesController.sol: handleActionAfter(). 
The function used to be restricted, so only valid RTokens or LP multiFeeDistribution can be called. However, at a commit 3df651a62b0446ff2113fe3f1a8506e3f7fff0f7 function is not restricted and can be called by anyone.

**Recommendation**: 

Restrict the function, so that only certain addresses with specific role can call them. 

**Post-audit**: 
Functions were removed and the minting system was changed. Now function mint() can be called only once and is called in the deployment script.

### Possible underflow when handling after-actions for token. 

**Description**

When ATokens are minted, transferred, or burnt, 
ChefIncentivesController.handleActionAfter() is called, and it updates balances and total supplies in mappings 'userInfo` and `poolInfo`. 
However, during disqualification, the contract calls ChefIncentivesController.disqualifyUser() when the user's balance is set to 0 (by calling_handleActionAfter ForToken() where_balance = 0). 
However, ATokens are still present on the user's balance. When he tries to transfer or burn them, the contract calls handleActionAfter() and performs a subtraction from 0. That might cause an underflow, granting users an enormous balance and affecting the correctness of rewards distribution. 

**Recommendation**: 

Do not set userInfo[user].amount to Ø during disqualification in order to avoid underflows when user transfers or burns his tokens. 

**Post-audit**: 

Subtraction is validated now. Also since user's current amount is subtracted from total supply, it won't cause an underflow.

### Owner is able to withdraw staking token. 

**Description**

MultiFee Distribution.sol: recoverERC20(). 
The owner can directly access users' funds and withdraw their tokens anytime since the owner can't recover only reward tokens. As a result, in the case of the private key exploit (of an owner account), users' funds can be withdrawn directly from the contract. That's why it is recommended to validate that the provided 'tokenAddress is not a staking token, in order to exclude a centralization risk. 

**Recommendation**: 

Validate that 'tokenAddress' is not a staking token in the function. 

**Post-audit**. 

Function recoverERC20() was removed.

### Possible frontrun via flashloan attack. 

**Description**

Uniswap PoolHelper.sol: swap(), line 188. 
LiquidityZap.sol: addLiquidityWETHOnly(), line 113; addLiquidityETHOnly(), line 147. 
BalancerPoolHelper.sol: swap(), line 311. 
The contract does not check the actual amountOut' value during the tokens swap. Usually, the swap in protocols such as Uniswap or Balancer requires a parameter 'minimumLiquidity` and checks an actual output amount of token so that it will be not less than minimumLiquidity'. This parameter is essential in order to protect users from too high slippage and possible manipulations of the pool's reserves through frontrun. It is highly recommended to check the 'amountOut' and protect users from any possible pool manipulations. 

**Recommendation**: 

Pass 'minimumLiquidity as a function parameter and check that amountOut is greater or equal to 'minimum Liquidity`. 

**Post-audit**. 

The main function of LockZap.sol contains a slippage check. As for other contracts, it was verified by the Radiant team, that users shouldn't interact with LiquidityZap, BalancerPoolHelper, Uniswap PoolHelper directly.

### Price can be manipulated via flashloans. 

**Description**

PriceProvider.sol: update(), getToken Price(), getLpToken Price(). 
Since the contract calculates the price based on a single DEX (Uniswap or Balancer), it can be manipulated via flashloans. There are several parts of the protocol where price manipulation is profitable for an attacker. 
1. Disqualifier.sol: getBaseBounty(). 
When a provided user can be disqualified, the contract calculates a bounty for him based on the price of RDNT. Though the calculated bounty can't exceed 'maxBaseBounty', the attacker can manipulate the price to receive the maximum allowed bounty. 
2. RadiantOFT.sol: _getBridgeFee(). 
The contract utilizes price to calculate the fee for using the bridge. In this case, an attacker can manipulate the price to pay lower fees. 
3. EligibilityDataProvider.sol: _locked UsdValue(). 
Price is used to calculate the locked user's amount. In this case, an attacker may use flashloan to manipulate the value of the user's locked funds and affect the disqualification of user. 
The issue is marked as high due to point 3, where users can be affected and disqualified due to price manipulation. 

**Recommendation**: 

Consider using off-chain oracles for price calculation OR use several sources for retrieving price (such as Uniswap, Balancer, Curve) and calculate a Volume Weighted Average Price OR calculate a historical price price based on a single source of price (TWAP). This can help reduce the chances of flashloan attack on the PriceProvider. 

**Post-audit**: 
TWAP oracles based on Uniswap V2 and Uniswap V3 were implemented.

### Transfer without enough token on contract balance. 

**Description**

Leverager: loop() 
The function will revert with isBorrow parameter = true since it cannot transfer fee without any tokens on balance. Also it seems like it is charging funds twice: before the for() cycle and in it. 

**Recommendation**:

Transfer fee when a contract has enough tokens on balance.

## Medium Risk


### Staking and withdrawal operations might be blocked. 

**Description**

MultiFee Distribution.sol: _stake(), line 644,_withdrawExpiredLocks For(), line 1134. 
During staking and withdrawing funds, a 'beforeLockUpdate hook is called on the Incentives Controller. This hook checks if a user is to be disqualified. For this purpose, the contract performs another external call to Disqualifier.sol, function processUser(). Inside this function, the contract calls an internal function of Disqualifier, _processUserWithBounty(). It has a "require" which will revert if the storage variable 'DISABLED' is set to true. Thus due to this statement on Disqualifier.sol, staking and withdrawing of tokens on MultiFee Distribution.sol might be blocked. 
Further, in_processUserWithBounty(), an external call is performed to ChefIncentivesController.disqualifyUser(), where there are two checks which can also block the operations (Lines 489, 491). 

**Recommendation**: 

Do not revert a whole stake or withdraw transaction due to require statements in the Disqualifier.sol and ChefIncentives Controller.sol. 

**Post-audit**: 

The team removed validations that might prevent staking and withdrawing. However, if a certain user is ineligible for rewards, a bounty should be claimed for him before staking or withdrawing.

### User lock and earning might contain empty elements. 

**Description**

MultiFee Distribution.sol: _cleanWithdrawableLocks(), line 1123. 
MultiFee Distribution.sol: withdraw(), line 775 
After the lock withdrawal, the contract deletes it with "delete" operator. However, deleting elements of an array with this operator only sets the value of an element to 0, and the element will still be present in the array. Due to this, the array `userLocks [user]` might contain empty elements, iteration through which will increase gas spending. This is why it is recommended to shift elements in the array to the left by 1 when any element is removed from the array. The issue is marked as medium-risk, since if there are too many elements in an array, a transaction might revert due to an "out of gas" error and even not fit in one block. 

**Recommendation**: 

Shift elements in the array when removing an element, so that there are no empty elements in the array. 

**Post-audit**: 

Both "userLocks" and "userEarnings" are cleaned now.

### Tokens are not approved to a new pool helper. 

**Description**

LockZap.sol: setPoolHelper(). 
After the LockZap.sol initialization, it requires granting an unlimited allowance to_poolHelper in WETH and RDNT tokens. However, when a new pool is set with the function setPoolHelper(), allowance is not granted to a new pool helper. Thus the protocol couldn't operate with any new pool helper due to the lack of allowance. 

**Recommendation**: 

Consider granting allowance before each transfer OR grant unlimited allowance in the setter as well. 

**Post-audit**: 
Approval is performed before each transfer now.

### Iteration though the whole storage array. 

**Description**

MultiFee Distribution.sol: _cleanWithdrawableLocks(), line 1119; withdraw(), line 754. 
After the withdrawal of locks or earnings, the contract performs an iteration through the whole storage array. For example, if a user creates a significant amount of locks, iteration might consume more gas than can fit in one transaction. Thus, the user's funds might get stuck. The issue is marked a medium-risk since it might affect certain users, not the whole protocol. However, it is still recommended not to allow users to create infinite elements in the array to mitigate this issue. 

**Recommendation**: 

Consider restricting the maximum amount of elements in storage arrays 
`userEarnings[user]` and `userLocks [onBehalfOf] OR set a minimal amount of tokens, which can be locked or minted per one time, so that a user can't create too many locks with low values. 

**Post-audit**: 

Restriction was added in function_cleanWithdrawableLocks(). User will have to call function multiple times if he has a great amount of locks.

### ETH might get stuck on the contract's balance. 

**Description**

1) LiquidityZap.sol: standardAdd(). 
BalancerPoolHelper.sol: zapTokens(). 
Function is marked as payable, allowing users to send Ether when calling it. However this function never interacts with msg.value, thus sending Ether to it has no effect. Thus any Ether, sent with this function, will be stuck on the contract. 
2) LockZap.sol: zapFromVesting(), line 118. 
In case parameter_borrow' is true, any sent Ether will get stuck on the contract's balance. 
Thus, in case_borrow is true, it should be validated that no Ether is sent with the function. 

**Recommendation**: 

1. Remove payable modifier OR add interaction with Ether in the function. 
2. Add a validation that in case '_borrow' is true, msg.value is equal to 0.


### Unlimited allowance. 

**Description**

LockZap.sol: constructor(), lines 53-54; setLpMfd(), line 66. 
Making an unlimited allowance of tokens might be potentially dangerous and lead to funds loss in case of exploitation. Though approval is performed on protocol contracts (Uniswap PoolHelper.sol or Balancer PoolHelper.sol and MultiFee Distribution.sol) these contracts are upgradeable. This is why approving tokens before each transfer on the amount necessary for a particular transfer is recommended. 

**Recommendation**: 
Approve tokens, necessary for transfer before each transfer operation instead of granting an unlimited allowance. 

**Post-audit**. 

Unlimited allowance was removed.

### Non-withdrawable locks can be withdrawn. 

**Description**

LockZap.sol: setPoolHelper(). 
After the LockZap.sol initialization, the contract grants an unlimited allowance to _poolHelper in WETH and RDNT tokens. However, when a new pool is set with the function setPoolHelper(), allowance is not granted to a new pool helper. Thus the protocol couldn't operate with any new pool helper due to the lack of allowance. 

**Recommendation**: 

Check the last element of the array with index locks.length - 1' in "if". 

**Post-audit**: 
Last element of locks is not checked now. User has to call function several times to withdraw a large amount of locks.

## Low Risk

### Share variables lack validation. 

**Description**

Disqualifier.sol: setHunterShare(). 
Eligibility DataProvider.sol: setRequired EthRatio(). 
Middle Fee Distribution.sol: setLpLocking Reward Ratio(), setOperation Expenses(). 
Leverager.sol: setFeePercent(). 
Leverager.sol: loop(), parameter 'borrowRatio (Should be validated not to exceed 10 ** 
BORROW_RATIO_DECIMALS'). 
Radiant OFT.sol: setFeeInfo(). 
StargateBorrow.sol: setXChain Borrow Fee Percent(). 
Some of the variables act as shares or use shares during calculations, however such variables lack validation, that they don't exceed 100%. For example, in Disqualifier.sol when setting HUNTER_SHARE, the contract should validate that the new value will not exceed 10000 (As division is performed with 10000 as a denominator in line 277). Issue is marked as low-risk since only owner can set these variables, though in case of invalid value it will affect the correctness of calculations in the protocol. 

**Recommendation**: 

Validate that share variable doesn't exceed 100%.

### Wrong hunter address is used. 

**Description**

Disqualifier.sol: processUserWithBounty(). 
Parameter_hunter' is never used. Based on the logic of the function and a similar function processUsersWithBounty(), where parameter '_hunter' is passed to the internal function, the same should happen in process UserWithBounty(). 

**Recommendation**: 

Pass parameter _hunter to the internal function instead of msg.sender OR remove 
parameter_hunter'. 

**Post-audit**: 

Contract was removed.

### Contracts can be initialized with wrong parameters. 

**Description**

MultiFee Distribution.sol: initialize(). 
TokenVesting.sol: initialize(), _notifyUnseenReward() 
1. MultiFee Distribution.sol: No parameters are being validated in initialize(). This might be a serious problem as addresses "mfdStats" and "priceProvider" do not have setters themselves as the variables REWARDS_DURATION, REWARDS_LOOKBACK, DEFAULT_LOCK_DURATION do not have setter either and are set only in initialize() method. 
2. TokenVesting.sol: rdntToken is not validated against zero address in initialize() method and there is no setter for rdntToken variable. 
3. Eligibility DataProvider.sol, Disqualifier.sol, Leverager.sol, PriceProvider.sol: all address parameters should be validated against zero addresses in constructor and initializer 
4. MultiFee Distribution.sol: _notify Unseen Reward(): At line 965 subtraction 
REWARDS_DURATION - REWARDS_LOOKBACK might underflow in case REWARDS_LOOKBACK is greater than REWARDS_DURATION. Thus the initiaize() function should verify that subtraction won't result in an underflow. 
5. RadiantOFT.sol: parameters_endpoint and_treasury in constructor should be validated against zero address. _fee should be validated not to exceed 1e4. 
6. StargateBorrow.sol: all address parameters. _xChainBorrow FeePercent not to exceed 1e4. 

**Recommendation**: 

Validate parameters in initialization of contracts. 

**Post-audit**: 

All validations were added.

### Vesting schedule can be overwritten and parameters are not validated. 

**Description**

TokenVesting.sol: addVest(). 
The owner is able to overwrite vesting, which is already ongoing. Any tokens from the previous vesting that weren't already claimed will get stuck on the contract. Though only the owner can call this function, already existing vestings can still be overwritten, leading to the loss of funds. Also, to prevent invalid vesting creation, it is recommended to validate parameters_amount' and '_duration against zero values, as in case '_duration' is zero, RDNT tokens still will be transferred. However, the claimer couldn't claim them due to validation in function claim(), line 53. 

**Recommendation**: 

Do not allow to overwrite vesting schedules which are already in progress and validate parameters_amount' and '_duration' not to be zero values. 

**Post-audit**: 

Parameters are validated and active vesting can't be overwritten.

## Informational

### Storage variables are never used. 

**Description**

AutoCompounder.sol: 'rdntToken', 'IpTokenAddress', 'baseToRdnt'. 
Eligibility DataProvider.sol: 'baseToken PriceInUsd ProxyAggregator'. 
Disqualifiers.sol: treasury`. 
MiddleFee Distribution.sol: 'minters', 'mintersAreSet'. 
Variables are never used in the contract's logic, and some variables are even never set. 
*baseToRdnt is initialized with zero addresses. The unused variable can mean that the contract is unfinished, so it is recommended not to have such variables. 

**Recommendation**: 

Remove unused variables OR finish contract's logic where variables will be used OR verify that these variables are necessary to other smart-contracts or the Dapp.

### Lack of events. 

**Description**

In order to track the historical changes of essential storage variables, it is recommended to emit events in setters on every change of variable. Thus the following setter function should emit an event. 
1. AutoCompounder.sol: addReward BaseTokens(), setRoutes(). 
2. Disqualifier.sol, EligibilityDataProvider.sol, MiddleFee Distribution.sol, Leverager.sol, RadiantOFT.sol, StargateBorrow.sol: all functions which start with "set". 

**Recommendation**: 

Emit events in setter functions.



### Storage constants should be used. 

**Description**

Disqualifier.sol: function_process UserWithBounty(), line 184; bountyForUser(), line 277, value "10000". 
Eligibility DataProvider.sol: required UsdValue(), line 159, value "1e4". 
MiddleFee Distribution.sol: mint(), line 171; forward Reward(), line 197, 204, value "1e4". 
MFDstats.sol: add Transfer(), line 80, 90, value "1e4". 
LockZap.sol: lines 105, 107, values "10000", "9500". 
Leverager.sol: lines 107, 117, 141, 154, value "1e4". 
RadiantOFT.sol: _getBridgeFee(), line 114, value "1e4". 
StargateBorrow.sol: getXChain Borrow FeeAmount(), line 105, value "1e4". 
In order to increase readability of the contracts, it is recommended to use storage constants instead of values directly. 

**Recommendation**: 

Use storage constants instead of values directly in code.

### Same tokens can be added to eligible tokens array. 

**Description**

EligibilityDataProvider.sol: add Token(). 
The owner is able to set the same token as eligible multiple times. Thus the same token will be added to the array 'eligibleTokens multiple times. The issue is marked as info since it doesn't affect the contract, as array is not used in any of its logic. However, it might be crucial for other parts of the protocol that the array doesn't contain repeatable tokens, so it is recommended to restrict the same token to be added to the array multiple times. 

**Recommendation**:

Validate that 'token' is not already added to array of eligible tokens. 

**Post-audit**. 
Function was removed.

### Comments don't correspond to the code. 

**Description**

1. MultiFee Distribution.sol: initialize(). 
The commentary section stated that the first token in the array of rewards must be a staking token. As a function has a parameter '_stakingToken', one may assume this token must be the first one in the rewards array (based on comments). However, another token, '_rdntToken', is pushed to the array instead. Thus, this doesn't correspond to the comments and should be verified by the team. 
Eligibility DataProvider.sol: lastEligibleTime(). 
The commentary section stated that the function returns locked RDNT and LP token values in ETH. However, the function's name and the return variable's name imply that the function returns the last eligible timestamp. 

**Recommendation**: 

1. Verify that _rdntToken' should be the first one in the rewards array and update comments OR verify that '_rdntToken' and '_stakingToken' are the same token OR update the code and push_staking Token' to the rewards array. 
2. Verify the correctness of the return value in function.

### Never used imports in multiple contacts. 

**Description**

Disqualifier.sol: 
ILendingPool, Locked Balance, IUniswapV2Router02, IUniswapV2Factory, IUniswapV2Pair, 
IChainlinkAggregator, IAToken. 
PriceProvider.sol: IERC20, SafeERC20. 
Contracts are imported in files, however they are never used. 

**Recommendation**: 

Remove unnecessary imports OR use them if needed. 

**Post-audit**. 

All unused imports were removed.

### Commented code. 

**Description**

MultiFee Distribution.sol: _withdraw Expired LocksFor(), line 1155. 
LiquidityZap.sol: standardAdd(), line 189; _addLiquidity(), lines 235-236. 
StargateBorrow.sol: setDAOTreasury(), line 94. 
Pre-production smart-contract should not contain commented code as it might mean that part of the contract's logic is unfinished. 

**Recommendation**: 

Remove or uncomment commented code.

### Available borrows Ether is not calculated for an actual borrower. LockZap.sol:  _executeBorrow(). 

**Description**

Function_execute Borrow() has a parameter '_onBehalf for which the availability of ETH to borrow is checked. However, the contract performs the actual borrow for msg.sender, which might mismatch with onBehalf parameter in case the function is called within zapWETH(). Because the user can pass an arbitrary '_onBehalf to zapWETH() function, removing this parameter and using msg.sender directly is recommended. The issue is marked as info, since LendingPool.sol still won't let borrow more than is available for msg.sender. 

**Recommendation**: 

Remove parameter_onBehalf and use msg.sender directly so that availability of Ether is checked for correct user. 

**Post-audit**: 

Availability is checked for '_onBehalf address. Since liquidity is staked for '_onBehalf, this might not be an issue that anyone can pass onBehalf and zap his funds (In case _onBehalf allowed LockZap to perform borrow on his behalf.)

### User might have access to tokens, stored on contract's balance. LockZap.sol: zapWETH(). 

**Description**

In case the user passes a parameter _from' equal to the address of LockZap.sol, WETH won't be transferred from msg.sender to address(this), and the contract will use WETH stored on LockZap's balance. The issue is marked as info, since the contract isn't supposed to store funds, however, such functionality should be verified. 

**Recommendation**: 

Verify that users should be able to pass 'from' as the address of LockZap.sol and use funds which could be stored on contract's balance. 

**Post-audit**: 

address(this) can't be passed when zapWETH is called() now.

### Disqualification mechanism should be verified. 

**Description**

Disqualifier.sol: _processUserWithBounty(). 
ChefIncentivesController.sol: disqualify User(), _isEligible(). 
In Disqualifier.sol (lines 167-168), the user will be disqualified from ChefIncentivesController.sol if the user's unlockable > 0 and either relock is disabled or the user's locked value is less than his collateral value in Lending Pool. 
Thus, the user is considered ineligible in case 'lastEligibleTime > block.timestamp && ! eligibility Provider.isEligible ForRewards(_user) (Disqualifier.sol, line 163.). However, in ChefIncentivesController.disqualifyUser(), the user is considered ineligible for rewards if lastEligibleTime < block.timestamp OR !eligibility Provider.isEligible ForRewards(_user) (It is checked in line 491 by calling!_isEligible()). Due to this mismatch, the transaction might fail. 
Also, disqualification may be called on ChefIncentives Controller in case relock is disabled or in case chef bounty > 0 (line 180). However, in this case, relock from ChefIncentivesController is not taken into account. Thus, if the user is ineligible due to relock, it will not disqualify him on ChefIncentives Controller, reverting the whole transaction due to check on line 491. 
In Disqualifier.sol, line 191, eligibility is also checked after locks might be withdrawn, which can affect the result of eligibility Provider.is Eligible For Rewards(). Thus, it should be verified if such functionality is correct, or eligibility should be tracked before the locks are withdrawn. 

**Recommendation**: 

Verify the correctness of the current disqualification mechanism and that its implementation corresponds to business logic. Make sure that transaction won't revert due to mismatching validations on Disqualifier.sol and ChefIncentives Controller.sol. 

**Post-audit**: 

Disqualification mechanism was updated. All logic, connected to disqualifications and bounty is executed in ChefIncentivesController or MFDPlus.

### Unnecessary validation checks and operations in Token Vesting. 

**Description**

TokenVesting.sol: initialize(), claim(). 
In TokenVesting.sol (line 29), the validation for msg.sender not being zero address is making no sense as zero address cant call any external methods. 
In TokenVesting.sol (line 66), the statement if(v.duration == 0) will never become true as zero duration could be set because of validation "duration>0" in the method for adding vesting. 

**Recommendation**: 

Remove useless validation and if-statement. 

**Post-audit**: 
Contract was removed.

### Functions have the same implementation. 

**Description**

Disqualifier.sol: getBaseBounty() and getCompound Bounty(). 
Implementation of functions getBaseBounty and getCompoundBounty is absolutely identical. Thus only one function can be left. 

**Recommendation**: 

Consider removing or modifying one of the functions. 

**Post-audit**: 
Contract was removed.

### ETH value is compared to USD value. 

**Description**

Eligibility DataProvider.sol: isEligible ForRewards(), line 205. 
The function validates that the user's locked value >= user's required value (with functions lockedUsdValue() and required UsdValue() respectively). 
The name of the function locked UsdValue() implies, that it will return value in USD, the natspec section (line 152) states that value is returned in ETH. In fact, the value is returned in USD. Thus the variable 'lockedValue is in USD units. 
The name of the function required UsdValue() implies, that it will return value in USD, the natspec section (lines 167-168) states that value is returned in ETH. In fact, the value is returned in ETH. Thus the variable 'requiredValue' is in ETH units. 
Thus when 'lockedValue' and 'requiredValue` are compared to each other, ETH value is compared to USD value, which is not a correct comparison. That leads to incorrect determining if user is eligible for rewards. Issue is marked as info, as it should be verified by the Radiant team if such functionality is valid. 

**Recommendation**: 

Correct the natspec or names of functions locked UsdValue() and required UsdValue(). It should be clear in which units the value is returned. Verify the correctness of comparison or bring values to the same units. 

**Post-audit**: 

In function required UsdValue(), variable 'totalCollateralETH', taken from LendingPool, was renamed into 'totalCollateralUSD. However no changes were applied in LendingPool, thus LendingPool still returns value in ETH. 

**Post-audit**: 

According to the Radiant Capital team, when AaveOracle is deployed, its base currency will be set as USD. Thus functions such as getUserAccountData() in Lending Pool will return values in USD, not ETH.

### Verification of oracles' implementation. 

**Description**

UniV3TwapOracle.sol: getPrecisePrice(), lines 73, 79. 
In case, value of 'decimals1' or 'decimals' is greater than 18, an overflow will occur. 
Though such a case where a token has more than 18 decimals is rare, such functionality should still be verified. 
1. UniV2TwapOracle.sol: consult(), line 118. 
Price is requested for a value '1 ether', thus in case token has any other number of decimals but 18, consult() will return an invalid price. 
2. Base Oracle.sol 
It should be noted that the owner of the contract can set any fallbackOracle at any time, thus affecting the price returned by oracle. Though it is not a security issue, it still should be noted in the final report. 

**Recommendation**: 

Correct the natspec or the name of functions locked UsdValue() and required UsdValue() so that it is clear in which units the value is returned. Verify the correctness of comparison or bring values to the same units. 

**Post-audit**: 
In function required UsdValue(), variable 'totalCollateralETH', taken from LendingPool, was renamed into 'totalCollateralUSD'. However no changes were applied in LendingPool, thus LendingPool still returns value in ETH. 
**Post-audit**: 
According to the Radiant Capital team, when AaveOracle is deployed, its base currency will be set as USD. Thus functions such as getUserAccountData() in LendingPool will return values in USD, not ETH.

### Event is never used. 

**Description**

MiddleFee Distribution.sol: MintersUpdated() 
Mint method was removed from the contract but the corresponding event remains present. 

**Recommendation**:

Remove unused event. 

**Post-audit**: 
Radiant team has removed unused event.

### Duplicating Event. 

**Description**

RadiantOFT: mint() 
Transfer event is duplicated since super._mint() uses same event. 

**Recommendation**: 
Remove duplicated event. 

**Post-audit**: 
Radiant team has removed duplicated event.

### Unnecessary variable. 

**Description**

RadiantOFT:_mint() 
Since contract can use_mint once per lifespan maxMintAmount is replacing maxSupply. 

**Recommendation**: 

maxMintAmount should be removed and replaced with maxSupply instead, also requires in this function cannot be reverted in the current contract version.

### Variable is initialized in storage in upgradable smart contract. 

**Description**

UniV3TwapOracle.sol: 'lookbackSecs`. 
Storage variables can't be initialized outside of functions in upgradable smart contracts. 
Thus variable 'lookbackSecs will be initially equal to 0. Issue is marked as info, since smart contract has additional setter for this variable. 

**Recommendation**: 

Initialize variable in initializer() function.
