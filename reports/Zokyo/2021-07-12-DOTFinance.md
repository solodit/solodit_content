**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Compilation error

![image](https://github.com/user-attachments/assets/9673f912-8e86-492c-8e88-610adfffb12d)

## Low Risk

### Data structure that often used but never filled

**Lines**: PinkChef.sol#51
mapping(address => mapping(address => UserInfo)) vaultUsers;

**Recommendation**:
Add method to change data structure.

### Missing events access control

**Description**
VaultController.setKeeper(address) should emit an event for: changing keeper address.

**Lines**: VaultController.sol#98-100

function setKeeper(address _keeper) external onlyKeeper {
require(_keeper != address(0), 'VaultController: invalid keeper address');
keeper = _keeper;
}

## Informational

### Vulnerability: Public function that could be declared external

**Lines**: PinkChef.sol#131
function addVault(address vault, address token, uint allocPoint) public onlyOwner {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Performing a multiplication on the result of division

Under some circumstances, it could result in inaccuracy.
**Lines**: PinkPool.sol#90-96

uint rewardPerTokenPerSecond = rewardRate.mul(tokenDecimals).div(__totalSupply);
uint PinkPrice = helper.tokenPriceInBNB(address(stakingToken));
uint flipPrice = helper.tvlInBNB(address(rewardsToken), 1e18);
_usd = 0;
_Pink = 0;
_bnb = rewardPerTokenPerSecond.mul(365 days).mul(flipPrice).div(PinkPrice);

### Vulnerability: Public function that could be declared external

Lines: VaultController.sol#117
function setPink(address _token) public onlyOwner {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: Treasure.sol#157
function zapAllAssetsToPinkBNB() public onlyKeeper {

**Recommendation**:
Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: PriceCalculatorBSC.sol#78
function priceOfBNB() view public returns (uint) {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

Lines: PriceCalculatorBSC.sol#83
function priceOfCake() view public returns (uint) {

**Recommendation**:
Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: PriceCalculatorBSC.sol#88
function priceOfPink() view public returns (uint) {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: PinkChef.sol#119
function pendingPink(address vault, address user) public view override returns (uint) {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: PinkChef.sol#141
function updateVault(address vault, uint allocPoint) public onlyOwner {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: PinkMinterV1.sol#191
function performanceFee(uint profit) public view override returns (uint) {
**Recommendation**:
Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: PinkToken.sol#35
function mint(address _to, uint256 _amount) public onlyOwner {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: PinkPool.sol#99
function withdrawableBalanceOf(address account) override public view returns (uint) {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: VaultFlipToFlip.sol#107
function withdrawableBalanceOf(address account) public view override returns (uint) {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: VaultPink.sol#106
function earned(address) override public view returns (uint) {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: VaultPink.sol#154
function harvest() public override {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: VaultController.sol#103
function setMinter(address newMinter) virtual public onlyOwner {

**Recommendation**:
Public functions that are never called by the contract should be declared external to save gas.

### Vulnerability: Public function that could be declared external

**Lines**: VaultController.sol#119
function setPinkChef(IPinkChef newPinkChef) virtual public onlyOwner {

**Recommendation**:

Public functions that are never called by the contract should be declared external to save gas.
