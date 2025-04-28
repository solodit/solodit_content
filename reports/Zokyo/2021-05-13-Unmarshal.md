**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Contract owner can transfer tokens allocated for investors

**Description**

Contract owner can invoke the smart contract PrivateDistributions’s method recoverToken
and pass argument _token as UnmarshalToken address to withdraw tokens allocated for
investors.

**Snippet**:
```solidity
function recoverToken(address _token, uint256 amount) external onlyOwner {
IERC20(_token).safeTransfer(_msgSender(), amount);
emit RecoverToken(_token, amount);
}
```

**Recommendation**:

Forbid to pass argument _token equal to UnmarshalToken address.

### Method releaseTokens has multiple issues

**Description**

Method releaseTokens of contract PrivateDistribution, that is invocable only by the contract
owner, has set of issues at the moment:
- it releases tokens for first 256 registered investors, because of uint8 overflow, if the
number of investors is greater than 256;
- it does not increase investor.withdrawnTokens amount, that lets investors withdraw
the same withdrawable amount later on;
- it does not generate event WithdrawnTokens;
- it does not track that amount of tokens should be more than zero.

**Snippet**:
```solidity
function releaseTokens() external onlyOwner initialized() {
for (uint8 i = 0; i < investors.length; i++) {
uint256 availableTokens = withdrawableTokens(investors[i]);
_oddzToken.safeTransfer(investors[i], availableTokens);
}
}
```
**Recommendation**:

It is expected that releaseTokens method should have similar behavior as for
`withdrawTokens`, also it should include changes;
- variable `i` should be of type `uint256`;
- method should accept two arguments `offset` & `limit` to release tokens for a portion of investors;
- skip transferring of tokens if withdrawable amount of tokens is 0;
- generate event WithdrawnTokens on successful release (transfer) of tokens.

**Resolution**:

Method is removed.

### Investors will not be able to withdraw tokens after the end of the vesting duration for Ecosystem in UnmarshalTokenVesting

**Description**

Array ecosystemVesting length is equal to 50. If more than 50 months have passed since the
initial time function _calculateAvailablePercentage will lead to error.

**Snippet**:
```solidity
if (currentTimeStamp > _initialTimestamp) {
if (distributionType == DistributionType.ECOSYSTEM) {
return ecosystemVesting[noOfMonths];
```

**Recommendation**:

Add check for noOfMonths > 50, if it evaluates to true, then returns last value of
ecosystemVesting.

### Investors will not be able to withdraw tokens after the end of the vesting duration for STAKING, MARKETING and RESERVES in UnmarshalTokenVesting

**Description**

Array marketingReserveVesting length is equal to 38. If more than 38 months have passed
since the initial time function _calculateAvailablePercentage will lead to error.

**Snippet**:
```solidity
if (currentTimeStamp > _initialTimestamp) {
if (distributionType == DistributionType.ECOSYSTEM) {
return ecosystemVesting[noOfMonths];
} else if (
distributionType == DistributionType.MARKETING ||
distributionType == DistributionType.RESERVES ||
distributionType == DistributionType.STAKING
) {
return marketingReserveVesting[noOfMonths];
```
**Recommendation**:

Add check for noOfMonths > 38 if true return last value of marketingReserveVesting.

### Incorrect withdraw logic for team distribution type

**Description**

Function _calculateAvailablePercentage for team distribution will always return 100% if current
timestamp is before of initial cliff (240 days).

**Snippet**:
```solidity
if (currentTimeStamp > initialCliff && currentTimeStamp < vestingDuration) {
uint256 noOfDays = BokkyPooBahsDateTimeLibrary.diffDays(initialCliff, currentTimeStamp);
if (noOfDays == 1) {
return uint256(15).mul(1e18);
} else if (noOfDays > 1) {
uint256 currentUnlockedPercentage = noOfDays.mul(everyDayReleasePercentage);
// console.log("Current Unlock %: %s", currentUnlockedPercentage);
return uint256(15).add(currentUnlockedPercentage);
}
} else {
return uint256(100).mul(1e18);
}
```

**Recommendation**:

Extend `else` block with the condition to check if the current timestamp is greater than initial
cliff.

### PrivateDistribution does not guarantee the allocation of tokens for investors

**Description**

Method _addInvestor does not allocate tokens for investors during their creation and requires
the governance address manager to transfer tokens manually to PrivateDistribution contract
address.

**Snippet**:
```solidity
function _addInvestor(
address _investor,
uint256 _tokensAllotment,
uint256 _allocationType
) internal onlyOwner {
require(_investor != address(0), "Invalid address");
require(_tokensAllotment > 0, "the investor allocation must be more than 0");
Investor storage investor = investorsInfo[_investor];
require(investor.tokensAllotment == 0, "investor already added");
investor.tokensAllotment = _tokensAllotment;
investor.exists = true;
investors.push(_investor);
investor.allocationType = AllocationType(_allocationType);
_totalAllocatedAmount = _totalAllocatedAmount.add(_tokensAllotment);
emit InvestorAdded(_investor, _msgSender(), _tokensAllotment);
}
```
**Recommendation**:

Invoke transferFrom to method _addInvestor on the amount specified in method argument
_tokensAllotment.

### Owner can set initial timestamp several times in UnmarshalTokenVesting

**Description**

Function setInitialTimestamp does not change the value of isInitialized to true.

**Snippet**:
```solidity
function setInitialTimestamp(uint256 _timestamp) external onlyOwner() notInitialized() {
// isInitialized = true;
_initialTimestamp = _timestamp;
```
**Recommendation**:

Remove comment.

## Medium Risk

### Require function can't throw exceptions in UnmarshalTokenVesting

**Description**

Function _addDistribution is internal. All distributions are declared when deploying the
contract including tokens allotment and beneficiary address, so require can’t throw exceptions
for _tokensAllotment > 0 and distribution.tokensAllotment == 0.

**Snippet**:
```solidity
require(_tokensAllotment > 0, "the investor allocation must be more than 0");
require(distribution.tokensAllotment == 0, "investor already added");
```
**Recommendation**:
Remove those exceptions.

## Low Risk

### Extra debug events are present in PrivateDistribution smart contract

**Description**

Typo in error message in PrivateDistribution smart contract.

**Snippet**:
```solidity
require(tokensAvailable > 0, "no tokens available for withdrawl");
```
**Recommendation**:

Make `withdrawl` to become `withdrawal`.

### Extra debug events are present in PrivateDistribution smart contract

**Description**

PrivateDistribution utilizes library hardhat/console.sol for debugging events.

**Snippet**:
```solidity
console.log("Withdrawable Tokens: %s", tokensAvailable);
console.log("Everyday Percentage: %s, Days: %s, Current Unlock %: %s",everyDayReleasePercentage,
noOfDays, currentUnlockedPercentage);
```
**Recommendation**:
Remove debug events.

## Informational

### Incorrect comment at Smart contract UnmarshalToken

**Description**

Comment states that `MAX_CAP` is equal to 125mln instead of 100mln.

**Snippet**:

uint256 public constant MAX_CAP = 100 * (10**6) * (10**18); // 125 million

**Recommendation**:

Remove comment or change it to point to the correct amount.

### Events and variables which are not used in UnmarshalTokenVesting

**Description**

Event DistributionRemoved and variable isFinalized declared but never used.

**Recommendation**:

Remove them if they are not needed.

### Variable that is not used in PrivateDistribution

**Description**

Variable isFinalized declared but never used.

**Recommendation**:

Remove if not needed.
