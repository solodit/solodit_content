**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### ATTACKER CAN GET OWNERSHIP OF THE CONTRACT

**Severity**: Low	

**Status**: Resolved

**Description**

The ownership of the `TonBridge.sol` contract is set in the `initialize()` function by the caller. This function, `initialize()` has no access control so anybody can call the function and become the owner. The `initialize()` function can only be called once due to the `initializer` modifier so when the contract is deployed and attacker can front-run the deployer transaction calling `initialize()` and get the ownership of the contract and it has been mentioned then the deployer would not be able to call `initialize()` again.
```solidity
function initialize(address _tonBridge, address _symbiosisBridge, uint256 _bridgeChainId, address _broadcaster) public virtual initializer { 
       __Ownable_init(); 

       tonBridge = _tonBridge;
       symbiosisBridge = _symbiosisBridge;
       bridgeChainId = _bridgeChainId;
       broadcaster = _broadcaster;
   }
```


**Recommendation**:

So there are several mitigation actions that can be taken:
Verify after deployment, allow the call from one a certain address, include initialization call in your proxy setup, use a multicall contract to make the initialization call in the same transaction as the proxy setup.

### OWNERSHIP CAN BE TRANSFERRED TO `ADDRESS(0)`

**Severity**: Low	

**Status**: Resolved

**Description**

The `TonBridge.sol` contract is `OwnableUpgradeable` which contains a `renounceOwnership()` function that transfers ownership to address(0), which cause the same effect as having the contract without owner:

```solidity
/**
    * @dev Leaves the contract without owner. It will not be possible to call
    * `onlyOwner` functions anymore. Can only be called by the current owner.
    *
    * NOTE: Renouncing ownership will leave the contract without an owner,
    * thereby removing any functionality that is only available to the owner.
    */
   function renounceOwnership() public virtual onlyOwner {
       _transferOwnership(address(0));
   }
```
If ownership is renounced then some crucial functions that implements `onlyOwner` modifier will not be callable.

**Recommendation**:

### MISSING SANITY CHECKS FOR IMPORTANT VARIABLES

**Severity**: Low	

**Status**: Resolved

**Description**

There are some important variables in the `TonBridge.sol` contract that are not checked before being set in the `initialize()` function and also in the functions used for changing their values.

- Initialize():
```solidity
function initialize(address _tonBridge, address _symbiosisBridge, uint256 _bridgeChainId, address _broadcaster) public virtual initializer {       

       __Ownable_init(); 
       tonBridge = _tonBridge;
       symbiosisBridge = _symbiosisBridge;
       bridgeChainId = _bridgeChainId;
       broadcaster = _broadcaster;
}
```

- Functions used for changing values:
```solidity
function changeBridgeChainId(uint256 _newBridgeChainId) external onlyOwner { 
       bridgeChainId = _newBridgeChainId;
       emit ChangeBridgeChainId(_newBridgeChainId);
   }

   /**
    * @notice Changes TON Bridge by owner
    */
   function changeTonBridge(address _newTonBridge) external onlyOwner {
       tonBridge = _newTonBridge;
       emit ChangeTonBridge(_newTonBridge);
   }

   /**
    * @notice Changes Symbiosis Bridge by owner
    */
   function changeSymbiosisBridge(address _newSymbiosisBridge) external onlyOwner {
       symbiosisBridge = _newSymbiosisBridge;
       emit ChangeSymbiosisBridge(_newSymbiosisBridge);
   }

   /**
    * @notice Changes Broadcaster by owner
    */
   function changeBroadcaster(address _newBroadcaster) external onlyOwner {
       broadcaster = _newBroadcaster;
       emit ChangeBroadcaster(_newBroadcaster);
   }


Override `renounceOwnership()` function and revert the execution to do not allow ownership getting revoked:

function renounceOwnership() public override onlyOwner { 
revert ("NotAllowed”); 
}
```

**Recommendation**:

Add a check to ensure that the set values in the `initialize()` function and in the functions used for changing the variable values are not equal to `address(0)`.

### USE A MULTISIG ACCOUNT FOR OWNER

**Severity**: Low	

**Status**: Acknowledged

**Description**

The functions `changeBridgeChainId()`, `changeTonBridge()`, `changeSymbiosisBridge()` and `changeBroadcaster` are onlyOwner callable.This means the owner has the authority to change key variable values that affect core functionality. Therefore, it's advisable to use a multisig account to prevent accidental updates or to protect against the owner account being compromised.
```solidity
function changeBridgeChainId(uint256 _newBridgeChainId) external onlyOwner { 
       bridgeChainId = _newBridgeChainId;
       emit ChangeBridgeChainId(_newBridgeChainId);
   }

   /**
    * @notice Changes TON Bridge by owner
    */
   function changeTonBridge(address _newTonBridge) external onlyOwner {
       tonBridge = _newTonBridge;
       emit ChangeTonBridge(_newTonBridge);
   }

   /**
    * @notice Changes Symbiosis Bridge by owner
    */
   function changeSymbiosisBridge(address _newSymbiosisBridge) external onlyOwner {
       symbiosisBridge = _newSymbiosisBridge;
       emit ChangeSymbiosisBridge(_newSymbiosisBridge);
   }

   /**
    * @notice Changes Broadcaster by owner
    */
   function changeBroadcaster(address _newBroadcaster) external onlyOwner {
       broadcaster = _newBroadcaster;
       emit ChangeBroadcaster(_newBroadcaster);
   }
```


**Recommendation**:

Use a multisig account for owner’s account.

## Informational

### PUBLIC FUNCTIONS CONSUME MORE GAS THAN EXTERNAL ONES

**Severity**: Informational	

**Status**: Acknowledged

**Description**

Calling an external function is usually more gas-efficient than a public function when called from outside the contract because external functions can bypass the step of copying argument data from calldata to memory.

In `TonBridge.sol` contract the `callBridgeRequest()` function has been marked as public but can be switched to external as it is not called internally within the contract.



**Recommendation**:

Switch visibility from `public` to `external` in the case of the `callBridgeRequest()` function to improve gas saving.
