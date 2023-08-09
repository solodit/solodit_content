**Auditors**

[Trust Security](https://twitter.com/trust__90)


---


# Findings

## High Risk
### TRST-H-1 A malicious operator can drain the vault funds in one transaction
**Description:**
The vault operator can swap tokens using the `trade()` function. They pass the following 
structure for each trade:
```solidity
         struct tradeInput { 
             address spendToken;
               address receiveToken;
                 uint256 spendAmt;
                   uint256 receiveAmtMin;
                address routerAddress;
         uint256 pathIndex;
         }
```
Notably, **receiveAmtMin** is used to guarantee acceptable slippage. An operator can simply 
pass 0 to make sure the trade is executed. This allows an operator to steal all the funds in the 
vault by architecting a sandwich attack. 
1. Flashloan a large amount of funds
2. Skew the token proportions in a pool which can be used for trading, by almost 
completely depleting the target token.
3. Perform the trade at >99% slippage
4. Sell target tokens for source tokens on the manipulated pool, returning to the original 
ratio.
5. Pay off the flashloan, and keep the tokens traded at 99% slippage.
In fact, this attack can be done in one TX, different to most sandwich attacks.

**Recommended Mitigation:**
The contract should enforce sensible slippage parameters.

**Team response:**
"Added Chainlink Interface to allow for off-chain price knowledge, in a new contract, 
ChainlinkInterface.sol, which is deployed by the VaultManager at deploy time, and ownership 
is given to the protocol owner. This contract has an "addPriceFeed" function, on per token 
basis. All feeds are assumed to be in USD units. Then, the function "getMinReceived", performs 
the needed math to get the min expected back from a trade. This function is called by the 
VaultManager at trade time, which then checks against the caller's minReceived input. 
Additionally, a slippage for a given pair is now get and set by the AuxInfo.sol contract.

**Mitigation review:**
The integration with Chainlink oracle introduces new issues. There is no check for a stale price 
feed, which makes trading possibly incur high slippage costs. 
```solidity
               (,int priceFromInt,,,) = AIFrom.latestRoundData();
         (,int priceToInt,,,) = AITo.latestRoundData();
```
Additionally, when the contracts are deployed on L2, there is a sequencer down-time issue, 
as detailed here(https://docs.chain.link/data-feeds/l2-sequencer-feeds). The contract should check the sequencer is up when deployed on L2.

**Team Response:**
"Stale price feed check added ChainlinkInterface.sol, "getMinReceived" function, lines 90 - 94. 
Sequencer uptime check added to "getMinReceived" function, line 76."



### TRST-H-2 A malicious operator can steal all user deposits
**Description:**
In the Orbital architecture, each Vault user has a numerator which represents their share of 
the vault holdings. The denominator is by design the sum of all numerators of users, an 
invariant kept at deposits and withdrawals. For maximum precision, the denominator should 
be a very large value. Intuitively, numerators could be spread across different users without 
losing precision. The critical calculations occur in these lines in `deposit()`:
```solidity
         if (D == 0) { //initial deposit
                   uint256 sumDenoms = 0; 
                        for (uint256 i = 0; i < tkns.length; i++) {
                              sumDenoms += 
                        AI.getAllowedTokenInfo(tkns[i]).initialDenominator;
                      }
                   require(sumDenoms > 0 && sumDenoms <= maxInitialDenominator, 
         "invalid sumDenoms");
                   deltaN = sumDenoms; //initial numerator and denominator are the 
                same, and are greater than any possible balance in the vault.
             //this ensures precision in the vault's 
         balances. User Balance = (N*T)/D will have rounding errors always 1 
         wei or less. 
         } else { 
            // deltaN = (amt * D)/T;
            deltaN = Arithmetic.overflowResistantFraction(amt, D, T);
         }
```
In the initial deposit, Vault sums all token **initialDenominators** to get the final denominator. 
It is assumed that the vault will never have this amount in total balances (each token 
denominator is worth around $100m dollars). 

In any other deposit, the **deltaN** (numerator) credited to the depositor is (denominator * 
deposit amount / existing balance). When denominator is huge, this calculation is highly precise. However, when denominator is **1**, a serious issue oc**curs. If user's deposit amount is 
one wei smaller than existing balance, **deltaN** would be zero. This property has lead to the 
well-known ERC4626 inflation attack, where an attacker donates (sends directly to the 
contract) an amount so that the following deposit is consumed without any shares given to 
the user. In fact, it is possible to reduce the denominator to 1 and resurrect that attack. The 
root cause is that the initial deposit denominator is not linear to the deposit amount. Consider 
the attack flow below, done by a malicious operator:
1. Deploy an ETH/BTC pool
2. Flash loan $100mm in ETH and BTC each
3. Perform an initial deposit of $100mm in ETH/BTC
4. From another account, deposit 1 wei ETH / BTC -> receive 1 deltaN
5. Withdraw 100% as operator, reducing denominator to 1. 
6. Pay off flash loan
7. Wait for victim deposits
8. When a deposit arrives at the mempool, frontrun with a donation of an equivalent 
amount. The victim will not receive any shares ( numerator).
9. Any future deposits can be frontran again. Any deposit of less than the current 
balance will be lost.

**Recommended Mitigation:**
Consider checking that user's received **deltaN** is reasonable. Calculate the expected 
withdrawable value (**deltaN / denominator * balance**), and verify that is close enough to the 
deposited amount.

**Team Response:**
"switched the Vault Arithmetic to the scheme described in this article: 
https://docs.openzeppelin.com/contracts/4.x/erc4626. In this scheme, an imaginary 1 wei 
deposit is made when the vault is created, setting the Denominator to initD right away. This 
prevents the Denominator from ever dropping below initD. To compensate, you must now add 
1 to any balance when computing pool share from Numerator and Denominator. The VaultV2 
contract now contains "virtualBalance" functions that perform this task. In the VaultManager 
code, the Deposit and Withdraw functions have been updated accordingly. Also, as a result of 
the increased precision, the VaultFactory now finds the greatest init denominator for each of 
the tokens, instead of adding them all together."

**Mitigation review:**
The new ERC4626 logic is sound. The denominator cannot ever be below **initD**, therefore it is 
impossible to make a victim deposit lose precision, assuming it is not an abnormally massive 
deposit.

**Team Response:**
"Integration Testing revealed that the ERC204626 arithmetic loses precision when a low 
precision token is traded for a higher precision token (USDC to WETH, for example). The math 
is not as simple as it appears at first and doing a proper error analysis might reveal a way to 
correct the problem. However, we've opted to revert back to the more straightforward 
approach where the Denominator starts at 0 and is kept equal to the sum of Numerators. In 
order to prevent the original attack described by Trust, the intial denominator starts extremely high: 2**128 - 1, and, the Vault works with "Tracked Balances" instead of the real balances. 
The tracked balances are set to the true balances under these circumstances: Trade events, 
Depsoit events if the msg.sender is the Vault Operator, and Withdraw Events if the msg.sender 
is the Operator, or if the withdrawal is the final withdrawal. This method is designed to ward 
off Donation Attacks and Denial Of Service Attacks because Donations can have no affect on 
the Vault behavior."



## Medium Risk
### TRST-M-1 Removing a trade path in router will cause serious data corruption
**Description:**
The RouterInfo represents a single UniV3-compatible router which supports a list of token 
paths. It uses the following data structures:
```solidity
         mapping(address => mapping(address => listInfo)) private allowedPairsMap;
                  pair[] private allowedPairsList;
```

```solidity
          struct listInfo {
               bool allowed;
                uint256 listPosition;
         }
         struct pair {
            address token0;
               address token1;
                  uint256 numPathsAllowed;
          }
```

When an admin specifies a new path from **token0** to **token1**, `_increasePairPaths()` is called. 
```solidity
               function _increasePairPaths(address token0, address token1) private {
                     listInfo storage LI = allowedPairsMap[token0][token1];
                  if (!LI.allowed){
                  LI.allowed = true;
                  LI.listPosition = allowedPairsList.length;
                      allowedPairsList.push(pair(token0, token1, 0));
                   }
                      allowedPairsList[LI.listPosition].numPathsAllowed++;
                   }
```
When a path is removed, the complementary function is called.
```solidity
      function _decreasePairPaths(address token0, address token1) private {
             listInfo storage LI = allowedPairsMap[token0][token1];
                require(LI.allowed, "RouterInfo: pair not allowed");
                   allowedPairsList[LI.listPosition].numPathsAllowed--;
            if (allowedPairsList[LI.listPosition].numPathsAllowed == 0){
         allowedPairsList[LI.listPosition] = 
      allowedPairsList[allowedPairsList.length - 1];
         allowedPairsList.pop();
         LI.allowed = false;
      }
      }
```
When the last path is removed, the contract reuses the index of the removed pair, to store 
the last pair in the list. It then removes the last pair, having already copied it. The issue is that 
the corresponding **listInfo** structure is not updated, to keep track of index in the pairs list. 
Future usage of the last pair will use a wrong index, which at this moment, is over the array 
bounds. When a new pair will be created, it will share the index with the corrupted pair. This 
can cause a variety of serious issues. For example, it will not be possible to remove paths from 
the corrupted pair until a new pair is created, at which point the new pair will have a wrong 
**numPathsAllowed** as it is shared.


**Recommended Mitigation:**
Update the **listPosition** member of the last pair in the list, before repositioning it.


**Team Response:**
"Added 'lastPair' variable, which is used to point what was the last pair, to the new location in 
the list."

**Mitigation review:**
The fix is simple and correct.


### TRST-M-2 Attacker can DOS deposit transactions due to strict verifications
**Description:**
When users deposit funds to the Vault, it verifies that the proportion between the tokens 
inserted to the vault matches the current vault token balances.
```solidity
      uint256[] memory balances = vlt.balances();
          //ensure deposits are in the same ratios as the vault's current balances
          require(functions.ratiosMatch(balances, amts), "ratios don't match");
```

The essential part of the check is below:
```solidity
         for (uint256 i = 0; i < sourceRatios.length; i++) {
         // if (targetRatios[i] != (targetRatios[greatestIndex] * 
                  sourceRatios[i]) / greatest) {
               if (targetRatios[i] != 
         Arithmetic.overflowResistantFraction(targetRatios[greatestIndex], sourceRatios[i], greatest)) {
         return false;
            }
         }
```
The exact logic here is not important, but note that a small change in the balance of one of 
the vault tokens will affect the expected number of tokens that need to be inserted to 
maintain correct ratio. The exact amounts to be deposited are passed as **targetRatios**, and 
**sourceRatios** is the current balances. Therefore, an attacker can **directly transfer** a negligible 
amount of some vault token to the contract to make the amount the user specified in 
**targetRatios** not line up with the expected proportion. As a result, the deposit would revert. 
Essentially it is an abuse of the over-granular verification of ratios, leading to a DOS of any 
deposit in the mempool.

**Recommended Mitigation:**
Loosen the restriction on deposit ratios. A DOS attack should cost an amount that the vault 
creditors would be happy to live with.

**Team Response:**
"Added a new function to the VaultV2 contract, "takeBalanceSnapshot", which stores the state 
of the vault balances in a new variable. This function is called at the end of every "official" 
balance state change (Deposit, Withdraw, and Trade in the VaultManager). The ratios are 
checked against this new list instead of the actual balances."

**Mitigation review:**
The fix does not eliminate the described issue. An attacker can simply donate followed by 
deposit() of a negligible amount, in order to make takeBalancesSnapshot() get called. The new 
ratio will make the deposit getting front-ran revert.

**Team response:**
"See TRST-H-2 for solution."


### TRST-M-3 User deposits can fail despite using the correct method for calculation of deposit amounts
**Description:**
Users can use the `getAmtsNeededForDeposit()` function to get the amount of tokens that 
maintain the desired proportion for vault deposits. It will perform a calculation very similar to 
the one in `ratiosMatch()`, which will verify the deposit.
```solidity
         for (uint256 i = 0; i < balances.length; i++) {
               if (i == indexOfReferenceToken) {
                amtsNeeded[i] = amtIn;
         } else {
         // amtsNeeded[i] = (amtIn * balances[i]) / 
                  balances[indexOfReferenceToken];
                     amtsNeeded[i] = Arithmetic.overflowResistantFraction(amtIn, 
                  balances[i], balances[indexOfReferenceToken]);
               }
            }
```
However, a difference between the verification function and the getter function is that the 
getter receives any reference token, while the verification will use proportions based on the 
deposit amount in the largest balance in the vault. Indeed, these fractions may differ by a 
small amount. This could cause the `getAmtsNeededForDeposit()` function to respond with 
values which will not be accepted at deposit, since they will be rounded differently.

**Recommended Mitigation:**
Calculation amounts needed using the ratio between largest balance and the deposit amount. 
This would line up the numbers as verification would expect.

**Team Response:**
"Reworked the functions.getAmtsNeededForDeposit method so that ratios are based on the 
greatest amt instead of the reference token. The "amtIn" of the reference token is rounded 
down, if needed."

**Mitigation review:**
Fix is sound.


### TRST-M-4 Deposits of fee-on-transfer tokens will favor later depositors, making earlier investors lose funds
**Description:**
When deposits are processed, the percentage of **Denominator** minted to the depositor is 
linear to the contribution, compared to the current balance. 
```solidity
         uint256 T = vlt.virtualTotalBalance(); //will be at least 1
         uint256 D = vlt.D();
         if (functions.willOverflowWhenMultiplied(amt, D)) {
            require(T > amt || T > D, "overflow");
         }
         deltaN = Arithmetic.overflowResistantFraction(amt, D, T);
             vlt.setN(msg.sender, vlt.N(msg.sender) + deltaN);
                  vlt.setD(D + deltaN); //D always kept = sum of all Ns, plus 
                    vlt.initD()
         for (uint256 i = 0; i < tkns.length; i++) {
            if (amts[i] > 0) {
         IERC20(tkns[i]).safeTransferFrom(msg.sender, vaultAddress, amts[i]);
            }
         }
```
The calculation will lead to incorrect results when using fee-on-transfer (tax) tokens. The 
"before-tax" amount of the depositor will be compared to the "after-tax" amount in the 
contract balance. It is exploitable by immediately withdrawing the shares, receiving more 
tokens than the amount contributed (unless fees are higher than the token tax). 

**Recommended mitigation:**
Compare the balance before and after the `safeTransferFrom()` call.

**Team response:**
"amt now calculated by comparing vault balances before and after safeTransferFrom. N and 
D updated afterwards. "



## Low Risk
### TRST-L-1 Several popular ERC20 tokens are incompatible with the vault due to MAX approve
**Description:**
There are several instances where the vault approves use of funds to the manager or a trade 
router. It will set approval to MAX_UINT256. 
```solidity
         for (uint i = 0; i < tokens.length; i++) {
         //allow vault manager to withdraw tokens
                   IERC20(tokens[i]).safeIncreaseAllowance(ownerIn, 
         type(uint256).max); 
         }
```
The issue is that there are several popular tokens(https://github.com/d-xo/weird-erc20#revert-on-large-approvals--transfers) (UNI, COMP and others) which do not 
support allowances of above UINT_96. The contract will not be able to interoperate with 
them.

**Recommended Mitigation:**
Consider setting allowance to UINT_96. Whenever the allowance is consumed, perform re-approval up to UINT_96.

**Team Response:**
"Changed all allowance increases to type(uint96).max"

**Mitigation review:**
Fix is correct.


### TRST-L-2 Autotrader can steal funds
**Description:**
The slippage attack described in TRST-H-1 can also be performed by the autotrader contract.

**Team Response:**
"See TRST-H-1 response."

**Mitigation review:**
The H-1 fix solves the problem.


### TRST-L-3 Autotrader can steal gas
**Description:**
The autotrader receives gas compensation for the invocation of a trade action.
```solidity
      if (useGasStation && (msg.sender == _autoTrader) && (_autoTrader != 
          vlt.operator())) { //operator pays gas to _autoTrader for auto trades
             uint256 gasPrice = tx.gasprice;
               if (gasPrice == 0){
            gasPrice = 1;
         }
         uint256 fee = gasPrice * (gasStart - gasleft() + 
      gasStationParam);
      GS.removeGas(fee, payable(_autoTrader), vlt.operator());
      }
```
Note that **tx.gasprice** is controlled by the sender, by passing an arbitrary priority fee. This way, 
the fee calculation may grant them a large profit. The key point is that **gasStationParam** is set 
in a way that the gas spent to execute the transaction would be less than the gas 
compensation.

**Team response:**
"Capped gasPrice at block.basefee*2"

**Mitigation review:**
The solution reduced the risk to an accepted level.


### TRST-L-4 Owner can steal all funds
**Description:**
The owner can set a new operator using the function below:
```solidity
      function setOperator(address operatorIn) external nonReentrant {
         require(msg.sender == owner() || msg.sender == operator, "only ownop");
      operator = operatorIn;
      }
```
They can also allow an arbitrary, malicious router with the code below:
```solidity
         function allowRouter(address routerAddress, string calldata nameIn, uint256 routerType) external      onlyOwner 
             returns (address routerInfoContractAddress){
                require(!allowedRoutersMap[routerAddress].allowed, "router allowed");
                   require(routerType == 0 || routerType == 1, "must be 0 or 1");
                   allowedRoutersList.push(routerAddress);
            RouterInfo ri = new RouterInfo(owner(), nameIn, routerAddress, 
         routerType);
            allowedRoutersMap[routerAddress] = routerInfo(true, 
            allowedRoutersList.length 
         - 1, 
            address(ri));
         
            return address(ri);
         
         }
```
During trading, routers receive MAX approval.
```solidity
         //make sure router can spend vault's spend token
         //alternate idea: transfer tokens to this contract, trade, transfer back
            uint256 currentAllowance =  IERC20(params.spendToken).allowance(vaultAddress, 
                params.routerAddress);
             if (currentAllowance < params.spendAmt)
         vlt.increaseAllowance(params.spendToken, params.routerAddress, 
            type(uint256).max - currentAllowance);
```
The combination of these permissions allows an owner to drain all deployed vaults, by adding 
a malicious router, taking over as operator and calling the `trade()` function.

**Team response:**
"Removed owner permissions from change operator. Autotrade can now only trade when 
autotradeActive flag is set to true. Added a lockRouters() function to Aux contract. This will 
allow me to keep control while the project is getting started. Later, the routers can be locked 
to prevent any new ones from being added. Added checks at the end of the Trade function to 
verify transaction resulted in balance updates as intended."

**Mitigation review:**
The code changes described above reduce the centralization risks by a significant degree. 
Users are urged to validate that all routers are legitimate after the router list is locked down.


### TRST-L-5 When owner is also the operator, they can drain all funds in a vault
**Description:** 
The code changes implemented to fix slippage changes introduced a significant centralization 
risk. The owner can change price feeds at any time.
```solidity
         function addPriceFeed(address token, address priceFeed) external  onlyOwner {
         // require(aggregatorAddresses[token] == address(0), "price feed exists");
               aggregatorAddresses[token] = priceFeed;
                  emit PriceFeedAdded(token, priceFeed, true);
         }
         function removePriceFeed(address token) external onlyOwner {
             aggregatorAddresses[token] = address(0);
                emit PriceFeedAdded(token, address(0), false);
         }
```
This can be used to skew the minimum to receive value queried in `trade()`:
```solidity
      //check slippage with chainlink oracle
      uint256 maxSlippage = AI.getPairMaxSlippage(params.spendToken, params.receiveToken);
         require(CLI.getMinReceived(params.spendToken, params.receiveToken, 
            params.spendAmt, maxSlippage) <= params.receiveAmtMin, "rec min too low");
```
Therefore, owner can perform the sandwich attack detailed under "Autotrader can steal 
funds". He would need to double as the operator or the autotrader.


## Informational
### Redundant ownership transfers
There are several instances in the contract when ownership is transferred to the msg.sender. 
By inheriting from Ownable, this will happen automatically.
```solidity
         contract GasStation is Ownable, ReentrancyGuarded {
            mapping (address => uint256) private gasBalances;
                constructor() {
                   transferOwnership(msg.sender);
          }
```
**Team response:**
"Removed several ownership transfers."

**Mitigation review:**
Fixed.


### Redundant reentrancy protection
Several functions in the vault factory contract are protected with a reentrancy guard. It is not 
clear why this guard is required, so it would be best to re-consider uses of the reentrancy 
guard in several of the contracts. Specifically, consider when an attacker could gain code 
execution and what is the current state that could be abused.

**Team response:**
"Removed several reentrancy protections, including all from the VaultV2 contract (anything 
that might be a problem is called only by the VaultManager, which has the protection in 
place)."

**Mitigation review:**
Fix is sound


### Mark variables as constant
Some variables, like **_feeOwnerMax**, are fixed at compile-time. It would be best to mark them 
as constant, for code clarity as well as gas savings.

**Team response:**
"Changed several variables to constant or immutable."

**Mitigation review:**
Change applied correctly.


### Mark variables as immutable
Some variables, like **vaultInfoAddress** and **AI**, are fixed at deployment-time. It would be best 
to mark them as immutable, for code clarity as well as gas savings.

**Team response**
"Changed several variables to constant or immutable."

**Mitigation review:**
Change applied correctly.


### Add zero address checks
It is standard practice to check incoming addresses are not zero. Consider adding such a check 
in `setAutoTrader()`.

**Team response:**
"added zero address check to setAutoTrader."

**Mitigation review:**
Change applied correctly.


### Adding emission of events
Many of the vault functions do not emit events. It is recommended to do so for transparency 
and operation with indexers.
```solidity
         function deactivate() external nonReentrant {
              require(msg.sender == owner() || msg.sender == operator, "only own/op");
               isActive = false;
         }
         function setOperator(address operatorIn) external nonReentrant {
               require(msg.sender == owner() || msg.sender == operator, "only ownop");
         operator = operatorIn;
         }
         function setAllowOtherUsers(bool allow) external nonReentrant{
               require(msg.sender == operator, "only op");
                allowOtherUsers = allow;
         }
         function setStrategy(string calldata stratString) external nonReentrant {
            require((msg.sender == operator), "only op");
               strategy = stratString;
         }
         function setStrategyAndActivate(string calldata stratString, bool activate) external nonReentrant {
               require((msg.sender == operator), "only op");
                 strategy = stratString;
                   autotradeActive = activate;
                }
```

**Team response:**
"Added events to several VaultV2 actions. I also added a new event to VaultInfo contract to 
act as a "global" autotrade state change event, "Alert". This allows my autotrade backend to 
just check one thing to see if the autotrade state has been updated. The VaultV2 contracts 
trigger this event."

**Mitigation review:**
Events are now emitted at every important step.


### Validation checks
There is no check that all tokens in a newly created vault are unique. This could cause a wide 
variety of problems.

**Team response:**
"Added unique token validation to the VaultV2 constructor"

**Mitigation review:**
The new check is sound.


### Overflow detection
The utility overflowResistantFraction() is used in various places to multiply and divide when 
the intermediate result could be above uint256.It is important to note that if the final result 
exceeds 256 bits, the function would return an incorrect number, due to overflow. In most 
cases, that scenario is impossible as the divisor is larger than the second multiplied number. 
However, in the call below in the deposit() code path, it is somewhat possible:
```solidity
          deltaN = Arithmetic.overflowResistantFraction(amt, D, T);
```
It is recommended to detect a possible overflow here. 

**Team response:**
"Added overflow detection to deltaN calculation in the deposit function."

**Mitigation review:**
Overflow check is safe. It could be argued that the T > D check is unnecessary, as that should 
never occur.
```solidity
      if (functions.willOverflowWhenMultiplied(amt, D)) {
             require(T > amt || T > D, "overflow");
      }
```



