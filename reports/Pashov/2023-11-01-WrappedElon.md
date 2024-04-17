**Auditor**

[Pashov](https://twitter.com/pashovkrum)

# Findings

## Medium Risk

### [M-01] Centralization attack vector is present in `setEnabledState`

**Severity**

**Impact:**
High, as an owner can block unwrapping of wrapped assets

**Likelihood:**
Low, as it requires a malicious or a compromised owner

**Description**

The `setEnabledState` method of `WrappedElon` allows the owner of the contract to disable (or enable) wrapping and unwrapping of tokens. The issue is that a malicious or a compromised owner can decide to act in a bad way towards users and block unwrapping of the tokens, essentially locking them out of their funds. If the ownership is burned then (or private keys are lost) it will be irreversible.

**Recommendations**

Potential mitigations here are to use governance or a multi-sig as the contract owner. Even better is to use a Timelock contract that allows users to be notified prior to enabling/disabling wrapping/unwrapping so that they can take action, although this removes the benefit of using the method as a risk mitigation for bridge attacks.

## Low Risk

### [L-01] Disabling unwrapping will block all bridges at the same time

The `wrapEnabled` and `unwrapEnabled` variables are added as a mitigation mechanism against flawed bridges (that have potentially infinite mint vulnerability). The problem is that if this wrapped token is used by or integrated with multiple bridges, setting `unwrapEnabled` to `false` will block all bridges at the same time, even if just one of them is faulty. Consider switching to a mechanism that can handle multiple bridges integrations in a fault-tolerant way.

### [L-02] Wrapped token name is the unwrapped token name

The `WrappedElon` contract serves as a wrapper for `$ELON` tokens. Here is how its constructor looks like:

```solidity
constructor() ERC20("Dogelon", "ELON") {}
```

The problem is that instead of naming the token with the same name, prepended with the "Wrapped" word, it is using the same name as the unwrapped version. This can lead to confusions, especially if a liquidity pool is created with the wrapped token for some reason. Change the constructor in the following way:

```diff
-constructor() ERC20("Dogelon", "ELON") {}
+constructor() ERC20("WrappedDogelon", "WELON") {}
```

### [L-03] Protocol is using a vulnerable library version

In `package.json` file in the repository we can see this:

```javascript
"@openzeppelin/contracts": "^4.7.3",
```

This version contains multiple vulnerabilities as you can see [here](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories). While the problems are not present in the current codebase, it is strongly advised to upgrade the version to v4.9.5 which has fixes for all of the vulnerabilities found so far after v4.7.3.