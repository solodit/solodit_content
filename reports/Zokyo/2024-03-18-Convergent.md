**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### No rewards calculations

**Severity**: High

**Status**: Acknowledged

**Description**

The smart contract allows to stake NFTs but doesn't calculate any rewards. The given documentation shows the points calculation, which is not implemented in the given codebase.

**Recommendation**: Implement rewards calculation	

**Client comment**: It is handled off-chain and therefore is not in the scope of the smart contract.	

## Informational

### Not commented code

**Severity**: Informational

**Status**: Acknowledged

A smart contract doesn't have any comments in the code

**Recommendation**: 

Add comments explaining the most important parts of the logic in the code repo. However, it's worth noting that it's not a security issue.
