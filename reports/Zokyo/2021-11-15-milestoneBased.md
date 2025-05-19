**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### Function setTrustedSigner should do sanity checks even if the caller is the owner, the owner can make mistakes too.

**Recommendation**:

Do a sanity check where you are checking if the new trusted signer has a different value from
the old one, to not set the same value twice and consume gas.

### Function setUrl should do sanity checks even if the caller is the owner, the owner can make mistakes too.



**Recommendation**:

Do a sanity check where you are checking if the new uri has a different value from the old one,
to not be able to set the same value twice and consume gas.


## Informational

### There are some known bugs in the older versions of solidity that have been fixed in the most recent one, , it’s part of best practices to always use the most recent updated version.

https://docs.soliditylang.org/en/latest/bugs.html

**Recommendation**:

Set the solidity pragma to the most recent one (=0.8.9).

### Event Withdrawn from Roadmap contract should index the recipient address too, to be able to filter based on it.

**Recommendation**:
Make the recipient parameter indexed in the Withdrawn event.

### Error messages should be more straightforward than just an abbreviation, even if there is documentation to support them, openzeppelin way of handling them by adding the contract source followed by a straight and short explanation it’s a good practice.

**Recommendation**:

Make error messages more human-readable and straightforward, look into how openzeppelin
contracts handle the error messages.
