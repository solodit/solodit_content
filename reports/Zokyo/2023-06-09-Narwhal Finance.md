**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### First depositor can break minting of shares

**Severity**: High

**Status**: Resolved

**Description**

The first depositor of an TradingVaultV2 can maliciously manipulate the share price by depositing the lowest possible amount of liquidity and then artificially inflating the currentBalanceUSDT value.
Attacker deposit (NarwhalTradingVault.deposit) 1 wei of USDT, minting 1 share. They receive 1e18 (1 wei) shares.
Attacker calls NarwhalTrading.openTrade, as a result of the calls chain the NarwhalTradingVault.currentBalanceUSDT value  increases to 100e18 + 1. 
Victim deposits 200e18 USDT. Due to a rounding issue in the inflated vault, they receive only 1 share.
Attacker withdraws their 1 share, which is now worth 150 USDT.

**Recommendation**:

Uniswap V2 solved this issue by sending the first 1000 LP tokens to the zero address. The same can be done in this case i.e. when totalSupply() == 0, mint the first min liquidity LP tokens to the zero address to enable share dilution.
Ensure the number of shares to be minted is non-zero: require(_shares != 0, "zero shares minted");

### Methods returning price in the wrong units

**Severity**: High

**Status**: Resolved

**Description**

In TradingVaultV2.sol - Method getPricePerFullShare() starts by assuming 18 decimal places for price (i.e. at totalSupply() = 0) . Once totalSupply() takes non-zero value the method returns the decimals of USDT (i.e. which is 6 on arbitrum). That causes a severe mismatch that shall in turn affect the tokenomics.
Similarly in method deposit(), we have :
```solidity
if (totalSupply() == 0) {
    shares = _amount;
} else {
    shares = (_amount.mul(totalSupply())).div(currentBalanceUSDT);
}
```
_amount and currentBalanceUSDT refer to USDT quantities, hence they are 6 decimals. While totalSupply() refers to the NarwhalTradingVault which is 18 decimals. Therefore we end up having shares possessing 6 decimals at totalSupply() == 0 and 18 decimals otherwise.

**Recommendation** 

Adjust the units returned to be consistent in both cases (totalSupply() =0 or totalSupply() > 0).

**Fix**:  As  of commit 32d6f65  , Issue still not fixed despite attempts to address it. 

The current result we get from this part of the codebase: 
```solidity
if (totalSupply() == 0) {
    shares =  1e6  // _amount refers to USDC decimals
} else {
    shares =  1e18    // 1e6 * 1e18 / 1e6 = TradingVaultV2 decimals
} 
```
Here _amount  inside the if-branch needs to be scaled properly to match the units of TradingVaultV2. 

Where as  getPricePerFullShare()   we face this result:
 ```solidity
totalSupply() == 0 ? 1e6 : 1e6 * 1e6 /  1e18;  
```
As balance() refers to currentBalanceUSDT  which is 6 decimal places. Inconsistency across branches need to be addressed.
**Fix**:  Issue still not addressed as of commit 3998b5, the following is a response to partner’s point:  
 totalSupply simply in the order of 1e18 because the NarwhalTradingVault  as an ERC20 , is implementing the decimals() default ERC20 method inherited by NarwhalTradingVault  contract :
 ```solidity
   function decimals() public view virtual override returns (uint8) {
        return 18;
    }
```
Therefore, decimals of NarwhalTradingVault is 18 which is reflected in totalSupply .
Here's an example of how it's being dealt with from another project:


**Fix**: As of commit  0ee6199 

Issue mentioned in deposit(uint,address) function is fixed by introducing a factor to scale up the deposited amount to suit the shares being minted.


Regarding  getPricePerFullShare() function 


There’s an issue in:
totalSupply() == 0 ? 1e18 : balance().mul(1e18).div(totalSupply());
It is suggested to have either units of 1e18 in both cases or 1e6 in both cases.
What is shown now is that it is in units of 1e18 in case totalSupply()==0 .  And it is in units of 1e6 otherwise.


**Recommendation**: 


totalSupply() == 0 ? 10**USDT.decimals(): balance().mul(1e18).div(totalSupply());

**Fix**:  issue resolved in commit 6d90f9f

## Medium Risk

### Centralized entities with excessive permissions

**Severity**: Medium

**Status**: Acknowledged

**Description**

The  setHandler function from Vester.sol and VesteNPL.sol contracts allows a governance to set a new handler address and activate or deactivate it. The entity with this privilege can e.g. claim users' reward for arbitrary address (claimForAccount), deposit his assets (depositForAccount) or influence the vesting reward calculations (settransferredCumulativeRewards). These abilities can significantly impact the protocol’s operation, potentially affecting its performance and outcomes.

**Recommendation**: 

Consider limiting the handler’s abilities.

Partially fixed: Protocol will add a timelock.

### Missing bound check on rewardsDuration function 

**Severity**: Medium

**Status**: Resolved

**Description**

In contracts NarwhalPool.sol and TradingVaultV2, at line 110 inside the setRewardDuration function allows the governance to to set the duration of rewards. This value is then used by the notifyRewardAmount function to calculate the reward rate (line 116). However, if the rewardsDuration parameter is set to zero, it will cause a divide-by-zero error when trying to calculate the reward rate, resulting in a contract malfunction. Additionally, if this value is very big, function may cease to produce meaningful results.

**Recommendation**: 

Add a minimum and maximum range check in the setRewardDuration function.

**Fixed**: 

Added range check.

### Insufficient check on reward token balance
 
**Severity**: Medium

**Status**: Resolved

**Description**

In contracts NarwhalPool.sol and TradingVaultV2, at line 127 inside the setRewardDuration function check if balance of esToken is enough to distribute the newly added rewards for their duration. Current check cannot be sufficient to make sure that the contract has enough amount of esToken/esNAR.
Let’s assume that rewardDuration is set for 7 days.
Governance mints/sends 100 esTokens to the contract and calls notifyRewardAmount function providing 100 tokens as an argument.
User stakes 1000 of whitelisted stake tokens.
Seven days later, the user calls the earned function which returns ~100 reward tokens, but the user chooses not to harvest the accrued reward.
Governance calls notifyRewardAmount function providing 100 tokens as an argument.
Seven days later the user calls the earned function which returns ~200 esTokens. When a user tries to harvest the reward, transition will fail due to insufficient balance of esToken.

```ts
    it("Insufficient check on reward token balance ", async () => { 
      await vault.setRewardToken(es.address, { from: deployer });
  
      const sevenDaysRewardsDuration = BN("604800"); //7 days in seconds
      vault.setRewardsDuration(sevenDaysRewardsDuration, { from: deployer });


      //Governance sends 100 esTokens 
      await es.transfer(vault.address, new BigNumber(100e18));


      //Governance calls notifyRewardAmount
      const reward = new BigNumber(100e18);
      await vault.notifyRewardAmount(reward, { from: deployer });


      //User stakes 100 usdt
      await vault.deposit(new BigNumber(100e18), user2, { from: user2, gas: 5000000, gasPrice: 500000000 }); 
      
      //7 days later
      await time.increase(time.duration.seconds(604800));
      let bal = await vault.earned(user2, { from: user2 })
      console.log("first 7 days: ", bal.toString())


      //Governance calls notifyRewardAmount
      await vault.notifyRewardAmount(reward, { from: deployer });


      //7 days later
      await time.increase(time.duration.seconds(604800));
      let bal2 = await vault.earned(user2, { from: user2 })
      console.log("second 7 days: ", bal2.toString())
      
      expectRevert(
        vault.harvest(user2, { from: user2 }), "BaseToken: transfer amount exceeds balance"
        );
    });
```
 
**Recommendation**: 

Consider adding a transferFrom function to the notifyRewardAmount function in order to transfer rewardTokens during each call.

**Fixed**: 

Issue fixed in commit a72e06b

### Front-running in referral link assignment leading to DoS of the functionality

**Severity**: Medium

**Status**: Resolved

**Description**

The NarwhalReferrals.signUp function allows a user to register on the platform with a referral. It takes in two parameters, _user which is the sender’s address and _referral which is the address of the user who referred them. Then the function checks whether the refLinkToUser mapping has a value assigned to the current block number. This assumption allows a malicious user to front-run legitimate calls to the signUp function by calling it within the same block before legitimate users can register, in consequence preventing them from registering with their intended referral. Since the protocol is planned to be deployed on the Arbitrum network, this kind of denial of service attack would be cheap enough to perform.

**Recommendation**: 

Consider using a nonce-based system that is not dependent on the block.number.

**Fixed**: Protocol implemented nonce-based system.


### Unconstrained for-loop

**Severity**: Medium

**Status**: Resolved

**Description**

In TradingStorage.sol - Method unregisterPendingMarketOrder() starts with a for loop that iterates for a number of times depending on the length of orderIds. The length of the array can go indefinitely since there is no size constraint is applied in method storePendingMarketOrder() which could check first the size of the array so that it does not pass a certain limit.

**Recommendation** 

-EnumerableSet can be used to replace uint[] as the type of orderIds. This helps to get the index i which has the value of _id in O(1) so that the for-loop is not needed in this case.
Another solution is to set a limit on the array contained in the mapping pendingOrderIds[trader] so that the for-loop does not grow indefinitely.
**Fix**: Dev team utilized EnumerableSet to avoid the unconstrained for-loop.


### Index returned overwrites an existing element

**Severity**: Medium

**Status**: Resolved

**Description**

In TradingStorage.sol - Methods: firstEmptyTradeIndex, firstEmptyOpenLimitIndex() returns index = 0 if all the array slots are occupied. This in turn overrides the first element (i.e. 0).

**Recommendation** 

Revert if all occupied (i.e. 3 by default) open limit slots are full.

**Fix** -   As of  commit a72e06b ,  A variable allOccupied is added in order to revert if all slots are full. Therefore issue is resolved.

### Lack of input validation in function

**Severity**: Medium

**Status**: Resolved

**Description**

In VestingSchedule.sol
Method addGroupCliff: _cliff is not validated to be non-zero value, having _cliff = 0 implies that it is not set to a value to begin with as shown in require(groupCliff[_group] == 0, ... ); which implies that it should be set only once for that given _group.
In that sense groupVestingDuration can be changeable for one given _group which leads to miscalculations as it gets involved in claimable amounts.
As for TradingVaultV2.sol - State _rewardsDuration is uint256, there is a chance a wrong large value is entered mistakenly. The consequences can lead to  DoS on setRewardsDuration() which in turn hinders the chance of handling the mistake by resetting the variable. That is because periodFinish would take a large value affected by the new value of _rewardsDuration.

**Recommendation** 

Add require statement to verify inputs are non-zero value.

Since _rewardsDuration is in order of years, it is recommended to have a maximum value around that value range. Then add a require statement to ensure that the _rewardsDuration is less than that maximum value.

**Fix**: As of  commit a72e06b , It is stated that _cliff can be zero for investors. The issue though is that it being stated in the revert message that groupCliff can not be changed if already set. But in this case groupCliff can be resettable if set to zero over and over which makes it severe because the vestingDuration can be altered in this scenario more than once. In order to counter that a new variable need to be introduced that shows if the groupCliff has been set already or not. 

The second part of the issue regarding TradingVaultV2.setRewardsDuration() is resolved by dev team.
**Fix**: As  of commit 32d6f65 , There is no change to address the issue in addGroupCliff().
**Fix**:  Fixed in 3998b5b,  by adding groupCliffSet to act as the check of whether the mapping groupCliff has been set for given _group or not. 


### Issue in generic withdrawal of Tokens

**Severity**: Medium

**Status**: Resolved

**Description**

Vester.sol/VesterNLP.sol - In withdrawToken, it withdraws any token including esToken. This can mess up the Vester balances as it is shown that amount of esToken transferred corresponds to a related amount of Vester minted back.

**Recommendation** 

exclude esToken from transfer.

**Fix**:  Require statement added that asserts token being transferred is not indeed the esToken.


### TradingStorage implementation contract is left uninitialized

**Severity**: Medium

**Status**: Resolved

**Description**

TradingStorage contract has been updated to use initialize() method which uses `initializer` modifier. It is recommended in OpenZeppelin’s documentation to not leave implementation contract uninitialised as attacker can take advantage of the same and that may affect the proxy contract. 

**Recommendation**: 

Use this suggested method in OpenZeppelin’s documentation to mitigate this issue.
**Fix-1** : Issue has been partially fixed in commit a72e06b. 
The method `disableInitializers()` is added which needs to be called to disable anyone to call the `initialize` method. 
When the proxy and implementation contract is deployed, initialization will be done in the context of proxy contract but not implementation contract.
Implementation contract will have  `storageT` as address(0) as it is not initialized.
Deployer can call `disableInitializers()` only if `gov` is set but it will revert as `storageT` is not set only.
Deployer will need to make a call to implementation contract’s `initialize(...)` method which can be front-run by Attacker to take control over the implementation contract.
Even if deployer is able to call the initialize(...) method, it will disable the initialize() method after the call. Deployer will not need to call `disableInitializers()` anymore to do so.
This is the exact reason why OpenZeppelin suggests to use the following:
```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
_disableInitializers();
}
```
Using this approach will disable the initializers for the implementation contracts directly without any other transaction. The above approach is suggested as well to avoid being dependent on a transaction to disable the initializer or being front-run by attackers.
**Fix-2**: Issue fixed in commit 3998b5b 


### Missing Zero-Address Validation for parameters

**Severity**: Medium

**Statue**: Resolved

**Description**

In NarwhatTradingCallbacks contract, there is no zero-address validation constructor parameters. There is also no method to modify these addresses if by mistake set to address(0). 

In TradingStorage contract, setTokens(address _USDT) set the USDT token address and _USDT is not validated for zero-address. 

In LimitOrdersStorage contract, constructor parameters are not validated for zero-addres.There is also no method to modify these addresses if bymistake set address(0). 

In NarwhalReferrals contract, constructor parameters are not validated for zero-address. There is also no method to modify these addresses if by mistake set address(0). 

**Recommendation**: 

Add zero-address validation

**Fixed**: Issue fixed in commit a72e06b 

### Stale Price Issue in NarwhalPriceAggregator 

**Severity** : Medium

**Status**: Resolved

**Description**

In the NarwhalPriceAggregator smart contract, the function tokenPriceUSDT() queries the latest price for the USDT token using the Pyth Oracle. The function does not perform a check to ensure that the returned price is not stale . As a result, it is possible for the function to return and use outdated pricing information


A price is considered stale when it has not been updated within a certain time threshold, which can be set based on the expected update frequency of the oracle. Stale prices can occur due to delays in updating the price feed or connectivity issues with the oracle.

**Recommendation** :

To address this issue, we recommend implementing a stale price check within the tokenPriceUSDT() function. compare the publishTime field in the priceFeed.price struct with the current block timestamp and define a tolerance (e.g., a few minutes) to account for an acceptable delay in price updates. If the difference between the current timestamp and the publishTime is greater than the tolerance, you can consider the price stale and handle it accordingly (e.g., by reverting the transaction with an error message).

**Fixed**: Issue fixed in commit a72e06b

### Missing Implementation of isPausable State Variable Leads to Inability to Prevent New Trades

**Severity** : Medium

**Status**: Resolved

**Description** : 

In NarwhalTradingCallback contract a state variable boolean 
isPaused that is intended to prevent new trades from being opened. However, the code does not include any mechanism for setting the value of this variable, meaning that it will always be initialized to false . Therefore, the intended functionality of preventing new trades from being opened will not be implemented.



**Recommendation** : 

Add function for IsPausable to make action on state variable and use openzeppelin pausable standard 
**Fixed**: Issue fixed in commit a72e06b




### Contract have been updated to be upgradeable lacks proper method to disable implementation contracts’ initialize(...) method

**Severity**: Medium

**Status**: Resolved

**Description**

In the latest commit a72e06b, several contracts have been made upgradeable using OpenZeppelin Upgradeable library.
Following are the contracts which are made upgradeable:
NarhwalRefferals
NarwhalTrading
NarwhalTradingCallbacks
TradingStorage
As it is recommended in OpenZeppelin’s documentation to not leave implementation contract uninitialised as attacker can take advantage of the same and that may affect the proxy contract. 
Because of the same reason, all of the above mentioned contracts use `disableInitializer()` method to disable the initialize() method. 
This approach does not resolve the issue completely and still put the risk of attacker taking control over the implementation contract. Let us look what happens when contracts are deployed.
When the proxy and implementation contract is deployed, initialization will be done in the context of proxy contract but not implementation contract.
Implementation contract will have  `storageT` as address(0) as it is not initialized.
Deployer can call `disableInitializers()` only if `gov` is set but it will revert as `storageT` is not set only.
Deployer will need to make a call to implementation contract’s `initialize(...)` method which can be front-run by Attacker to take control over the implementation contract.
Even if deployer is able to call the initialize(...) method, it will disable the initialize() method after the call. Deployer will not need to call `disableInitializers()` anymore to do so.
This is the exact reason why OpenZeppelin suggests to use the following:
```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
_disableInitializers();
}
```
Using this approach will disable the initializers for the implementation contracts directly without any other transaction. The above approach is suggested as well to avoid being dependent on a transaction to disable the initializer or being front-run by attackers.
**Fixed**: Issue fixed in commit 3998b5b 

## Low Risk

### Front-running Vulnerability in signUp Function

**Severity**: Low

**Status**: Acknowledged

**Description**:

The signUp function in the given contract is vulnerable to front-running attacks. An attacker can monitor pending transactions in the mempool and observe a user attempting to sign up with a specific referral. The attacker can then send a transaction with a higher gas price to register with the same referral before the user's transaction gets processed. This may result in the attacker benefiting from the referral rewards that were intended for the user.

**Proof**:

User A initiates a transaction to call the signUp function with a specific referral address (Referrer B).
Attacker C observes the pending transaction in the mempool.
Attacker C sends a transaction calling the signUp function with the same referral address (Referrer B) and sets a higher gas price.
Due to the higher gas price, Attacker C's transaction gets processed before User A's transaction.
Attacker C is now registered under Referrer B, potentially affecting the rewards intended for User A.

**Recommendation** : 

Implement a commit-reveal scheme to prevent front-running attacks on the signUp function. This solution involves two steps: a commit phase where users submit a hashed version of their referral information, and a reveal phase where users later reveal their referral information. By concealing the referral information during the commit phase, the attacker cannot observe the pending transactions and exploit the front-running vulnerability.

**Fix** : The fix requires two steps for signup and the Recommendation increases the complexity of the contract while there is no serious impact on funds with this vulnerability acknowledged and no change by the dev team


### Instant governance transfer

**Severity**: Low

**Status**: Resolved

 **Description**
 
Contracts TradingStorage.sol, BaseToken.sol, Vester.sol, VesteNPL.sol use setGov function for transferring ownership. In case of a mistake in the provided address, the management of the particular contract will be irretrievably lost.

**Recommendation**: 

Modify the process of updating the governance to be a two-step process, similar to the one used in the NarwhalPool.sol contract. This will require the new owner to explicitly accept the ownership update.

**Fixed**: Protocol added a two-step of updating the governance.


### Events lacking

**Severity**: Low

**Status**: Resolved

**Description**

no events emitted in admin/gov setters:
Vester.sol/VesterNLP
setHandler()
setNarwhalPool() - setTradingVault()
setHasMaxVestableAmount()
setTransferredAverageStakedAmounts()
settransferredCumulativeRewards()

In TradingVaultV2.sol
setRewardsDuration()
setAllowed()
setRewardToken()
setVester()
setWithdrawTimelock()

In NarwhalReferrals.sol
setBaseRebatesAndDiscounts()
setTier3Tier2RebateBonus()
setTradingVault()
setWhitelistedAddress()

In NarwhalPriceAggregator.sol
setUSDTFeed(bytes32 _feed)
setOracle(address _oracle)
setAge(uint256 _age)
setNarwhalTrading(address _NarwhalTrading)

In NarwhalTrading.sol
setAllowedToInteract(address, bool)
Fixed: Issue fixed in commit a72e06b 


### Redundant if branching in deposit

**Severity**: Low

**Status**: Resolved

**Description**

In TradingVaultV2.sol - Method deposit(uint,address): Unnecessary if branching
```solidity
if (allowed[msg.sender]) {
    console.log("Allowed");
    IERC20(USDT).safeTransferFrom(msg.sender, address(this), _amount);
    harvest(user);
} else {
    console.log("Not allowed");
    IERC20(USDT).safeTransferFrom(user, address(this), _amount);
    harvest(msg.sender);
}
```
Since user already takes the value of msg.sender in the else branch and that is given by the if-statement  before it (i.e. which checks the same condition).

**Recommendation** 

Remove if statement and replace by
```solidity
require(storageT.USDT().transferFrom(msg.sender, address(this), _amount));
harvest(user);
```
**Fixed**: Issue fixed in commit a72e06b 


### Redundant updateReward invocation

**Severity**: Low

**Status**: Resolved

**Description**

In TradingVaultV2.sol - deposit(uint,address)/withdraw(uint) invokes updateReward(), also in the body of the method the harvest(address) invokes that method. This updates the storage twice and might lead to confusing results. It is better to follow best practices on how to implement that logic.

**Recommendation** 

Remove the invocation in deposit() since it is already invoked in harvest().
**Fixed**: Issue fixed in commit a72e06b 

### Nft order retrieved is unchecked

**Severity**:  Low

**Status**: Resolved

**Description**

In TradingCallbacks.sol - Method executeOpenOrderCallback, nft order n retrieved from an external call to TradingStorage is not validated.

**Recommendation** 

Require statement to ensure n.trader is not equal to zero or return the method without side effects if it is equal to zero.
 **Fixed**: Issue fixed in commit a72e06b 


### USDT token is settable more than once

**Severity**: Low

**Status**: Resolved

**Description**

In TradingStorage.sol - Method setTokens() can set the address of the USDT token more than once. In TradingVaultV2.sol, we have setUSDT() which does not accept resetting the USDT address. Therefore, it is consistent to have the USDT setters in the contracts apply the same scheme.

**Recommendation** 

Reference to USDT better be immutable and set its value on deployment. Or apply the same approach which is already applied in TradingVaultV2.sol in method setUSDT().

**Fix** -  As of  commit a72e06b ,  dev team fixed the issue by making it settable only once on invoking initialize().

### Lack of validation in constructor

**Severity**: Low

**Status**: Resolved

**Description**

VestingSchedule.sol - In constructor _cliffs and _vestingDurations are not validated to be the same length while it is necessary that they be so.

**Recommendation** 

require statement to be added in order to ensure that condition.

**Fix** -  As of  commit a72e06b ,  issue fixed by dev team by adding the needed require statement which asserts both lengths are the same.

### Trade index has been set twice while registering the trade

**Severity**: Low

**Status**: Resolved

**Description**

In NarwhatTradingCallbacks contract, registerTrade method called the trading storage contract to find the first empty trade index and sets it as `trade.index`.
In the same method, NarwhatTradingCallbacks makes a call to store the trade in trading storage contract which also sets the `trade.index` using the same method as above.

**Recommendation**: 

Instead of assigning the trade.index again in `storeTrade` method in TradingStorage, it will be better to check if the assigned index is correct or not.

**Fixed**: 

Issue fixed in commit a72e06b 

### Methods `firstEmptyTradeIndex(...)` and `firstEmptyOpenLimitIndex` will always return 0 if no empty index available

**Severity**: Low

**Status**: Resolved

**Description**

In TradingStorage contract, methods firstEmptyTradeIndex(...) and firstEmptyOpenLimitIndex(...) uses for loop for finding the next empty index. 
These methods will always return 0 if the loop does not break and no empty index is found. 


**Recommendation**: 

Always use these methods with the following check
```solidity
require(
           storageT.openTradesCount(t.trader, t.pairIndex) +
               storageT.pendingMarketOpenCount(t.trader, t.pairIndex) +
               storageT.openLimitOrdersCount(t.trader, t.pairIndex) <
               storageT.maxTradesPerPair(),
           "MAX_TRADES_PER_PAIR"
       );
```

This will ensure not check for empty index if one doesn’t exist anymore.
**Fixed**: Issue fixed in commit a72e06b 

### Method `referKOLUnder` does not validate if same address is used for tier2 and tier3 

**Severity**: Low

**Status**: Resolved

**Description**

In NarwhalReferrals contract, the method `referKOLUnder(address _tier3, address _tier2)` does not check if  `_tier3 == _tier2`. In that case, any address will be able to refer itself.
This method is called by owner only so severity is low but adding a check is advised.

**Recommendation**: 

Add the following require statement.
`require(_tier3 != _tier2, “no same addresses”)`
**Fixed**: Issue fixed in commit a72e06b 

### Use require for checking if  USDT.approve was successful

**Severity**: Low

**Status**: Resolved

**Description**

In NarhwhalReferrals contract, in the claimRewards method, if user opted for compound then the USDT is approved first before depositing to the vault.
USDT token contract returns true if approval is successful so it is advised to use require statement.

**Recommendation**: 

Update the USDT.approve(...) line with the following
`require(IERC20(USDT).approve(TradingVault, pendings), “approval failed”)`
**Fixed**: Issue fixed in commit a72e06b 

### No events are emitted 

**Severity**: Low

**Status**: Resolved

**Description**

In NarwhalReferrals contract, no state changing methods emit events. It is advised to emit events atleast for claimRewards(), changeReferralLink() and signUp() methods.

**Recommendation**: 

Emit events for the suggested methods.

**Fixed**: Issue fixed in commit a72e06b 

### Use nonReentract modifier for claimRewards method

**Severity**: Low

**Status**: Resolved

**Description**

In NarwhalReferrals contract, add nonReentract modifier to the claimRewards method, just as added for signUp() and changeReferralLink() methods, as it transfer tokens and makes external calls.

**Recommendation**: 

Use nonReentrant modifier.

**Fixed**: Issue fixed in commit a72e06b 

### Missing validation

**Severity** : Low

**Status**: Resolved

**Description**

The setTradingVault function in the NarwhalReferrals contract allows the owner to set the address of the trading vault. However, the function does not check whether the new TradingVault address is valid or not, so there is a risk that an invalid address may be set accidentally or intentionally. This can compromise the security of the system
**Impact** : 
If Attacker gets the access of Owner then attacker can setup Malicious Vault as TradingVault

**Recommendation** 

TradingVault address can be initialised in the constructor to ensure that it is set correctly from the start.

**Fixed**: 

Issue fixed in commit a72e06b

### Inconsistent Calculation of Rewards Period

**Severity**: Low

**Status**: Resolved

**Description**

setRewardsDuration and notifyRewardAmount functions in the NarwhalTradingVault & NarwhalPool contract. The problem is that the notifyRewardAmount function calculates the periodFinish variable and sets it based on the rewards duration, which can be modified by the setRewardsDuration function. This means that if the notifyRewardAmount function is called before the setRewardsDuration function, the calculated periodFinish value will not reflect the new rewards duration.
This can lead to an inconsistency in the rewards distribution and may result in an incorrect calculation of rewards earned by users. 

**Impact**:

Gov could potentially exploit this vulnerability by calling the notifyRewardAmount function with a large reward amount and then calling the setRewardsDuration function with a small duration. This would result in a higher reward rate than intended and could potentially drain the reward pool.

**Recommendation** 

 notifyRewardAmount function should not calculate the periodFinish variable and instead rely on the duration of the reward set by the setRewardsDuration function. Alternatively, the setRewardsDuration function could be updated to also update the periodFinish variable to reflect the duration of the new reward.
**Another fix **: 

if rewardsDuration is set to zero. In that case, the calculation of periodFinish will result in the same value as block.timestamp, which means that the rewards period will end immediately after starting. This could potentially cause confusion or unexpected behavior.
To avoid this issue, it would be better to add a check for rewardsDuration and throw an error or revert the transaction if it is set to zero. For example:

**Fixed**: Issue fixed in commit a72e06b by adding the range the issue is fixed 

The function has a requirement that the input _rewardsDuration must be within the range of 183 days to 730 days. If the value is outside of this range, the transaction will revert and the change will not be made.

### Setting rewardsDuration to 0 can cause unexpected behavior

**Severity**: Low

**Status**: Resolved


**Description**: 

The rewardsDuration variable in the NarwhalTradingVault & NarwhalPool contract determines the duration of the rewards period, which is used to calculate the reward rate and distribute rewards to users. However, if the owner sets rewardsDuration to 0, the calculation of periodFinish in the notifyRewardAmount function will result in the same value as block.timestamp, causing the rewards period to end immediately after starting. This can result in confusion and unexpected behavior, and may lead to a loss of funds if users are not aware of the issue and continue to stake their tokens.

**Impact**: 

A malicious actor could potentially exploit this vulnerability by setting rewardsDuration to 0 and then depositing a large amount of tokens to earn rewards for a short period of time. This could drain the reward pool and result in a loss of funds for users.

**Recommendation**: 

To mitigate this vulnerability, a check should be added to the setRewardsDuration function to ensure that rewardsDuration is not set to 0. Alternatively, the notifyRewardAmount function could be updated to set periodFinish to a minimum value if rewardsDuration is set to 0, to ensure that the rewards period is at least one block long. Additionally, users should be informed of the potential issue and advised to avoid staking their tokens if rewardsDuration is set to 0.

**Fixed**: Issue fixed in commit a72e06b by adding the range the issue is fixed 

### Missing Check for Price returned from tokenPriceUSDT

**Severity** : Low

**Status**: Resolved

**Descritption** :

The tokenPriceUSDT function should have a check in place to ensure that the price returned from the queryPriceFeed function is not zero, as this could lead to a vulnerability. Without this check, users could potentially exploit the system by using a feed with a zero price to manipulate the values returned by the function.

**Recommendation** : 

Add Check For Non-zero Price after the Querying price from PythOracle 
```solidity
require(priceFeed.price.price != 0, “PRICE_FEED_ERROR”); // add check for non-zero price
```
**Fixed**: Issue fixed in commit a72e06b


### Missing Validation Zero address validation

**Severity** : Low

**Status**: Resolved

**Description**:

The setManager function in the PairInfos contract allows the gov address to set a new manager address. However, this function does not include a check to ensure that the new manager address is not a zero address. This can lead to unexpected behavior or unintended consequences.


If the manager address is set to the zero address, it can cause issues with the functionality of the contract. For example, if the manager address is used to interact with external contracts or services, then the contract may fail to interact properly or lose access to critical functionality if the manager address is set to zero. Additionally, if the manager address is used to control access to the contract, then setting the address to zero could potentially allow unauthorized users to interact with the contract.

**Recommendation**:

To mitigate this vulnerability, a check should be added to the setManager function to ensure that the new manager address is not a zero address. This can be accomplished by adding an if statement that checks if the new address is zero, and if so, reverts the transaction.

**Fixed**: Issue fixed in commit a72e06b

### Missing Zero Check Validation in setMaxNegativePnlOnOpenP Function

**Severity**: Low

**Status**: Resolved

**Description**:

The setMaxNegativePnlOnOpenP function in the PairInfos contract sets the maximum negative profit and loss percentage that can be incurred on opening a trade. However, there is no check for whether the value passed as the input parameter is zero, which could lead to unexpected results.
If a manager passes a value of zero as the input parameter, it would effectively disable the maximum negative PnL check, which could result in trades being opened that incur significantly higher losses than expected. 

**Recommendation**:

To address this vulnerability, a zero check validation should be added to the setMaxNegativePnlOnOpenP function to prevent the input parameter from being set to zero. This would ensure that the maximum negative PnL check is always enforced, which would help to prevent unexpected losses.

**Fixed**: Issue fixed in commit a72e06b

### Missing Validations and events in Critical functionality of PriceAggregator Contract 

**Severity**  : Low

**Status**: Resolved

**Description**:


setUSDTFeed function: The function lacks input validation, which means that any value can be passed to it, potentially causing unexpected behavior or errors in the system. It would be best to validate the input to ensure that it is a valid feed address before setting it.
setOracle function: Similar to setUSDTFeed, this function also lacks input validation, allowing any address to be set as the oracle address, which could result in unexpected behavior or errors. Additionally, the function does not emit an event, making it difficult to track changes made to the oracle address.
setAge function: The function has input validation to prevent the age value from exceeding a certain limit. However, it does not emit an event, making it difficult to track changes made to the age value.
setNarwhalTrading function: The function lacks input validation, allowing any address to be set as the NarwhalTrading address, which could result in unexpected behaviour or errors. Additionally, the function does not emit an event, making it difficult to track changes made to the NarwhalTrading address.

**Recommendations** : 

For the setUSDTFeed function:
Add a check to ensure that the _feed parameter is not equal to zero.
Emit an event after setting the new feed.
For the setOracle function:
Add a check to ensure that the _oracle parameter is not equal to zero.
Emit an event after setting the new oracle.
For the setAge function:
Add a check to ensure that the _age parameter is not greater than 60.
Emit an event after setting the new age.
For the setNarwhalTrading function:
Add a check to ensure that the _NarwhalTrading parameter is not equal to zero.
Emit an event after setting the new NarwhalTrading address.

**Fixed**: Issue fixed in commit a72e06b

### BaseToken contract can withdrawToken() function allows role gov to be able to withdraw tokens. 

**Severity**  : Low

**Status**: Resolved

**Recommendations**:

The gov address should be a multisig wallet to avoid malicious gov from withdrawing tokens.

**Fixed**: Issue fixed in commit  3998b5 

### In contract BaseToken, gov address has too much power and controls most critical  functions such as setting handler, adding and removing admins, toggling private mode, and setting token info. 

**Severity**  : Low

**Status**: Unresolved

**Recommendation**:

It is recommended that the gov address be a multisig address to prevent any single person acting maliciously at the expense of other participants.  

**Comment**: it's Partially fixed as side note below description and acknowledged as status since the client answer that this mechanism was implemented by design  

## Informational

### Missing import of `Initializable` contract

**Severity**: Informational

**Status**: Resolved

**Description**

Contract NarwhalRefferal and Contract Narwhal Trading is using `initialize(...)` method with `initializer` modifier. 
This modifier is from `Initializable` contract which is not imported and inherited in both of these unlike in Trading Storage & NarwhalTradingCallbacks contract. It is recommended in OpenZeppelin documentation as well.

**Recommendation**: 

Import and inherit the `Initializable` contract.

**Fix1**: Initializable has been imported but not inherited for the Trading Storage & NarwhalTradingCallbacks contract. Import the initializable contracts same as in NarhwalTradingCallbacks.sol


**Fix-2**: Issue fixed in commit 5055e7 

### Function do not emit proper events

**Severity**: Informational

**Status**: Resolved

**Description**

The following state changing function do not emit proper event:
NarwhalPool.setRewardsDuration
NarwhalPool.notifyRewardAmount
VesterNLP.setGov
BaseToken.setGov
Vester.setGov
TradingVaultV2.setRewardToken
TradingVaultV2.setVester
XXX

**Recommendation**: 

Consider adding events to functions that change important state variables.

### Uncalled public functions

**Severity**: Informational

**Status**: Resolved

**Description**

Public functions that are never called from within the contract should be marked as external.
NarwhalPool.setTokenWhitelisted
NarwhalPool.setWithdrawTimelockEnabled
NarwhalPool.getUserTokenBalance
NarwhalPool.getReservedAmount
NarwhalPool.getUserNARInfo
NarwhalPool._lockNAR
NarwhalPool._unLockNAR
NarwhalPool.stake
NarwhalPool.unstake
Vester.setNarwhalPool
VesterNLP.setTradingVault
VestingSchedule.start
VestingSchedule.changeClaimerStatus
VestingSchedule.transferAnyERC20

**Recommendation**: 

Make those functions external to optimize gas costs.

**Fixed**: Issue fixed in commit a72e06b 

### Excessively using Console.log

**Severity**: Informational

**Status**: Resolved

**Description**

Contracts are still including console.log statements which is pointless on deployment.

**Recommendation** 

Remove console.log lines.

**Fixed**: Issue fixed in commit a72e06b 


### Long revert message

**Severity** Informational

**Status**: Resolved

**Description**

In TradingVaultV2.sol - Method: setRewardsDuration, long revert messages are not recommended.

**Fixed**: Issue fixed in commit a72e06b 


### No need for Safe Math costly computation

**Severity**: Informational

**Status**: Acknowledged

**Description**

Vester.sol/VesterNLP.sol & VestingSchedule.sol - Methods: _mintPair, _burnPair, _mint, _burn, some arithmetic operations are checked unnecessarily as the conditions for it to pass without overflow/underflow is already existing.
```solidity
// _burnPair
pairAmounts[_account] = pairAmounts[_account].sub(_amount,"Vester: burn amount exceeds balance");
pairSupply = pairSupply.sub(_amount);

// _mintPair
pairSupply = pairSupply.add(_amount);
pairAmounts[_account] = pairAmounts[_account].add(_amount);

// _burn
balances[_account] = balances[_account].sub(_amount,"Vester: burn amount exceeds balance");
totalSupply = totalSupply.sub(_amount);

// _mint
totalSupply = totalSupply.add(_amount);
balances[_account] = balances[_account].add(_amount);

Given that balances <= totalSupply and pairAmounts <= pairSupply, In _mint() there is no need to safely add same amount to balances if the summation already passed the totalSupply addition without overflow, same in _mintPair(). In _burn() there is no need to safely subtract same amount from totalSupply if the subtraction already passed the balances subtraction, same in _burnPair().
Same applies to VestingSchedule
// _mint
totalSupply = totalSupply.add(_amount);
balances[_account] = balances[_account].add(_amount);

// _burn
balances[_account] = balances[_account].sub(_amount, "Vester: burn amount exceeds balance");
totalSupply = totalSupply.sub(_amount);
```

**Recommendation** 

Use unchecked block on these arithmetic operations to save the compute cost.

**Fix** -  As of  commit a72e06b , informational note is acknowledged and no change by dev team. There is no significance in this note as it does not affect the logic nor introduce a vulnerability.


### Inefficient storage usage

**Severity**: Informational

**Status**: Acknowledged

**Description**

In TradingVaultV2.sol - method notifyRewardAmount(), we have lastUpdateTime can be set twice to 2 different values which consumes more unnecessary gas cost.

**Recommendation** 

Implement a new version of method updateReward to be called by notifyRewardAmount() without having lastUpdateTime mutated in its implementation.

**Fix** -  As of  commit a72e06b , the informational note is acknowledged and no change by dev team.

### Checks-effects-interactions

**Severity**: Informational

**Status**: Resolved

**Description**

Methods not following checks-effects-interactions pattern
```solidity
TradingVaultV2.harvest(address)
IERC20(esnar).safeTransfer(user, pendingTokens);
rewardsToken += pendingTokens;

TradingVaultV2.distributeRewardUSDT(uint,bool)
transferFrom precedes updating contract's state
if (_send) {
    IERC20(USDT).safeTransferFrom(msg.sender, address(this), _amount);
}
currentBalanceUSDT = currentBalanceUSDT.add(_amount);

TradingVaultV2.receiveUSDTFromTrader(address,uint,uint,bool)
Interaction with storageT is taking place before state update.
storageT.transferUSDT(address(storageT), address(this), _amount);
currentBalanceUSDT += _amount;
```

**Recommendation** 

transfer which represents external interactions should take place after the effects.

**Fixed**: Issue fixed in commit a72e06b 


### Unnecessarily default safe math computation

**Severity**: Informational

**Status**: Resolved

**Description**

In TradingVaultV2.sol - By default arithmetic operations in soldity v0.8 are safe which incurs more computation. It is a good practice to avoid that extra computation if we are assured that a given arithmetic operation would be valid without the extra safe validations.
In TradingVaultV2.claimUSDT()
```solidity
require(currentBalanceUSDT > amount, "BALANCE_TOO_LOW");
currentBalanceUSDT -= amount;
```
Since currentBalanceUSDT is already greater than amount then subtraction shall not underflow.

**Recommendation** 

wrap arithmetic operations within unchecked.

**Fixed**: Issue fixed in commit a72e06b 


### Require statements lacking error messages

**Severity**: Informational

**Status**: Resolved

**Description**

In TradingStorage.sol - require statements in this contract are missing the inclusion of error messages. It is advised to provide a string containing a description about the error to be passed back to the caller.

**Recommendation** 

Include concise and short error messages.

**Fixed**: Issue fixed in commit a72e06b 

### Empty body if branching

**Severity**: Informational

**Status**: Resolved

**Description**

In LimitOrdersStorage.sol - Method setOpenLimitOrderType() the if-statement in this method is having no purpose as it only executes console.log code which is not meant for deployment.

**Recommendation** 

Remove the console.log lines and the if-statement if they are not meant to be used.

**Fixed**: Issue fixed in commit a72e06b 

### No validation of input address of storageT

**Severity**: Informational

**Status**: Resolved

**Description**

In PairsStorage.sol - Method: changeStorageInterface(address) does not validate the input address to make sure it is non-zero address.

**Recommendation** 

Add a require statement to validate the input argument of the method.

**Fixed**: Issue fixed in commit a72e06b 

### Misleading revert message

**Severity**: Informational

**Status**: Resolved

**Description**

In PairsStorage.sol - Modifier feedOk() contains a misleading revert message in
```solidity
require(_feed.feedCalculation != FeedCalculation.COMBINE, "FEED_2_MISSING");
```
**Recommendation** 

Change the error message returned to describe what actually went on in this faulty transaction.

**Fix** -  As of  commit a72e06b ,  Issue is resolved by changing the revert message to be a better match for the context.

### Missing validation of input address of user on signup

**Severity**: Informational

**Status**: Resolved

**Description**

In NarwhalReferrals.sol - Method signUp() , we have _user is not assured to be non-zero address in case that the caller of this method is storageT.

**Recommendation** 

Add the require statement that mitigates that.

**Fixed**: Issue fixed in commit a72e06b 


### Missing pure in function definition

**Severity**: Informational

**Status**: Resolved

**Description**

In NarwhalPriceAggregator - tenPow() is recommended to be defined as a pure function.

**Recommendation** 

Add pure to function definition.

Fixed: Issue fixed in commit a72e06b 


### Same contract written twice

**Severity**: Informational


**Status**: Acknowledged

**Description**

Vester.sol/VesterNLP.sol - Vester and VesterNLP are basically the same contract. Code repetition on that big scale is not advisable.

**Recommendation** 

Vester/VesterNLP inherit the common logic (which is basically the whole contract) from a base contract. Or implement one vester contract with two different settings on deployment (i.e. one points at NarwhalPool, other points at TradingVault).

**Fix** -  As of  commit a72e06b , note is acknowledged and no change by dev team. There is no significance in this issue as it does not affect the logic nor introduce a vulnerability.

### Checks-effects-interactions not obeyed

**Severity**: Informational

**Status**: Resolved

**Description**

Vester.sol - In _claim() this snippet:
```solidity
IERC20(claimableToken).safeTransfer(_receiver, amount);
cumulativeRewardDeductions[_account] += amount;
```
shows we have effects coming after interactions.

**Recommendation** 

Move the effect (updating the mapping cumulativeRewardDeductions) before the interaction (external call to safeTransfer).

**Fixed**: Issue fixed in commit a72e06b 

### Unnecessary usage of variable

**Severity**: Informational

**Status**: Resolved

**Description**

VestingSchedule.sol - Variable bool started is unneeded since startTime (i.e. being set to block.timestamp) already serves the purpose to be a start flag since it is conveying the information that method start() has been invoked.

**Fixed**: Issue fixed in commit a72e06b 

### PausableInterface variables never used

**Severity**: Informational

**Status**: Resolved

**Description**

In TradingStorage contract, `trading` and `callbacks` variables uses PausableInterface but never uses the only method defined in it, i.e. isPaused().

**Recommendation**: 

Remove the unused variables or Update the interface.

**Fixed**: Issue fixed in commit a72e06b 

### PRECISION storage variable can be constant 

**Severity**: Informational

**Status**: Acknowledged

**Description**

In TradingStorage contract, PRECISION variable (line#13) has been updated to be not constant while making the contract upgradeable. 
Since there is no method to update the PRECISION value, PRECISION can be constant and it will be compatible with upgradeability as well. The same is mentioned in OpenZeppelin’s doc.

**Recommendation**: 

Update PRECISION variable to be constant. 

**Fixed**: Issue Acknowledged in commit a72e06b 

### Method removeTradingContract(...) needs to check if the trading contract is added or not

**Severity**: Informational

**Status**: Resolved

**Description**

In TradingStorage contract, `removeTradingContract(address _trading)` does not check if _trading contract is added or not before flagging it false. 

**Recommendation**: 

Update the method to add the following require statement.

`require(isTradingContract[_trading], “No contract to remove”)`

**Fixed**: Issue fixed in commit a72e06b 

### No need to initialise variable to their default values

**Severity**: Informational

**Status**: Resolved

**Description**

In TradingVaultV2, these variables periodFinish & rewardRate is initialized to 0 which is not necessary as storage variable are by default initialized to their default values.

**Recommendation**: 

update it to remove the initialization with the default values.

**Fixed**: Issue fixed in commit a72e06b 


### USDT variable can be removed from NarwhalReferrals contract

**Severity**: Informational

**Status**: Resolved

**Description**

In NarwhalReferrals contract, USDT variable has been declared and initialised in constructor which can be replaced with storageT.USDT() when needed to approve or transfer tokens.

**Recommendation**: 

Use storageT.USDT() instead of USDT variable.


**Fixed**: Issue fixed in commit a72e06b 

### No  need to use `storage` reference

**Severity**: Informational

**Status**: Resolved

**Description**

In NarwhalReferrals contract, in the method `referKOLUnder(address _tier3, address _tier2)`, the referralDetails accessed on line#139 and line#140 uses storage reference variables. As these values are not modified in the method, memory reference can be used as well.

**Recommendation**: 

Update storage reference to memory on line#139 and line#140.

**Fixed**: Issue fixed in commit a72e06b

### LimitOrders interface variable is named `nftRewards` in several contracts

**Severity**: Informational

**Status**: Resolved

**Description**


In NarwhatTrading contract and NarwhatTradingCallbacks contract, the LimitOrdersInterface variable is named `nftRewards` which has nothing to do with NFTs. 

**Recommendation**: 

Rename those variables to more meaningful name.

**Fixed**: Issue fixed in commit a72e06b 

### Remove commented code

**Severity**: Informational

**Status**: Resolved

**Description**

In NarwhatTradingCallbacks contract (line#132, line#435, line#501, line#819) , there are many commented code which can be removed to make the code more clean.

**Recommendation**: 

Remove the commented code.

**Fixed**: Issue fixed in commit a72e06b 

### Typos or Misleading comment

**Severity**: Informational

**Status**: Resolved

**Description**

These are the few typos or misleading comments in the contracts.
Trading Storage -> Line#12 -> Variables are not constants anymore.

**Recommendation**: 

Update the comments.\

**Fixed**: Issue fixed in commit a72e06b 

### Add Missing Events in Critical functions 

**Severity**: Informational

**Status**: Resolved

**Description**

The NarwhalReferrals contract is missing events for critical functions such as signUp, changeReferralLink, referKOLUnder, incrementRewards, and incrementTier2Tier3. This can potentially lead to a security vulnerability, as it makes it more difficult for external parties to track and audit the actions taken by the contract.
In particular, the signUp function is a critical function, as it determines the user's initial referral link and referral source. Without an event to track this, it would be difficult for external parties to verify the integrity of the referral system. Additionally, the changeReferralLink function allows a user to change their referral source, and the lack of an event for this function could make it difficult to track when and how referral links are being changed.

**Recommendation** : 


Add events to this critical functions

**Fixed**: Issue fixed in commit a72e06b


###  Missing initialization of Reentrancy Guard Upgradeable Contract

**Severity**: Informational

**Status**: Resolved

**Description**

In the contract Narwhal Referrals and contract Narwhal Trading, these contracts inherits ReentrancyGuardUpgradeable which has an init method `__ReentrancyGuard_init` that needs to be called in the `initialize()`  to set the _status to NOT_ENTERED state.

**Recommendation**: 

Call the `__ReentrancyGuard_init` method in the `initialize()` method.

**Fixed**: Issue fixed in commit 3998b5b 

### Missing initialization of PausableUpgradeable Contract

**Severity**: Informational

**Status**: Resolved

**Description**

In the contract Narwhal Trading Callbacks, the contracts inherits PausableUpgradeable
which has an init method __Pausable_init that needs to be called in the `initialize()`  to set the _paused to `false` state.

**Recommendation**: 

Call the __Pausable_init method in the `initialize()` method.

**Fixed**: Issue fixed in commit 3998b5b 

### Use .call(...) for ETH withdraw rather than .transfer(...)

**Severity**: Informational

**Status**: Resolved

**Description**

In commit e15dd1,  method withdrawETH(...) has been added in Contract NarwhalPriceAggregator.sol which uses .transfer() to send all the contacts ETH to the recipient address. 

**Recommendation**: 

It is good practice to avoid using transfer() for sending native coins because this method uses a hardcoded gas amount, which may not be sufficient in the future and can potentially lead to unexpected failures. By using .call.value(...)("") instead, developers have more control over the gas usage and can adapt to changing gas costs, ensuring the robustness of contracts.

**Fix**: Issue fixed in commit a7755

### Unsafe casting from uint256 to int256

**Severity**: Informational

**Status**: Acknowledged

**Description**

In contract TradingStorage.sol, the method getNetOI(...) is converting openInterestUSDT[_pairIndex][0] and openInterestUSDT[_pairIndex][1] of type uint256 to type int256 unsafely. We understand that it will not cause any issue unless openInterestUSDT[_pairIndex][X] is greater than type(int256).max which is highly unlikely but this conversion is considered anti-pattern.

**Recommendation**: 

Use openzeppelin SafeCast Library to cast.
