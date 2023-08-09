**Auditors**

[Trust Security](https://twitter.com/trust__90)


---


# Findings

## High Risk
### TRST-H-1 Linear vesting users may not receive vested amount
**Description:**
TokenTransmuter supports two types of transmutations, linear and instant. In linear, 
allocated amount is released across time until fully vested, while in instant the entire 
amount is released immediately. **transmuteLinear()** checks that there is enough output 
tokens left in the contract before accepting transfer of input tokens.
 ```solidity
        require(IERC20(outputTokenAddress).balanceOf(address(this)) >= 
            (totalAllocatedOutputToken - totalReleasedOutputToken), 
        "INSUFFICIENT_OUTPUT_TOKEN");
             IERC20(inputTokenAddress).transferFrom(msg.sender, address(0), 
        _inputTokenAmount);
 ```
However, `transmuteInstant()` lacks any remaining balance checks, and will operate as long 
as the function has enough output tokens to satisfy the request.

```solidity
        IERC20(inputTokenAddress).transferFrom(msg.sender, address(0), 
             _inputTokenAmount);
        SafeERC20.safeTransfer(IERC20(outputTokenAddress), msg.sender, 
             allocation);
        emit OutputTokenInstantReleased(msg.sender, allocation, 
             outputTokenAddress);
```
As a result, it is not ensured that tokens that have been reserved for linear distribution will 
be available when users request to claim them. An attacker may empty the output balance 
with a large instant transmute and steal future vested tokens of users.

**Recommended Mitigation:**
In transmuteInstant, add a check similar to the one in transmuteLinear. It will ensure 
allocations are kept faithfully.

**Team response:**
Issue was fixed.

**Mitigation review:**
The suggestion has been implemented. transmuteInstant checks that a sufficient balance is 
reserved for future linear vested tokens



## Medium Risk
### TRST-M-1 Vesting end time is not enforced
**Description:**
vestingEntryCloseTime is defined to be the time when vesting ends. However, there is a lack 
of check in **transmuteInstant()** and **transmuteLinear()**, that current time is lower than close 
time. Therefore, users may initiate new transmutations when desired period is over.

**Recommended Mitigation:**
Add a timestamp check in **transmuteInstant()** and **transmuteLinear()**

**Team Response:**
Issue was fixed.

**Mitigation review:**
Timestamp checks implemented successfully.



## Low Risk
### TRST-L-1 Zero duration time causes divide-by-zero exception

**Description:**
linearVestingDuration is used as the total period from start to end of vesting in linear 
transmutation. It is set in the constructor and is fixed. There is no validation in construction 
that the variable is not set to zero. When users call `releaseTransmutedLinear()` to claim 
released tokens, `_vestingSchedule()` is called which divides by linearVestingDuration in one 
flow. 
  ```solidity
     } else {
        return (totalAllocation * (timestamp - start(_vester))) / 
         duration();
        }
 ```
This flow has no chance of completing as the function will revert with divide-by-zero 
exception. 
**Recommended Mitigation:**
Require linearVestingDuration to not be zero in the constructor.

**Team Response:**
Specific issue as well as addition safety checks implemented in the constructor.

**Mitigation review:**
Correct safety checks are included in the constructor.



### TRST-L-2  vestedAmount and vestedAmountAtTimestamp return false information for instant transmutations
**Description:**
**vestedAmount()** and **vestedAmountAtTimestamp()** are view functions for user to see their
vested **amount currently and in a given timestamp respectively. The issue is that they 
assume user is on a linear vesting plan, rather than instant vesting. Therefore, they will 
display false information because instant vesting users will not receive any additional 
released amounts later.

**Recommended Mitigation:**
Add a requirement in both functions, that user is in the linear vesting plan, using the
addressToVestingCode mapping.

**Team Response:**
Issue was fixed.

**Mitigation review:**
Both functions now support linear and instant vesting. However, 
vestedAmountAtTimestamp may still return incorrect results for timestamp < time of vesting
for instant vesting. Function does not take into account timestamp of `transmute()` call.
```solidity
        if (addressToVestingCode[_vester] == 1) {
                 return addressToTotalAllocatedOutputToken[_vester];
                    } else if (addressToVestingCode[_vester] == 2) {
                        return 
                    _vestingSchedule(addressToTotalAllocatedOutputToken[_vester],
            uint64(_timestamp), _vester);
         }
```

### TRST-L-3 Multiplier implementation causes limited functionality
**Description:**
linearMultiplier and instantMultiplier are used to calculate output token amount from input 
token amount in transmute functions. 
```solidity
    uint256 allocation = (_inputTokenAmount * linearMultiplier) / 
        tokenDecimalDivider;
    …
    uint256 allocation = (_inputTokenAmount * instantMultiplier) / 
        tokenDecimalDivider;
```
The issue is that they are uint256 variables and can only multiply _inputTokenAmount, not 
divide it. It results in limited functionality of the protocol as vesting pairs where output 
tokens are valued more than input tokens cannot be used.

**Recommended Mitigation:**
Add a boolean state variable which will describe whether to multiply or divide by the 
multiplier.

**Team response:**
Acknowledged, but will not be fixed at this time as use case does not require division.


### TRST-L-4 tokenDecimalDivider implementation causes limited functionality
**Description:**
tokenDecimalDivider stores the decimals difference between input token and output token. 
However, this value assumes input decimals is always larger or equal to output decimals. As 
a result, the contract cannot be used for many input/output token pairs.
 ```solidity
         uint256 allocation = (_inputTokenAmount * linearMultiplier) / 
             tokenDecimalDivider;
```

**Mitgation review:**
Add a boolean state variable which will describe whether to divide or multiply by 
tokenDecimalDivider.

**Team response:**
Acknowledged, but will not be fixed at this time as use case does not require division.



### TRST-L-5 transmute functions may charge input tokens but not allocate any output tokens
**Description:** 
`transmuteInstant()` calculates and distributes allocation like so:
```solidity
        uint256 allocation = (_inputTokenAmount * instantMultiplier) / 
             tokenDecimalDivider;
        …
        IERC20(inputTokenAddress).transferFrom(msg.sender, address(0), 
                _inputTokenAmount);
        SafeERC20.safeTransfer(IERC20(outputTokenAddress), msg.sender, 
        allocation);
```
The issue is that allocation result could be zero due to division by tokenDecimalDivider
which trims many decimal points. If user does not provide a sufficient input amount, 
allocation will be zero but the function won't revert. Therefore, function will charge user the 
input amount but not give in return any output amount. The issue repeats in 
`transmuteLinear()`. It is not severe because if allocation is zero, input amount was probably 
quite small, but still important to address for user experience.

**Recommended mitigation:**
If allocation amount is calculated to be zero, revert in transmute functions.

**Team response:**
Issue was fixed

**Mitigation review:**
Successful fix

### TRST-L-6 emergencyPull introduces substantial risks
The **emergencyPull()** function is only callable by owner, and transfers the entire output 
token balance to a controlled destination. Use of **emergencyPull()** will make any linear 
vesting be cancelled without refund to the user. It is recommended that only the non allocated output tokens can be transferred out, as done in **outputTokenPull()**. Additionally, 
project should make sure the multisig address has a timelock in order to further protect 
users from compromised owner scenarios.

**Mitigation review:**
The fix adds a check that remaining balance is greater than the required balance, but 
actually transfers the entire output token balance. It should transfer out only the delta.
Therefore, the previous issue still exists.
```solidity
        uint256 outputTokenBalance = 
             IERC20(outputTokenAddress).balanceOf(address(this));
                uint256 vestingRequiredBalance = totalAllocatedOutputToken -
        totalReleasedOutputToken;
        require(outputTokenBalance > vestingRequiredBalance, 
            "NO_UNALLOCATED_TOKENS");
        SafeERC20.safeTransfer(IERC20(outputTokenAddress), 
             _emergencyOutputDestination, 
        IERC20(outputTokenAddress).balanceOf(address(this)));
```
### TRST-L-7 Owner can pause withdrawals of vested amount
Owner can use **setEmergencyPause()** to set the isPaused flag. The flag is checked in both 
transmute functions and **releaseTransmuteLinear()**. As a result, owner can immediately 
suspend withdrawals of vested amount. It is advisable that the pause flag would only be 
applied to transmute functions as it is severe for users to not be able to withdraw an already 
vested amount.

**Mitigation review:**
Centralization issue fixed


## Informational
### Immutable-only variables
The contract currently makes use of two immutable variables. However, there are several 
more variables which do not change throughout the lifetime of the contract. It is very 
advisable to make them immutable as it saves gas and protects against future errors.

### Safety checks
The constructor and setter functions lack some important safety checks. For example, it is 
important to make sure vestingEntryStartTime < vestingEntryEndTime, at all times. Also, the 
DAO multisig address should be verified to not be zero.

**Mitigation review:**
Additional safety checks properly implemented.

### NatSpec documentation
It is recommended to document code using NatSpec syntax which also supports 
autogenerated documentation resources