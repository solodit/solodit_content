**Auditors**

[Kalogerone](https://x.com/kalogerone), [Talfao](https://x.com/talfao1), and [0xMrjory](https://github.com/fvelazquez-X)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/006_CODESPECT_TEMPEST_BRIDGE_ORACLE.pdf)

# Findings

## Low Risk

### [L-01] GUARDIAN_ROLE is not transferrable

**Files:** [`BridgeOracle.sol`](https://github.com/Tempest-Finance/tempest_smart_contract/blob/f9da49e15ea8f8f66670cb269814c9dd9fde875c/src/utils/BridgeOracle.sol)

**Description:**

The new `GUARDIAN_ROLE` cannot be transferred to other addresses because an admin role for this role hasn’t been set, unlike the `GOVERNANCE_ROLE` which other governors have been allowed to `grantRole` and `revokeRole` from other addresses. As shown below, `_setRoleAdmin(...)` for the `GUARDIAN_ROLE` is missing.

```solidity
constructor(
  address _baseUsdOracle,
  address _quoteUsdOracle,
  address _sequencerUptimeOracle,
  uint64 _baseOracleTimeLimit,
  uint64 _quoteOracleTimeLimit,
  uint64 _sequencerDowntimeLimit,
  bool _isL2,
  string memory _name,
  address governor
) {
  // ...
  _setRoleAdmin(GOVERNANCE_ROLE, GOVERNANCE_ROLE);
  _grantRole(GOVERNANCE_ROLE, governor);
  _grantRole(GUARDIAN_ROLE, governor);
}
```

**Impact:** The `GUARDIAN_ROLE` which is granted to the address, which updates the `isSequencerDown` for L2 networks without a Chainlink Sequencer Uptime Feed, can’t be transferred and is stuck to the initial governor forever.

**Recommendation:** Like the `GOVERNANCE_ROLE`, also set an admin role for the `GUARDIAN_ROLE`.

**Status:** Fixed

**Update from the Tempest:** Fixed in [7af4beaa4d5b011334333e21283eca60f46f9e52](https://github.com/Tempest-Finance/tempest_smart_contract/pull/203/commits/7af4beaa4d5b011334333e21283eca60f46f9e52)

## Informational

### [I-01] Initially isSequencerDown is false which may incorrectly allow price feed calls

**Files:** [`BridgeOracle.sol`](https://github.com/Tempest-Finance/tempest_smart_contract/blob/f9da49e15ea8f8f66670cb269814c9dd9fde875c/src/utils/BridgeOracle.sol)

**Description:**

During contract construction, the `isSequencerDown` is not initialized, meaning it is initially set to `false`. Contract deployment may happen at a time that the sequencer is down at the L2 chain without a Chainlink Sequencer Uptime Feed. For a short time, until the off-chain keeper is set up to call `setIsSequencerDown(...)` with the correct value, all calls for the `latestAnswer()` and `numeraireLatestAnswer()` will incorrectly go through.

**Impact:** After contract deployment, the `BridgeOracle` contract will return prices for any `latestAnswer()` and `numeraireLatestAnswer()` calls even if the sequencer is actually down.

**Recommendation:** Set the `isSequencerDown` variable to equal to `true` in the constructor until the off-chain keeper is fully set up.

**Status:** Fixed

**Update from the Tempest:** Fixed in [da18fc4f0f642926e53890833c268c83ce1a829d](https://github.com/Tempest-Finance/tempest_smart_contract/pull/203/commits/da18fc4f0f642926e53890833c268c83ce1a829d)
