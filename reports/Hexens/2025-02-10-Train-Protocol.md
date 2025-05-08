**Auditors**

[Hexens](https://hexens.io/)

---

# Findings
## High Risk

### [LYSWP2-7] The LP can steal the user's funds by initiating a refund before the user executes the redeem function

**Severity:** High

**Path:** chains/evm/solidity/contracts/HashedTimeLockERC20.sol#L172-L176 

**Description:** The timelock for all user and LP locks is enforced to be at least 15 minutes. However, LPs can exploit this by setting their lock duration to the minimum (15 minutes). This is because users must call `addLock()` after the LPs lock their tokens, ensuring that the user's timelock is always longer than the LP's timelock.

As a result, at the LP’s `timelock` expiration, the LP can call `refund()` on the destination chain to reclaim their tokens and then call redeem() on the source chain to claim the user’s funds. Meanwhile, the user is unable to retrieve their funds since their `timelock` has not yet expired.

Exploit Scenario:
1. Alice (the user) creates a commit object on the source chain, setting Bob (the LP) as `srcReceiver = Bob`.

2. At timestamp T, Bob sees the event on the source chain and locks tokens on the destination chain with a timelock of T + 900 (15 minutes).

3. Alice must call `addLock()` to finalize her commitment, which always happens after Bob’s lock is created. Consequently, Alice's timelock is always longer than Bob’s.

4. At T + 900, Bob calls `refund()` on the destination chain to reclaim his funds. At this point, Alice’s funds are still locked.

5. Bob then calls `redeem()` on the source chain to claim Alice’s funds.

Outcome: Alice loses her tokens, while Bob profits from the exploit.
```
  /// @dev Modifier to ensure the provided timelock is at least 15 minutes in the future.
  modifier _validTimelock(uint48 timelock) {
    if (block.timestamp + 900 > timelock) revert InvalidTimelock();
    _;
  }
```

**Remediation:**  The issue can be partially mitigated by forcing the LP lock their tokens at least 30 minutes (2 * 15 minutes). 

**Status:**  Fixed


- - -
## Medium Risk

### [LYSWP2-6] Discrepancy between the actual locked and stored amount of tokens when using fee-on-transfer tokens

**Severity:** Medium

**Path:** chains/evm/solidity/contracts/HashedTimeLockERC20.sol#L337-L350

**Description:** To bridge the funds the user needs to lock their funds on one chain to later unlock their funds on another chain. To do so the user can call the `lock()` function in the `HashedTimeLockERC20` contract:
```
if (token.balanceOf(msg.sender) < amount + reward) revert InsufficientBalance();
if (token.allowance(msg.sender, address(this)) < amount + reward) revert NoAllowance();

token.safeTransferFrom(msg.sender, address(this), amount + reward);
contracts[Id] = HTLC(
  amount,
  hashlock,
  uint256(1),
  tokenContract,
  timelock,
  uint8(1),
  payable(msg.sender),
  payable(srcReceiver)
);
```
However as the user given amount is being stored in the `contracts` mapping when using fee-on-transfer tokens the actual amount of tokens that will be transferred and locked in the contract will be less than the value stored in the mapping. This will lead to cases where when the user refunds or actually unlocks their tokens they will use the funds of other users.

If there is low amount of liquidity of that token it can lead to cases where the contract will be unable to refund or unlock the tokens because it will try to transfer the amount stored in the mapping while not having that amount of tokens actually on the contract.

**Remediation:**  Implement balance checks and store the balanceOf difference in the mapping instead of the user supplied amount.

**Status:**  Acknowledged

- - -

### [LYSWP2-8] Unchecked ERC20 transfer return value may cause HTLC inconsistency across chains

**Severity:** Medium

**Path:** chains/starknet/src/HashTimeLockedERC20.cairo#L391-L464, HashTimeLockedERC20.cairo#L339, HashTimeLockedERC20.cairo#L423

**Description:** The contract does not check the return value of `transferFrom()` and `transfer()` in multiple places, assuming that all ERC20 transfers will succeed. This is an issue because some ERC20 tokens do not revert on failure but instead return false. If the contract does not explicitly validate these return values, it may proceed with logic without actually transferring the funds, leading to severe inconsistencies. 

The `commit()` function executes an ERC20 `transferFrom()` call to move tokens from the user to the contract. However, the function does not check the return value of `transferFrom()`, assuming it will always succeed. In cases where the ERC20 token does not revert on failure but instead returns `false`, the function will continue execution without actually transferring funds.

This issue becomes particularly critical in a cross-chain scenario where an LP (Liquidity Provider) observes a commit event and locks funds on the destination chain (e.g., Ethereum). If the initial token transfer on the source chain (e.g., Starknet) failed silently, the LP would lock funds on the destination chain without receiving corresponding funds on the source chain.

This issue is not limited to the `commit()` function. Other functions, such as `lock()`, `redeem()`, and `refund()`, also fail to check ERC20 transfer return values, potentially leading to similar inconsistencies and unexpected behavior.
```
fn commit(
            ref self: ContractState,
            Id: u256,
            amount: u256,
            sender_key: felt252,
            dstChain: felt252,
            dstAsset: felt252,
            dstAddress: ByteArray,
            srcAsset: felt252,
            srcReceiver: ContractAddress,
            timelock: u64,
            tokenContract: ContractAddress,
        ) -> u256 {
            //Check that the ID is unique
            assert!(!self.hasHTLC(Id), "Commitment Already Exists");
            assert!(self.validTimelock(timelock), "Invalid TimeLock");
            assert!(amount != 0, "Funds Can Not Be Zero");

            // transfer the token from the user into the contract
            let token: IERC20Dispatcher = IERC20Dispatcher { contract_address: tokenContract };
            assert!(token.balance_of(get_caller_address()) >= amount, "Insufficient Balance");
            assert!(
                token.allowance(get_caller_address(), get_contract_address()) >= amount,
                "Not Enough Allowence"
            );
            token.transfer_from(get_caller_address(), get_contract_address(), amount);

            //Write the PreHTLC data into the storage
            self
                .contracts
                .write(
                    Id,
                    HTLC {
                        amount: amount,
                        hashlock: 0,
                        secret: 0,
                        tokenContract: tokenContract,
                        timelock: timelock,
                        claimed: 1,
                        sender: get_caller_address(),
                        sender_key: sender_key,
                        srcReceiver: srcReceiver,
                    }
                );

            let hop_chains = array!['null'].span();
            let hop_assets = array!['null'].span();
            let hop_addresses = array!['null'].span();

            self
                .emit(
                    TokenCommitted {
                        Id: Id,
                        hopChains: hop_chains,
                        hopAssets: hop_assets,
                        hopAddress: hop_addresses,
                        dstChain: dstChain,
                        dstAddress: dstAddress,
                        dstAsset: dstAsset,
                        sender: get_caller_address(),
                        srcReceiver: srcReceiver,
                        srcAsset: srcAsset,
                        amount: amount,
                        timelock: timelock,
                        tokenContract: tokenContract,
                    }
                );
            Id
        }
```

**Remediation:**  Explicitly check all ERC20 transfer return values (`transferFrom()` and `transfer()`). If the function returns `false`, the transaction should revert immediately.

**Status:**  Fixed


- - -
## Low Risk

### [LYSWP2-5] Immutable DOMAIN_SEPARATOR becomes invalid after a hard fork

**Severity:** Low

**Description:** The `DOMAIN_SEPARATOR` in the `HashedTimeLockERC20`, `HashedTimeLockEther` contracts are defined as a constant and remains unchanged after contract deployment. However, if a hard fork occurs, the `block.chainid` on one of the forked chains will change. This approach is potentially unsafe as such signatures could be deemed valid in forked networks, thus creating a vulnerability to replay attacks.
```  constructor() {
    DOMAIN_SEPARATOR = hashDomain(
      EIP712Domain({
        name: 'LayerswapV8',
        version: '1',
        chainId: block.chainid,
        verifyingContract: address(this),
        salt: 0x2e4ff7169d640efc0d28f2e302a56f1cf54aff7e127eededda94b3df0946f5c0
      })
    );
  }
```

**Remediation:**  Consider using the implementation from `OpenZeppelin`, which recalculates the domain separator if the current `block.chainid` is not the cached chain ID: 
[openzeppelin-contracts/contracts/utils/cryptography/EIP712.sol at 441dc141ac99622de7e535fa75dfc74af939019c · OpenZeppelin/openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/441dc141ac99622de7e535fa75dfc74af939019c/contracts/utils/cryptography/EIP712.sol#L22-L23)

**Status:**   Fixed


- - -

### [LYSWP2-13] Id should be checked by the LP before redeeming

**Severity:** Low

**Path:** chains/evm/solidity/contracts/HashedTimeLockERC20.sol#L253-L272 

**Description:** When a user calls the `addLock()` function, an event containing the HTLC ID, hashlock, and timelock is emitted. The Solver implementation documentation recommends verifying the correctness of the hashlock and timelock.

However, if the Solver does not verify which HTLC ID the hashlock and timelock correspond to and only checks their validity, an attacker could deceive the LP by associating the hashlock with a different HTLC—one that locks a lower amount than originally committed. This exploit would enable the attacker to receive significantly more tokens on the destination chain than they locked on the source chain.
```
function addLock(
    bytes32 Id,
    bytes32 hashlock,
    uint48 timelock
  ) external _exists(Id) _validTimelock(timelock) nonReentrant returns (bytes32) {
    HTLC storage htlc = contracts[Id];
    if (htlc.claimed == 2 || htlc.claimed == 3) revert AlreadyClaimed();
    if (msg.sender == htlc.sender) {
      if (htlc.hashlock == bytes32(bytes1(0x01))) {
        htlc.hashlock = hashlock;
        htlc.timelock = timelock;
      } else {
        revert HashlockAlreadySet(); // Prevent overwriting hashlock.
      }
      emit TokenLockAdded(Id, hashlock, timelock);
      return Id;
    } else {
      revert NoAllowance(); // Ensure only allowed accounts can add a lock.
    }
  }
```

**Remediation:**  Documentation should add the verification of the ID as an important step in the Solver implementation guide.

**Status:** Fixed

- - -