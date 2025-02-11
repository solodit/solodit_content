**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Informational

### Unnecessary Ownable inheritance

**Description**

Contract inherits Ownable contract, though there is no any function to use the abilities of the
Ownable contract.

**Recommendation**:
Remove Ownable inheritance
