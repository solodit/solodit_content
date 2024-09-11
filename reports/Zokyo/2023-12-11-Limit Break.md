**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Native coin sent can be lost

**Severity**: Medium

**Status**: Resolved

**Description**

In CPortModule.sol - Function `_executeOrderBuySide()` accepts native coin `msg.value` in a sale that requires ERC20 payment without reverting. Native coin in this scenario is transferred to the contract to be effectively lost. This takes place when `feeOnTop` is included in that sale (i.e. invoking `buyListingWithFeeOnTop()`) .
The bug is shown in the following code snippet: msgValueItemPrice is assigned to zero value. Whilst function `_validateBasicOrderDetails()` should revert in that transaction, it does not because `msgValueItemPrice` just been assigned to zero value despite that msg.value can be non-zero (i.e. unintentionally set by external caller).
```solidity
       uint256 msgValueItemPrice = 0;

        if (saleDetails.paymentMethod == address(0)) {
            if (feeOnTop.amount + saleDetails.itemPrice != msgValue) {
                revert cPort__IncorrectFundsToCoverFeeOnTop();
            }
            
            msgValueItemPrice = saleDetails.itemPrice;
        }

        _fulfillSingleOrderWithFeeOnTop(
            disablePartialFill,
            msg.sender,
            saleDetails.maker,
            IERC20(saleDetails.paymentMethod),
            _getOrderFulfillmentFunctionPointers(saleDetails.paymentMethod, saleDetails.protocol),
            saleDetails,
            _validateBasicOrderDetails(msgValueItemPrice, saleDetails),
            feeOnTop);
```
It is worth noting that issue also arises after calling `buyListingCosigned()` which does not require a feeOnTop since it internally invokes `_executeOrderBuySide()` with zero feeOnTop. Therefore fixing `_executeOrderBuySide()` should suffice to fix that scenario as well.
```solidity
   function buyListingCosigned(
        bytes32 domainSeparator, 
        Order memory saleDetails, 
        SignatureECDSA memory sellerSignature,
        Cosignature memory cosignature
    ) public payable {
        _executeOrderBuySide(
            domainSeparator, 
            true,
            msg.value,
            saleDetails, 
            sellerSignature,
            cosignature,
            FeeOnTop({recipient: address(0), amount: 0})
        );
    }
```
**Recommendation**

Function `_executeOrderBuySide()` should revert if `saleDetails.paymentMethod != address(0)` and `msg.value != 0`

**Fix**:  Unused native coin is now returned to the user in the form of wrapped native coin (i.e. WETH in Ethereum main net).













### Price is fixed for any ERC1155 amount

**Severity**: Medium

**Status**: Resolved

**Description**

CPortModule.sol - In the scenario of listing ERC1155 via protocol ERC1155_FILL_PARTIAL, the taker pays the same price for any requestedAmount they demand of a given tokenId.

This is because of a bug in the following code snippet:

```solidity
if (quantityToFill != saleDetails.amount) {
    saleDetails.amount = quantityToFill;
    saleDetails.itemPrice = saleDetails.itemPrice / saleDetails.amount * quantityToFill;
}
```
Occuring in functions: `_executeOrderBuySide()` & `_executeOrderSellSide()` and their overloads. As `saleDetails.amount` is assigned to quantityToFill before calculating the correct itemPrice that is to be proportional with the quantityToFill. According to this snippet the itemPrice is just fixed for any quantityToFill.

**Recommendation** 

saleDetails.amount should be assigned to quantityToFill after calculating the itemPrice that is to be paid by buyer. 








### Signature is replayable across domains

**Severity**: Medium

**Status**: Resolved

**Description**

In CPortModule.sol - The domain of the signature (i.e. Contract name, version, address, chain) is not asserted to be conforming with the deployment state of the contract. A common vulnerable scenario that results from this issue is taking place if the contract is deployed across chains. Signature can be taken from one chain to be replayed on another.
Recommendation - The domainSeparator (i.e. which can be fed arbitrarily by taker) needs to be asserted with the current domain of contract deployment (i.e. chain, address, ...etc).
**Fix**:  Client carried out necessary changes as follow: 
domainSeparator is added to the calldata in newly implemented modifier delegateCallReplacementDomainSeparator shown in the following 

This is implemented in PaymentProcessor.sol (i.e. Previously called CPort.sol).
Which is used in the functions that are involved in the trading as shown in the following code snippet:


### Transfer Ownership to an incorrect address

**Severity**: Medium

**Status**: Acknowledged

**Description**

The CPort contract contains the transferOwnership() function in contract allows the current admin
to transfer his privileges to another address. However, inside transferOwnership(), the newOwner is directly stored into the storage owner, after validating the newOwner is a non-zero address, and immediately overwrites the current owner. This can lead to cases where the admin has transferred ownership to an incorrect address and wants to revoke the transfer of ownership or in the cases where the current admin comes to know that the new admin has lost access to his account.

**Recommendation**: 

It is advised to make ownership transfer a two-step process such as by using the Ownable2Step contract by Openzeppelin. 

**Refer**- https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol 
and https://github.com/razzorsec/RazzorSec-Contracts/blob/main/AccessControl/SafeOwn.sol   

## Low Risk

### Applied fees can be bypassed

**Severity**: Low

**Status**: Acknowledged

**Description**

CPortModule.sol - As the maker signs a listing then post it on the market, a taker who know the nuances of the contracts should be able to take the signature from the market as it is. After that the taker executes the order directly from the smart contracts (i.e. without using market frontend or using their own implemented frontend) with zero fees if they desire so. The issue arises because feeOnTop is not included in the signature. If it is included the taker can not use the same signature and if they want to bypass the fee they have to be able to get in touch with the maker to request a new signature with zero fee for them.

**Recommendation** 

The information about fees feeOnTop is to be added to the signature.


### Accidental calls to Renounce Ownership

**Severity**: Low

**Status**: Acknowledged

**Description**

The Cport contract uses Ownable which uses the renounceOwnership() function which can be called accidentally by the admin leading to immediate renouncement of ownership to zero address, after which any onlyOwner functions will not be callable which can be risky.

**Recommendation**: 

It is advised that the Owner cannot call renounceOwnership() without first transferring ownership to a  different address. Additionally, if a multi-signature wallet is utilized, executing the renounceOwnership method for two or more users should be confirmed. Alternatively, the Renounce Ownership functionality can be disabled by overriding it.


### No sanity checks for `pushPaymentGasLimit`

**Severity**: Low

**Status**: Resolved

**Description**

In the CPort Module contract, the pushPaymentGasLimit can only be set once. There are no sanity checks for it. It can also be accidentally set to zero. If it is incorrectly set, it cannot be set again. 

**Recommendation**: 


It is advised to add appropriate sanity value checks such as a non-zero check and a max limit check or introduce a function to change the pushPaymentGasLimit in case it is set incorrectly.

### Usage of keccack256 hash function for hashing leaves

**Severity**: Low

**Status**: Acknowledged

**Description**

In CPortModule contract, the _executeOrderSellSide() and similar functions, keccak256 hash function has been used for hashing leaves. Openzeppelin’s MerkleProof.sol contract warns against using the keccak256 hash function for hashing leaves. This is because the concatenation of a sorted pair of internal nodes in the Merkle tree could be reinterpreted as a leaf value.


**Recommendation**:  

it is advised to use a hash function other than keccak256 for hashing leaves.

Refer- https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/MerkleProof.sol#L14C15-L16C59 

### `createPaymentMethodWhitelist()` can be spam called

**Severity**: Low

**Status**: Resolved

**Description**

The function createPaymentMethodWhitelist() has no access modifier. Consequently, it can be called spam called multiple times by any address. This can result in overflow of paymentMethodWhitelistId and can result in overwriting of   
appStorage().paymentMethodWhitelistOwners[paymentMethodWhitelistId] 
A successful overwriting of the mapping variable can result in a malicious actor doing unauthorized call to whitelistPaymentMethod() and unwhitelistPaymentMethod()

**Recommendation**: 

It is advised to add appropriate access modifiers to the createPaymentMethodWhitelist() according to Business logic.


**Recommendation**: 

It is advised to add appropriate access modifiers to the createPaymentMethodWhitelist() according to Business logic.

**Fix**: This issue can result in High severity as the paymentMethodWhitelistId is only uint32 and can result in overwritting of values easily.

**Comments**: The client created removed the unchecked block so that overwriting of values is not possible. Still it is advised to take caution when deploying the contracts on blockchains or layer 2s that have cheaper gas fees as it could make spam calling the function practical. This could then result in a Denial of Service if the all the storage space gets used for the variable.

## Informational

### Similar naming of variables

**Severity**: Informational

**Status**: Resolved

**Description**

In the ModuleBulkTrades contract, the variable declared on line: 46 is named as msgValue, which clashes with the native msg.value syntax used in Solidity for transaction's ETH value. This can be confusing while coding and auditing, and can introduced errors. 

**Recommendation**: 

It is not advised to use similar or repititive variable names. It is advised to use different variable names instead.

### Multiple if statements can be clubbed

**Severity**: Informational

**Status**: Resolved

**Description**

In ModuleBulkTrades.sol, this can be clubbed into a single if statement 
```solidity
            if (address(paymentCoin) == address(0)) {
                revert cPort__DispensingTokenWasUnsuccessful();
            }
            if (disablePartialFill) {
                revert cPort__DispensingTokenWasUnsuccessful();
            }
```
as follows
```solidity
            if ((address(paymentCoin) == address(0)) || disablePartialFill) {
                revert cPort__DispensingTokenWasUnsuccessful();
            }
```
**Recommendation**: 

The above clubbing of if statements can be done in order to optimize the code.

### Insufficient verification and checks for FeeOnTop

**Severity**: Informational

**Status**: Resolved

**Description**


The FeeOnTop can be set arbitrarily to any value(lesser than itemPrice) by the caller. This can result in the caller/buyer bypassing the FeeOnTop entirely, or paying a value lesser than the expected FeeOnTop amount. For example, if the expected FeeOnTop amount is 50%, the buyer can just pay 20% or pay even no fees at all. 

**Recommendation**: 

It is advised to add appropriate checks to ensure that the FeeOnTop is paid as expected and not any value lesser than the expected fees. Or it is advised to clearly state in the docs or code comments that the FeeOnTop amount is a fee that is a maxFee(i.e. Any amount can be paid between 0 and FeeOnTop amount) and this fee is optional.

