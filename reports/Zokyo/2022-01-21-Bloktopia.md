**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Informational

### Update solidity version

**Description**

Contract utilizes 0.7.6 version, though it is recommended to use the version from the latest
release, which is 0.8.11

**Recommendation**:

update version to 0.8.11

### Omit SafeMath after Solidity version upgrade

**Description**


In order to save gas during transactions SafeMath usage can be omitted, because 0.8.11
Solidity has built in support of safe operations.

**Recommendation**:

after upgrading to 0.8.11 SafeMath may be omitted.
