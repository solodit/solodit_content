**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### The contracts that are implementing the DesctructibleInterface give the owner too much control, the owner can call the destroyAndSend function and transfer all the ether from the contracts to an arbitrary address this is a risk especially if the private key of the owner gets compromised, this kind of functionalities should be used with a multi-sig.

**Recommendation**:

Implement a multi-signature functionality to be able to call the functions from the Destructible
contract.

## Low Risk

### An malicious entity can forcefully send ether to the RewardPool contract by creating another contract, sending ether to it, and then calling the selfdestruct function with the parameter being the RewardPool address, and then call the fill function with value 0, this way making the _fill variable true, but the value of the other variables like _collectorPool will be 0, this holds no security risk at the moment from what I can observe because funds can be unlocked by adding more ether through the fill function or by calling self-destruct with the parameter being the owner address, but it’s clearly not intended behavior because from the design of the RewardPool contract we can observe it is built to not accept ether through the fallback or receive functions.

**Recommendation**:

To mitigate this exact scenario where _fill variable will be true and the other variables will have
the value 0, add a sanity check for the msg.value variable at the beginning of the function.

### In the RewardPool contract, the functions payReward and fill should be made external to optimize the gas cost, the external functions are cheaper to call than the public function, and there are no cases when you call these functions internally, so it will make more sense to be external.

**Recommendation**:
Change the accessibility of the functions payReward and fill from public to external to optimize
gas costs.

## Informational

### Some require functions in the project (for example RewardPool, lines 49, 65, 66, 68, 72,...), does not contain a message, to be fully compatible with best standards, all required messages should contain a message, to be able to tell where the error happened in case of revert.

**Recommendation**:
Be sure that all the required functions from the project contain a descriptive error message.

### For gas optimization and reusability the function _compareStrings from RewardPool, _concat, and _uintToString from StampToken should be moved in a library and imported from there.

**Recommendation**:

Create libraries to improve accessibility, modularity, and gas costs.

### Specifying a pragma version with the caret symbol (^) upfront which tells the compiler to use any version of solidity bigger than specified considered not a good practice. Since there could be major changes between versions that would make your code unstable. The latest known versions with bugs are . 0.8.3 and 0.8.4

**Recommendation**:
Set the latest version without the caret. (The latest version that is also known as bug-free is
0.8.9)
