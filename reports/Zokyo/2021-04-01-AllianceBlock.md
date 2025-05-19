**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### AbstractPoolsFactory contract owner can transfer staked tokens to any address by calling method withdrawLPRewards and passing argument lpTokenContract that refers staking token contract.

Recommendation:
Forbid contract owner to transfer staked tokens to any address.

### ThrottledExit contract is affected by Out-of-gas error, because of usage of cycle to calculate nextAvailableExitBlock in case if current mined block is far in future comparing to initial nextAvailableExitBlock.

**Recommendation**:
Instead of doing iterations it is better to use mathematical formula to calculate
nextAvailableExitBlock as
```solidity
nextAvailableExitBlock = nextAvailableExitBlock + ((block.number - nextAvailableExitBlock) /
throttleRoundBlocks + 1) * throttleRoundBlocks
```

### Solidity compiler version is not up-to-date.

Recommendation:
Update Solidity compiler version.


### Contract AutoStake lets anyone to set initial pool address, that can be exploited by setting immediately pool address after deploying/instantiating new contract.

**Recommendation**:
Allow only contract owner to set initial pool address.

## Low Risk

### Smart contract is not fully covered by NatSpec annotations.

**Recommendation**:
Cover by NatSpec all Contract methods.

## Informational

### Error messages used in operation `require` are utilizing different formats: `Contract::method:: explanation`, `Intent:: explanation`, `Method::explanation`.

**Recommendation**:
Use same format for all error messages.
