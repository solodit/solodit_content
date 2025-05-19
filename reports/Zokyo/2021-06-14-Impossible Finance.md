**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Error in fee calculation.

**Description**

ImpossibleLibrary.sol, Line 112
amountInPostFee = amountInPostFee.sub(sqrtK.sub(reserveIn));
- amountInPostFee is powered by 10000
- sqrtK.sub(reserveIn) - regular.
There is a calculation mistake with the loose of accuracy

**Recommendation**:

Fix the calculation mistake

## Medium Risk

### Different Solidity versions.

**Description**

Different pragma directives are used. Throughout the project (including interfaces). Version
used: '=0.5.16', '>=0.5.0'
Issue is classified as Medium, because it is included to the list of standard smart contracts’
vulnerabilities.

**Recommendation**:

Use the same pragma directives for the entire project.

### Solidity version update.

**Description**

The solidity version should be updated. Throughout the project (including interfaces).
Issue is classified as Medium, because it is included to the list of standard smart contracts’
vulnerabilities.

**Recommendation**:

You need to update the solidity version to the latest one - at least to 0.6.12, though 0.7.6 will
be the best option. This will help to get rid of bugs in the older versions.

### Hardcoded address

**Description**

StableXPair.sol, line 189
The governance address has unique permissions to change the setter and collect the fees.
Though there is no way to change that address and the address is hardcoded, so the contract
has no ability to change that permission in case of governance change or compromise.

**Recommendation**:

Add the governance address to the arguments of the initialize function and/or add the ability
in the StableXFactory.sol contract to change this address and pass the required one.

## Low Risk

### Explicitly mark visibility of state

**Description**

StableXPair.sol The variables do not specify the visibility of the state. lines 34-42, 45, 46.

**Recommendation**:

Explicitly mark visibility of state.

### Functions should be declared as external

**Description**

ImpossiblePair.getFeeAndXybk() (ImpossiblePair.sol#67-70)
ImpossiblePair.updateGovernance(address) (ImpossiblePair.sol#140-143)
ImpossiblePair.makeXybk(uint8,uint8,uint32,uint32) (ImpossiblePair.sol#146-169)
ImpossiblePair.makeUni() (ImpossiblePair.sol#173-183)
ImpossiblePair.updateTradeFees(uint16) (ImpossiblePair.sol#185-190)
ImpossiblePair.updateDelay(uint256) (ImpossiblePair.sol#194-198)
ImpossiblePair.updateHardstops(uint8,uint8) (ImpossiblePair.sol#201-207)
ImpossiblePair.updateBoost(uint32,uint32) (ImpossiblePair.sol#210-221)
ImpossibleRouter01.quote(uint256,uint256,uint256) (ImpossibleRouter01.sol#344-350)
ImpossibleRouter02.quote(uint256,uint256,uint256) (ImpossibleRouter02.sol#470-476)
ImpossibleRouter01.getAmountOut(uint256,address,address)
(ImpossibleRouter01.sol#352-358)
ImpossibleRouter02.getAmountOut(uint256,address,address)
(ImpossibleRouter02.sol#478-484)
ImpossibleRouter01.getAmountIn(uint256,address,address)
(ImpossibleRouter01.sol#360-366)
ImpossibleRouter02.getAmountIn(uint256,address,address)
(ImpossibleRouter02.sol#486-492)
ImpossibleRouter01.getAmountsOut(uint256,address[]) (ImpossibleRouter01.sol#369-376)
ImpossibleRouter02.getAmountsOut(uint256,address[]) (ImpossibleRouter02.sol#494-502)
ImpossibleRouter01.getAmountsIn(uint256,address[]) (ImpossibleRouter01.sol#379-386)
ImpossibleRouter02.getAmountsIn(uint256,address[]) (ImpossibleRouter02.sol#504-512)

**Recommendation**:

Functions should be declared as external

### Extra variable

**Description**

StableXPair.sol line 177, 181. It is possible to optimize the code for lower gas consumption.
Immediately assign the blockTimestampLast value without using temporary variables.

**Recommendation**:
Set the blockTimestampLast value without using the blockTimestamp variable.

## Informational 

### Unused contract

**Description**

Migration.sol
Standard Migration contract does not have any role in the system and can be removed.

**Recommendation**:

Remove the unused standard contract.

### Set the commission percentage to a variable

**Description**

StableXPair.sol line 252, 253.
In order to increase the readability of the code and further development security it is
recommended to move the commision value (201) to the public variable or public constant.

**Recommendation**:

Move the value into a constant.
