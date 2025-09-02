**Auditors**

[Hexens](https://hexens.io/)

---

# Findings
## High Risk

### [STAKE-6] Unprotected flash loan callback can be abused to manipulate/claim other users' positions

**Severity:** Critical

**Path:** LeverageStrategy.sol:receiveFlashLoan#L304-L333

**Description:** The Leverage Strategy contract uses flash loans from the osToken Flash Loan module to flash loan osTokens and perform actions, such as deposit, claim and rescue. Each of these functions performs the flash loan with specific parameters in the `userData` field.

The callback function `receiveFlashLoan` will be called by the Flash Loan module and while the function correctly checks that the caller is the module, it does not check whether the flash loan was initiated by the contract itself. 

So it becomes possible for an attacker to initiate a flash loan with the Leverage Strategy as receiver. The `userData` can then be arbitrarily chosen by the attacker, which depending on the action can be the proxy address, vault address or exit position ID. 

The attacker can make use of the specific Flash Loan callback functions to manipulate any arbitrary proxy and change their leveraged positions or make a claim and cause a deadlock in exiting because `isStrategyProxyExiting` was never cleared. 
```
    function receiveFlashLoan(uint256 osTokenShares, bytes memory userData) external {
        // validate sender
        if (msg.sender != address(_osTokenFlashLoans)) {
            revert Errors.AccessDenied();
        }

        // decode userData action
        (FlashloanAction flashloanType) = abi.decode(userData, (FlashloanAction));
        if (flashloanType == FlashloanAction.Deposit) {
            // process deposit flashloan
            (, address vault, address proxy) = abi.decode(userData, (FlashloanAction, address, address));
            _processDepositFlashloan(vault, proxy, osTokenShares);
        } else if (flashloanType == FlashloanAction.ClaimExitedAssets) {
            // process claim exited assets flashloan
            (, address vault, address proxy, uint256 exitPositionTicket) =
                abi.decode(userData, (FlashloanAction, address, address, uint256));
            _processClaimFlashloan(vault, proxy, exitPositionTicket, osTokenShares);
        } else if (flashloanType == FlashloanAction.RescueVaultAssets) {
            // process vault assets rescue flashloan
            (, address vault, address proxy, uint256 exitPositionTicket) =
                abi.decode(userData, (FlashloanAction, address, address, uint256));
            _processVaultAssetsRescueFlashloan(vault, proxy, exitPositionTicket, osTokenShares);
        } else if (flashloanType == FlashloanAction.RescueLendingAssets) {
            // process lending assets rescue flashloan
            (, address proxy, uint256 assets) = abi.decode(userData, (FlashloanAction, address, uint256));
            _processLendingAssetsRescueFlashloan(proxy, assets, osTokenShares);
        } else {
            revert InvalidFlashloanAction();
        }
    }
```


**Remediation:**  The `receiveFlashLoan` callback function should check whether the contract itself initiated the flash loan. This can be done using a storage variable that is set to true before the initiating function calls `flashLoan` and set to false after.

For example:
```
function deposit(address vault, uint256 osTokenShares) external {
  [..]
  if (performingFlashLoan) {
    revert Errors.Reentrant();
  }
  performingFlashLoan = true;
  // execute flashloan
  _osTokenFlashLoans.flashLoan(
    address(this), flashloanOsTokenShares, abi.encode(FlashloanAction.Deposit, vault, proxy)
  );
  performingFlashLoan = false;
  [..]
}

function receiveFlashLoan(uint256 osTokenShares, bytes memory userData) external {
  if (!performingFlashLoan) {
    revert Errors.AccessDenied();
  }
  [..]
}
```


**Status:**  Fixed


- - -

##Medium Risk

### [STAKE-7] Lack of slippage protection in deposit function

**Severity:** Medium

**Path:** src/leverage/LeverageStrategy.sol::deposit()#L125-L182

**Description:** The `deposit()` function in the `LeverageStrategy.sol` contract executes several key operations involving token transfers, asset borrowing, and minting, including the potential use of flash loans. However, it lacks slippage protection, meaning that if token prices fluctuate between the time the transaction is submitted and executed, users could receive fewer tokens or face adverse outcomes without the ability to revert or mitigate the loss.

In particular, the function lacks checks to ensure that the actual number of `osTokenShares` minted matches expectations after asset borrowing.

Example:
A user submits a transaction to deposit 100 `osTokenShares`, expecting to leverage them to borrow additional assets. Between the time the transaction is broadcasted and mined, market conditions change (e.g., a large price movement), leading to less favorable borrowing conditions. Since there is no slippage check, the user could end up with fewer minted `osTokenShares` than expected, negatively impacting their strategy.
```
    function deposit(address vault, uint256 osTokenShares) external {
        if (osTokenShares == 0) revert Errors.InvalidShares();

        // fetch strategy proxy
        (address proxy,) = _getOrCreateStrategyProxy(vault, msg.sender);
        if (isStrategyProxyExiting[proxy]) revert Errors.ExitRequestNotProcessed();

        // transfer osToken shares from user to the proxy
        IStrategyProxy(proxy).execute(
            address(_osToken), abi.encodeWithSelector(_osToken.transferFrom.selector, msg.sender, proxy, osTokenShares)
        );

        // fetch vault state and lending protocol state
        (uint256 stakedAssets, uint256 mintedOsTokenShares) = _getVaultState(vault, proxy);
        (uint256 borrowedAssets, uint256 suppliedOsTokenShares) = _getBorrowState(proxy);

        // check whether any of the positions exist
        uint256 leverageOsTokenShares = osTokenShares;
        if (stakedAssets != 0 || mintedOsTokenShares != 0 || borrowedAssets != 0 || suppliedOsTokenShares != 0) {
            // supply osToken shares to the lending protocol
            _supplyOsTokenShares(proxy, osTokenShares);
            suppliedOsTokenShares += osTokenShares;

            // borrow max amount of assets from the lending protocol
            uint256 maxBorrowAssets =
                Math.mulDiv(_osTokenVaultController.convertToAssets(suppliedOsTokenShares), _getBorrowLtv(), _wad);
            if (borrowedAssets >= maxBorrowAssets) {
                // nothing to borrow
                emit Deposited(vault, msg.sender, osTokenShares, 0);
                return;
            }
            uint256 assetsToBorrow;
            unchecked {
                // cannot underflow because maxBorrowAssets > borrowedAssets
                assetsToBorrow = maxBorrowAssets - borrowedAssets;
            }
            _borrowAssets(proxy, assetsToBorrow);

            // mint max possible osToken shares
            leverageOsTokenShares = _mintOsTokenShares(vault, proxy, assetsToBorrow, type(uint256).max);
            if (leverageOsTokenShares == 0) {
                // no osToken shares to leverage
                emit Deposited(vault, msg.sender, osTokenShares, 0);
                return;
            }
        }

        // calculate flash loaned osToken shares
        uint256 flashloanOsTokenShares = _getFlashloanOsTokenShares(vault, leverageOsTokenShares);

        // execute flashloan
        _osTokenFlashLoans.flashLoan(
            address(this), flashloanOsTokenShares, abi.encode(FlashloanAction.Deposit, vault, proxy)
        );

        // emit event
        emit Deposited(vault, msg.sender, osTokenShares, flashloanOsTokenShares);
    }
```

**Remediation:**  Allow users to specify a minimum number of `osTokenShares` they are willing to accept after leverage calculations.

**Status:**  Acknowledged

- - -

## Low Risk

### [STAKE-5] Permit function can be frontrunned

**Severity:** Low

**Path:** src/leverage/LeverageStrategy.sol::permit()#L114-L122

**Description:** The `permit()` function within the `LeverageStrategy.sol` contract is designed to allow token approvals through signatures, leveraging the ERC-2612 standard. However, the function is vulnerable to front-running attacks. A malicious actor could observe e.g. Alice's signature in the mempool and use it to execute a transaction on her behalf before Alice’s original transaction is mined. Since signatures are single-use and tied to nonces, once the signature is exploited, Alice’s original transaction would fail, resulting in wasted gas and the function DOS.
```
    function permit(address vault, uint256 osTokenShares, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external {
        (address proxy,) = _getOrCreateStrategyProxy(vault, msg.sender);
        IStrategyProxy(proxy).execute(
            address(_osToken),
            abi.encodeWithSelector(
                IERC20Permit(address(_osToken)).permit.selector, msg.sender, proxy, osTokenShares, deadline, v, r, s
            )
        );
    }
```

**Remediation:**  Consider adding `try/cath`, or `if` block, so if the `permit()` is already called, just check the allowance of `msg.sender` and skip the call to pemit().

**Status:**  Fixed


- - -

### [STAKE-9] Incorrect rounding in mintShares and cumulativeFeePerShare calculations

**Severity:** Low

**Path:** lib/v3-core/contracts/tokens/OsTokenVaultController.sol::mintShares()#L103-L129

lib/v3-core/contracts/tokens/OsTokenVaultController.sol::cumulativeFeePerShare()#L200-L228

**Description:** In the `OsTokenVaultController.sol` contract, the functions `mintShares()` and `cumulativeFeePerShare()` use rounding mode `Math.Rounding.Floor`, which can lead to incorrect asset-to-share conversions. Specifically, this rounding method allows the user to underpay during share minting and fee calculations, which results in values that consistently favor the user over the protocol.

The `mintShares()` function calculates the number of assets needed to mint a certain number of shares by using the `convertToAssets()` method, which in turn calls `_convertToAssets()` with `Math.Rounding.Floor`:
```
  function convertToAssets(uint256 shares) public view override returns (uint256 assets) {
    return _convertToAssets(shares, _totalShares, totalAssets(), Math.Rounding.Floor);
  }
```
allowing the user to pay less than the required amount of assets. This discrepancy might accumulate over multiple transactions, resulting in a protocol loss.
The fee calculation also uses the Math.Rounding.Floor method, potentially under-crediting the treasury with fewer assets than it is entitled to. Over time, this rounding method results in the protocol losing part of its fee revenue.

Example:
If a user mints 100 shares, and the actual cost should be 91.5 assets, the rounding down behavior causes the user to pay only 91 assets. Over time, this discrepancy can accumulate, leading to a significant shortfall in the protocol's asset pool. Similarly, the protocol's treasury fee is rounded down, reducing the fees collected by the protocol.
```
  function mintShares(address receiver, uint256 shares) external override returns (uint256 assets) {
    if (
      !IVaultsRegistry(_registry).vaults(msg.sender) ||
      !IVaultsRegistry(_registry).vaultImpls(IVaultVersion(msg.sender).implementation())
    ) {
      revert Errors.AccessDenied();
    }
    if (receiver == address(0)) revert Errors.ZeroAddress();
    if (shares == 0) revert Errors.InvalidShares();

    // pull accumulated rewards
    updateState();

    // calculate amount of assets to mint
    assets = convertToAssets(shares);

    uint256 totalAssetsAfter = _totalAssets + assets;
    if (totalAssetsAfter > capacity) revert Errors.CapacityExceeded();

    // update counters
    _totalShares += SafeCast.toUint128(shares);
    _totalAssets = SafeCast.toUint128(totalAssetsAfter);

    // mint shares
    IOsToken(_osToken).mint(receiver, shares);
    emit Mint(msg.sender, receiver, assets, shares);
  }
```
```
  function cumulativeFeePerShare() external view override returns (uint256) {
    // SLOAD to memory
    uint256 currCumulativeFeePerShare = _cumulativeFeePerShare;

    // calculate rewards
    uint256 profitAccrued = _unclaimedAssets();
    if (profitAccrued == 0) return currCumulativeFeePerShare;

    // calculate treasury assets
    uint256 treasuryAssets = Math.mulDiv(profitAccrued, feePercent, _maxFeePercent);
    if (treasuryAssets == 0) return currCumulativeFeePerShare;

    // SLOAD to memory
    uint256 totalShares_ = _totalShares;

    // calculate treasury shares
    uint256 treasuryShares;
    unchecked {
      treasuryShares = _convertToShares(
        treasuryAssets,
        totalShares_,
        // cannot underflow because profitAccrued >= treasuryAssets
        _totalAssets + profitAccrued - treasuryAssets,
        Math.Rounding.Floor
      );
    }

    return currCumulativeFeePerShare + Math.mulDiv(treasuryShares, _wad, totalShares_);
  }
```

**Remediation:**  In the mentioned functions, use `Math.Rounding.Ceil` for the `convertToShares` calculations.

**Status:**   Acknowledged


- - -

## Informational

### [STAKE-8] flashloanOsTokenShares should be validated as non-zero before the flashloan

**Severity:** Low

**Path:** src/leverage/LeverageStrategy.sol#L165-L178

**Description:** In the `deposit()` function, there's a check that returns without executing the flashloan if `leverageOsTokenShares == 0`. However, the `flashloanOsTokenShares` variable, calculated by `_getFlashloanOsTokenShares()`, may still be 0 even when `leverageOsTokenShares` > 0.
```
uint256 flashloanOsTokenShares = _getFlashloanOsTokenShares(vault, leverageOsTokenShares); 
```
This can cause reverts because of borrowing zero during deposit when `leverageOsTokenShares` is very small. 
Therefore, to avoid the zero flashloan, it should check `flashloanOsTokenShares` instead of `leverageOsTokenShares`.
```
function deposit(address vault, uint256 osTokenShares) external {
    if (osTokenShares == 0) revert Errors.InvalidShares();

    // fetch strategy proxy
    (address proxy,) = _getOrCreateStrategyProxy(vault, msg.sender);
    if (isStrategyProxyExiting[proxy]) revert Errors.ExitRequestNotProcessed();

    // transfer osToken shares from user to the proxy
    IStrategyProxy(proxy).execute(
        address(_osToken), abi.encodeWithSelector(_osToken.transferFrom.selector, msg.sender, proxy, osTokenShares)
    );

    // fetch vault state and lending protocol state
    (uint256 stakedAssets, uint256 mintedOsTokenShares) = _getVaultState(vault, proxy);
    (uint256 borrowedAssets, uint256 suppliedOsTokenShares) = _getBorrowState(proxy);

    // check whether any of the positions exist
    uint256 leverageOsTokenShares = osTokenShares;
    if (stakedAssets != 0 || mintedOsTokenShares != 0 || borrowedAssets != 0 || suppliedOsTokenShares != 0) {
        // supply osToken shares to the lending protocol
        _supplyOsTokenShares(proxy, osTokenShares);
        suppliedOsTokenShares += osTokenShares;

        // borrow max amount of assets from the lending protocol
        uint256 maxBorrowAssets =
            Math.mulDiv(_osTokenVaultController.convertToAssets(suppliedOsTokenShares), _getBorrowLtv(), _wad);
        if (borrowedAssets >= maxBorrowAssets) {
            // nothing to borrow
            emit Deposited(vault, msg.sender, osTokenShares, 0);
            return;
        }
        uint256 assetsToBorrow;
        unchecked {
            // cannot underflow because maxBorrowAssets > borrowedAssets
            assetsToBorrow = maxBorrowAssets - borrowedAssets;
        }
        _borrowAssets(proxy, assetsToBorrow);

        // mint max possible osToken shares
        leverageOsTokenShares = _mintOsTokenShares(vault, proxy, assetsToBorrow, type(uint256).max);
        if (leverageOsTokenShares == 0) {
            // no osToken shares to leverage
            emit Deposited(vault, msg.sender, osTokenShares, 0);
            return;
        }
    }

    // calculate flash loaned osToken shares
    uint256 flashloanOsTokenShares = _getFlashloanOsTokenShares(vault, leverageOsTokenShares);

    // execute flashloan
    _osTokenFlashLoans.flashLoan(
        address(this), flashloanOsTokenShares, abi.encode(FlashloanAction.Deposit, vault, proxy)
    );

    // emit event
    emit Deposited(vault, msg.sender, osTokenShares, flashloanOsTokenShares);
}
```

**Remediation:**  Replace the check for `leverageOsTokenShares == 0` with `flashloanOsTokenShares`, as follows:
```
      // mint max possible osToken shares
      leverageOsTokenShares = _mintOsTokenShares(vault, proxy, assetsToBorrow, type(uint256).max);
  }

  // calculate flash loaned osToken shares
  uint256 flashloanOsTokenShares = _getFlashloanOsTokenShares(vault, leverageOsTokenShares);
  
  if (flashloanOsTokenShares == 0) {
      // no osToken shares to leverage
      emit Deposited(vault, msg.sender, osTokenShares, 0);
      return;
  }
  // execute flashloan
  _osTokenFlashLoans.flashLoan(
      address(this), flashloanOsTokenShares, abi.encode(FlashloanAction.Deposit, vault, proxy)
  );
```

**Status:** Fixed

- - -

