**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### MultiSig can transfer all tokens to it’s address with function kill() at CoiinVault.sol

**Description**

Function kill allows the multisig address to transfer all deposited tokens to its address.

**Recommendation**:

Remove token transfer from function or share additional information on why this function
needs to be included.

### Missed interface import at SafeERC20.sol

**Description**

Contract SafeERC20.sol has import import "../interfaces/IERC20Burnable.sol" which not
included in repo. In all other contracts where needed SafeErc20 you use it from openzeppelin
library.

**Recommendation**:

If needed add an interface to the contracts folder, so the contract can be compiled or remove
the contract if you don’t need to use it.

### Requires don’t give access to mint function at CoiinMinter.sol

**Description**


In constructor of CoiinMinter.sol you define variable lastMintAmount equal to 1000000 * 1e18,
but in function mint you have requires:
// _amount cannot be less than minMintMonthly
require(_amount >= coiin.minMintMonthly(), "amount cannot be less than min mint amount");
// _amount cannot exceed maxMintMonthly
require(_amount <= coiin.maxMintMonthly(), "amount cannot be more than max mint amount");
// _amount cannot be more than 25% of the lastMintAmount
require(
_amount > lastMintAmount.add(lastMintAmount.mul(25).div(100)),
"amount cannot be more than 25% of last mint amount"
);
// _amount cannot be less than 25% of the lastMintAmount
require(
_amount < lastMintAmount.sub(lastMintAmount.mul(25).div(100)),
"amount cannot be less than 25% of last mint amount"
);
So you can mint in the range 1000000 * 1e18 to 25000000 * 1e18 (according to the first two
requirements) but according to 4 requirement mint amount should be less than 75% of
lastMintAmount and this amount would be less than minMintMonthly from CoiinToken.sol.
Additionally, 3 and 4 requires have misleading comments and revert messages.

**Recommendation**:

Update requires, change comments and revert messages.

## Medium Risk

### Misleading functions names at RaiinmakerCampaignVault.sol

**Description**


Function setFeeAmount() takes as a parameter address and assigns it to the variable
feeAddress. Also in function missed check for zero address.
Function setFeeAddress takes as a parameter uint256 _fee and assigns it to the variable fee.

**Recommendation**:

Change functions names and add an additional checks:
function setFeeAddress(address _address) external onlyMultiSig {
require(_address != address(0), "Wrong fee address");
feeAddress = _address;
emit FeeAddressChanged(_address);
}
function setFeeAmount(uint256 _fee) external onlyMultiSig {
fee = _fee;
emit FeeAmountChanged(_fee);

## Low Risk

### Excessive validation in the claimVestedTokens () and removeTokenGrant () functions at CoiinVestingVault.sol

**Description**


require(_totalSupply.sub(amountVested) >= 0, "not enough tokens");
Underflow already checked in sub() function from SafeMath.

**Recommendation**:

Remove these require() functions.

### Extra comments in function kill() at RaiinmakerCampaignVault.sol

**Description**


function kill() public onlyMultiSig {
pause();
// uint256 amount = totalSupply();
// _totalSupply[token] = _totalSupply[token].sub(amount);
// _totalWithdrawn[token] = _totalWithdrawn[token].add(amount);
// mtrToken.safeTransfer(multiSig, amount);
// emit Withdrawn(multiSig, 0, amount);
}

**Recommendation**:

Remove extra comments.

### Additional checks are required for constructor of CoiinMinter.sol

**Description**


There is no verification for the zero address for the coin and mint addresses, and checking of
the valid timestamp.

**Recommendation**:

Add additional checks:
require(_firstIssuedTimeStamp >= now, "Wrong timestamp");
require(_coiinAddress != address(0), "Wrong coiin address");
require(_mintAddress != address(0), "Wrong mint address");

### Additional checks are required for function setMintAddress() at CoiinMinter.sol

**Description**


There is no verification for the zero address for the mint address.

**Recommendation**:

Add additional check:
require(_mintAddress != address(0), "Wrong mint address");

### Additional checks are required for function setMultiSig() at CoiinMinter.sol

**Description**


There is no verification for the zero address for the multiSig address.

**Recommendation**:

Add additional check:
require(_multiSig != address(0), "Wrong multiSig address");

### Missed revert message in function mint() at CoiinMinter.sol

**Description**

require(lastMintTimestampSec.add(minMintTimeIntervalSec) < now);

**Recommendation**:

Add revert message.

### Not reachable require statement at CoiinMinter.sol

**Description**


In function _inMintWindow() you have two requires :
require(now.mod(minMintTimeIntervalSec) >= mintWindowOffsetSec, "too early");
require(now.mod(minMintTimeIntervalSec)<(mintWindowOffsetSec.add
(mintWindowLengthSec)), "too late");
mintWindowOffsetSec is set in the constructor of the contract and is equal to 0. So there is no
need for the first require and you can remove mintWindowOffsetSec from the second require.

**Recommendation**:

Remove “require” or add additional information about mintWindowOffsetSec if you want to use
it with different values and function _inMintWindow().

### Additional checks are required for constructor of CoiinVault.sol

**Description**

There is no verification for the zero address for the _coiinToken and _withdrawSigner.

**Recommendation**:

Add additional checks:
require(_withdrawSigner != address(0), "Wrong signer address");
require(_coiinToken != address(0), "Wrong token address");

### Additional checks are required for function setWithdrawSigner() at CoiinVault.sol

**Description**

There is no verification for the zero address for the _withdrawSigner.

**Recommendation**:

Add additional check:
require(account != address(0), "Wrong signer address");

### Additional checks are required for function setMultiSig() at CoiinVault.sol

**Description**


There is no verification for the zero address for the MultiSig address.

**Recommendation**:

Add additional check:
require(_multiSig != address(0), "Wrong multiSig address");

### Additional checks are required for constructor of RaiinmakerCampaignVault.sol

**Description**


There is no verification for the zero address for the _feeToken and _withdrawSigner.


**Recommendation**:

Add additional checks:
require(_withdrawSigner != address(0), "Wrong signer address");
require(_feeToken != address(0), "Wrong token address");

### Additional checks are required for function setWithdrawSigner() at RaiinmakerCampaignVault.sol

**Description**

There is no verification for the zero address for the _withdrawSigner.

**Recommendation**:

Add additional check:
require(account != address(0), "Wrong signer address");

### Additional checks are required for function deposit() at RaiinmakerCampaignVault.sol

**Description**

There is no verification for the zero address for the token and amount equal to zero.

**Recommendation**:

Add additional checks:
require(token != address(0), "Wrong token address");
require(amount > 0, "Wrong amount");

### Additional check are required for function changeMultiSig() at RaiinmakerCampaignVault.sol

**Description**

There is no verification for the zero address for the multiSig.

**Recommendation**:

Add additional check:
require(_multiSig != address(0), "Wrong multiSig address");

### Missed revert message in constructor at CoiinVestingVault.sol

require(address(_token) != address(0));

**Recommendation**:

Add revert message.

### Additional check are required for function deposit() at CoiinVestingVault.sol

**Description**

There is no verification for the deposit amount.

**Recommendation**:

Add additional check:
require(amount > 0, "Wrong amount");

### Additional check are required for function addTokenGrant() at CoiinVestingVault.sol

**Description**

There is no verification for the recipient address.

**Recommendation**:

Add additional check:
require(_recipient != address(0), "Wrong recipient address");

### Lock pragma to the specific version

**Description**

Lock the pragma to a specific version, since not all the EVM compiler versions support all the
features, especially the latest one’s which are kind of beta versions (for example you use
pragma solidity >=0.5.12 at CoiinVestingVault.sol ), so the intended behavior written in code
might not be executed as expected.

**Recommendation**:

Lock pragma to a specific version.

## Informational

### Misleading comment at CoiinToken.sol

**Description**

The variables names are maxMintMonthly and minMintMonthly but in notice, we see the daily
minting allowed:
/// @notice The daily allowed COIIN to be minted
uint256 public maxMintMonthly;
uint256 public minMintMonthly;

**Recommendation**:

Change comment or variables names.

### Additional optimization at CoiinToken.sol

**Description**

maxMintMonthly and minMintMonthly can be changed to constant as you don’t change them
further in the code:
/// @notice The daily allowed COIIN to be minted
uint256 public maxMintMonthly;
uint256 public minMintMonthly;

**Recommendation**:

Use constant for those variables.

### Additional optimization at CoiinMinter.sol

**Description**

minMintTimeIntervalSec, intWindowOffsetSec, mintWindowLengthSec can be changed to
constant as you don’t change them further in the code
minMintTimeIntervalSec = 4 weeks;
mintWindowOffsetSec = 0; // 12 AM UTC mint
mintWindowLengthSec = 14 days;
lastMintAmount = 1000000 * 1e18;

**Recommendation**:

Use constant for those variables.
