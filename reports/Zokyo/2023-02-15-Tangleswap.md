**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Medium Risk

### Users cannot mint positions due to incorrect variable value

**Severity**: Critical

**Status**: Resolved

**Description**

The contract NonfungiblePositionManager, allows users to create pools of token pair and fees if it does not exist from the parent contract PoolInitializer using the function createAndInitializePoolIfNecessary(...). This function call is then passed to the TangleSwapFactory contract, which deploys a new pool with a deterministic address(see this code).


NonfungiblePositionManager contract also allows users to create positions in the pool and mint nfts (see mint function). This function call is pass on to the addLiquidity(...) function, which is inherited from the LiquidityManagement parent contract. Link

At line 6 in PoolAddress library, the variable POOL_INIT_CODE_HASH is set at compile time, which is the incorrect hash value of the byte code of the TangleswapPool contract. Due to the incorrect value, the mint function cannot compute the correct value of the deployed pool, and the transaction reverts because the pool does not exist at the calculated address. So, no user can create a mint NFT position using the periphery contract.




**Recommendation**: 

Update the value of POOL_INIT_CODE_HASH so that NonfungiblePositionManager can mint the pool positions correctly.

**Existing value**:

 Suggested value as per the Trangleswap git commit:


## Low Risk

### Improper setup can lead to unpredictable behavior.

**Severity**: Low

**Status**: Resolved

**Description**

In SwapRotuer, the swap fails if nonFungiblePositionManager is not set, the swap transactions revert if the nonfungiblePositionManager is TangleswapFactory not initialized by the owner of the contract. All swap transactions through  SwapRotuer depend on TangleswapPool contract. 


**Recommendation**: 

Revert the contract calls with exact error message whenever  nonfungiblePositionManager is expected to be initialized but is actually not set by the contract owner.

 
