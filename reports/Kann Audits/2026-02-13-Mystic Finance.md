**Auditor**

[Kann Audits](https://kannaudits.com/audits/mystic-finance-0x-deployment-february-2026)

# Findings

## Low Risk

### [L-01] Incorrect Constructor Arguments Used in ZeroExOptimized Verification

**Severity**

Low

**Description**

In the deployment script, the `ZeroExOptimized` contract is deployed using the bootstrapper address obtained from `fullMigration.getBootstrapper()`.

```ts
const zeroExOptimized = await deployContract(ZeroExOptimizedFactory, [await
    fullMigration.getBootstrapper()]);
console.log("ZeroExOptimized deployed to:", zeroExOptimized.address);
```

However, during the verification step, the script incorrectly uses the `FullMigration` contract address as the constructor argument.

```ts
await verifyContract(hre, deploymentResults.contracts.ZeroExOptimized, [
    deploymentResults.contracts.FullMigration]);
```

This mismatch causes verification failures because the constructor arguments provided to the verification function do not match those used during deployment.

**Root cause**

The verification logic does not account for the actual constructor parameter used when deploying `ZeroExOptimized`. Instead of using the bootstrapper address returned by `fullMigration.getBootstrapper()`, it mistakenly uses `fullMigration.address`, resulting in incorrect verification data.

**Recommendation**

Pass bootstrapper as the constructor argument during verification.

```ts
 const bootstrapper = await fullMigration.getBootstrapper();
deploymentResults.bootstrapper = bootstrapper;
await verifyContract(hre, deploymentResults.contracts.ZeroExOptimized, [
    deploymentResults.bootstrapper])
```

**Team Response**

Fixed.
