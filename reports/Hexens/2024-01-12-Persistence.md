**Auditors**

[Hexens](https://hexens.io/)

---

# Findings
## High Risk

### [PRST-5] User can lock and stake arbitrary amount of tokens without paying

**Severity:** Critical

**Path:** superfluid_lp/src/contract.rs

**Description:**

In the `superfluid_lp/src/contract.rs` contract the user has the ability to lock and stake native tokens. While staking the user has the following methods of paying for the staking:

Send the actual tokens with the transaction when staking

Use their locked up tokens as payment, which will be transferred from the contract

The locking function has a faulty amount check `superfluid_lp/src/contract.rs:L95-109`:

```
            match &asset.info {
                AssetInfo::NativeToken { denom } => {
                    for coin in info.funds.iter() {
                        if coin.denom == *denom {
                            // validate that the amount sent is exactly equal to the amount that is expected to be locked.
                            if coin.amount != asset.amount {
                                return Err(ContractError::InvalidAmount);
                            }
                        }
                    }
                }
                AssetInfo::Token { contract_addr: _ } => {
                    return Err(ContractError::UnsupportedAssetType);
                }
            }
```
The issue lies in the for loop part. If the user doesn’t supply any tokens while calling the function, the `coin.amount` check will never happen and because the function doesn’t check if the user has sent some tokens, the user locked amount will be incremented by the value that was supplied to the lock function. Even in the case where there was a requirement which enforced that the user has sent some tokens, the user could supply another native token and once again bypass the check.

Because of this the user can lockup any amount of tokens and later stake using other user's tokens without actually paying them which can lead to the following scenario:

Alice locks up 100 native tokens inside of the contract to be later used in staking.

Bob seeing as Alice has locked up native tokens without staking decides to abuse the invalid check and locks up 100 native tokens without actually paying those.

Bob immediately after falsely locking the tokens, instantly joins the pool using his fake locked tokens and the contract transfers the locked tokens of Alice but from the name of Bob.

```
ExecuteMsg::LockLstAsset { asset} => {

            let user = info.sender.clone();

            // validate that the asset is allowed to be locked.
            let config = CONFIG.load(deps.storage)?;
            let mut allowed = false;
            for allowed_asset in config.allowed_lockable_tokens {
                if allowed_asset == asset.info {
                    allowed = true;
                    break;
                }
            }

            if !allowed {
                return Err(ContractError::AssetNotAllowedToBeLocked);
            }

            let mut locked_amount: Uint128 = LOCK_AMOUNT
                .may_load(deps.storage, (&user, &asset.info.to_string()))?
                .unwrap_or_default();
       
            // confirm that this asset was sent along with the message. We only support native assets.
            match &asset.info {
                AssetInfo::NativeToken { denom } => {
                    for coin in info.funds.iter() {
                        if coin.denom == *denom {
                            // validate that the amount sent is exactly equal to the amount that is expected to be locked.
                            if coin.amount != asset.amount {
                                return Err(ContractError::InvalidAmount);
                            }
                        }
                    }
                }
                AssetInfo::Token { contract_addr: _ } => {
                    return Err(ContractError::UnsupportedAssetType);
                }
            }

             // add the amount to the locked amount
            locked_amount = locked_amount + asset.amount;

            // update locked amount
            LOCK_AMOUNT.save(deps.storage, (&user, &asset.info.to_string()), &locked_amount)?;
            Ok(Response::default())
        }
```

**Remediation:**  Add a check which enforces that the user sends the required token with the transaction.

**Status:**  Fixed


- - -

### [PRST-1] Division by zero in share conversion leads to panic and denial-of-service

**Severity:** High

**Path:** x/liquidstake/types/liquidstake.go:NativeTokenToStkXPRT, StkXPRTToNativeToken#L167-L175

**Description:**

Both functions `NativeTokenToStkXPRT` and `StkXPRTToNativeToken` are used to convert `xprt` to `stkxprt` and vice versa. The conversion is based on the total net amount of assets and the stkXPRT total supply.

However, neither function checks whether the denominator is zero, nor is this check done when either function is called.

This becomes a problem in the `keeper.LiquidStake` function when the user deposits `xprt` for `stkxprt`. Here `NativeTokenToStkXPRT` is called to calculate the amount of `stkxprt`.

If the protocol is in a state where there are some stkXPRT shares but no net assets, then a division by zero will happen. This will cause the message to revert with a Go run-time panic. This would make staking not possible.

```
// NativeTokenToStkXPRT returns StkxprtTotalSupply * nativeTokenAmount / netAmount
func NativeTokenToStkXPRT(nativeTokenAmount, stkXPRTTotalSupplyAmount math.Int, netAmount math.LegacyDec) (stkXPRTAmount math.Int) {
	return math.LegacyNewDecFromInt(stkXPRTTotalSupplyAmount).MulTruncate(math.LegacyNewDecFromInt(nativeTokenAmount)).QuoTruncate(netAmount.TruncateDec()).TruncateInt()
}

// StkXPRTToNativeToken returns stkXPRTAmount * netAmount / StkxprtTotalSupply with truncations
func StkXPRTToNativeToken(stkXPRTAmount, stkXPRTTotalSupplyAmount math.Int, netAmount math.LegacyDec) (nativeTokenAmount math.LegacyDec) {
	return math.LegacyNewDecFromInt(stkXPRTAmount).MulTruncate(netAmount).Quo(math.LegacyNewDecFromInt(stkXPRTTotalSupplyAmount)).TruncateDec()
}
```

**Remediation:**  Both functions should correctly handle the case where the denominator is zero in the same was that `MintRate` is calculated in `GetNetAmountState`.

**Status:**  Fixed

- - -

### [PRST-6] First depositor can steal assets due to missing slippage protection

**Severity:** High

**Path:** x/liquidstake/keeper/liquidstake.go:LiquidStake

**Description:**

The function to mint `stkXPRT` does not have any slippage protection. It only uses the input amount in `xprt`, but the output amount of minted shares is not checked, such as against a minimum amount out parameter. The output amount is only checked against zero, which is insufficient.

When a user deposits their `xprt` for `stkXPRT` the amount of minted shares are calculated from the exchange rate. The exchange rate is calculated from the total net asset amount and the total supply. The total net asset amount includes a `balanceOf` of `xprt`, which can be increased by an attacker to manipulate the share rate and cause rounding issues.

As a result, the first depositor can lose part of their funds to an attacker that front-runs their deposit message with the well-known deposit/donation attack.

```
func (k Keeper) LiquidStake(
	ctx sdk.Context, proxyAcc, liquidStaker sdk.AccAddress, stakingCoin sdk.Coin) (newShares math.LegacyDec, stkXPRTMintAmount math.Int, err error) {
    [..]
    nas := k.GetNetAmountState(ctx)
    [..]
	// mint stkxprt, MintAmount = TotalSupply * StakeAmount/NetAmount
	liquidBondDenom := k.LiquidBondDenom(ctx)
	stkXPRTMintAmount = stakingCoin.Amount
	if nas.StkxprtTotalSupply.IsPositive() {
		stkXPRTMintAmount = types.NativeTokenToStkXPRT(stakingCoin.Amount, nas.StkxprtTotalSupply, nas.NetAmount)
	}
	[..]
}

func (k Keeper) GetNetAmountState(ctx sdk.Context) (nas types.NetAmountState) {
	totalRemainingRewards, totalDelShares, totalLiquidTokens := k.CheckDelegationStates(ctx, types.LiquidStakeProxyAcc)

	totalUnbondingBalance := sdk.ZeroInt()
	ubds := k.stakingKeeper.GetAllUnbondingDelegations(ctx, types.LiquidStakeProxyAcc)
	for _, ubd := range ubds {
		for _, entry := range ubd.Entries {
			// use Balance(slashing applied) not InitialBalance(without slashing)
			totalUnbondingBalance = totalUnbondingBalance.Add(entry.Balance)
		}
	}

	nas = types.NetAmountState{
		StkxprtTotalSupply:    k.bankKeeper.GetSupply(ctx, k.LiquidBondDenom(ctx)).Amount,
		TotalDelShares:        totalDelShares,
		TotalLiquidTokens:     totalLiquidTokens,
		TotalRemainingRewards: totalRemainingRewards,
		TotalUnbondingBalance: totalUnbondingBalance,
		ProxyAccBalance:       k.GetProxyAccBalance(ctx, types.LiquidStakeProxyAcc).Amount,
	}

	nas.NetAmount = nas.CalcNetAmount()
	nas.MintRate = nas.CalcMintRate()
	return
}

func NativeTokenToStkXPRT(nativeTokenAmount, stkXPRTTotalSupplyAmount math.Int, netAmount math.LegacyDec) (stkXPRTAmount math.Int) {
	return math.LegacyNewDecFromInt(stkXPRTTotalSupplyAmount).MulTruncate(math.LegacyNewDecFromInt(nativeTokenAmount)).QuoTruncate(netAmount.TruncateDec()).TruncateInt()
}
```


**Remediation:**  We would recommend to also add a parameter containing the minimum amount of shares expected to be minted by the user. The amount would be calculated at the moment of signing and as a result, the transaction would fail if any manipulation would have happened before the moment of execution.

**Status:**  Acknowledged


- - -
## Medium Risk

### [PRST-4] Unbonding of validators does not give priority to inactive validators

**Severity:** Medium

**Path:** x/liquidstake/keeper/liquidstake.go:LiquidUnstake#L344-L459

**Description:**

When a user wants to withdraw their `stkXPRT` for `xprt`, they will call `LiquidUnstake`. In the function, the module will back out delegations for each validator according to their weight for a total of the unbonding amount. The module takes the whole set of validators and does not check their active status.

By not giving priority to unbonding inactive validators first, it will further lower the APY of staked XPRT.

```
func (k Keeper) LiquidUnstake(
	ctx sdk.Context, proxyAcc, liquidStaker sdk.AccAddress, unstakingStkXPRT sdk.Coin,
) (time.Time, math.Int, []stakingtypes.UnbondingDelegation, math.Int, error) {

	// check bond denomination
	params := k.GetParams(ctx)
	liquidBondDenom := k.LiquidBondDenom(ctx)
	if unstakingStkXPRT.Denom != liquidBondDenom {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), errors.Wrapf(
			types.ErrInvalidLiquidBondDenom, "invalid coin denomination: got %s, expected %s", unstakingStkXPRT.Denom, liquidBondDenom,
		)
	}

	// Get NetAmount states
	nas := k.GetNetAmountState(ctx)

	if unstakingStkXPRT.Amount.GT(nas.StkxprtTotalSupply) {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), types.ErrInvalidStkXPRTSupply
	}

	// UnstakeAmount = NetAmount * StkXPRTAmount/TotalSupply * (1-UnstakeFeeRate)
	unbondingAmount := types.StkXPRTToNativeToken(unstakingStkXPRT.Amount, nas.StkxprtTotalSupply, nas.NetAmount)
	unbondingAmount = types.DeductFeeRate(unbondingAmount, params.UnstakeFeeRate)
	unbondingAmountInt := unbondingAmount.TruncateInt()

	if !unbondingAmountInt.IsPositive() {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), types.ErrTooSmallLiquidUnstakingAmount
	}

	// burn stkxprt
	err := k.bankKeeper.SendCoinsFromAccountToModule(ctx, liquidStaker, types.ModuleName, sdk.NewCoins(unstakingStkXPRT))
	if err != nil {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), err
	}
	err = k.bankKeeper.BurnCoins(ctx, types.ModuleName, sdk.NewCoins(sdk.NewCoin(liquidBondDenom, unstakingStkXPRT.Amount)))
	if err != nil {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), err
	}

	liquidVals := k.GetAllLiquidValidators(ctx)
	totalLiquidTokens, liquidTokenMap := liquidVals.TotalLiquidTokens(ctx, k.stakingKeeper, false)

	// if no totalLiquidTokens, withdraw directly from balance of proxy acc
	if !totalLiquidTokens.IsPositive() {
		if nas.ProxyAccBalance.GTE(unbondingAmountInt) {
			err = k.bankKeeper.SendCoins(
				ctx,
				types.LiquidStakeProxyAcc,
				liquidStaker,
				sdk.NewCoins(sdk.NewCoin(
					k.stakingKeeper.BondDenom(ctx),
					unbondingAmountInt,
				)),
			)
			if err != nil {
				return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), err
			}

			return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, unbondingAmountInt, nil
		}

		// error case where there is a quantity that are unbonding balance or remaining rewards that is not re-stake or withdrawn in netAmount.
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), types.ErrInsufficientProxyAccBalance
	}

	// fail when no liquid validators to unbond
	if liquidVals.Len() == 0 {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), types.ErrLiquidValidatorsNotExists
	}

	// crumb may occur due to a decimal error in dividing the unstaking stkXPRT into the weight of liquid validators, it will remain in the NetAmount
	unbondingAmounts, crumb := types.DivideByCurrentWeight(liquidVals, unbondingAmount, totalLiquidTokens, liquidTokenMap)
	if !unbondingAmount.Sub(crumb).IsPositive() {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), types.ErrTooSmallLiquidUnstakingAmount
	}

	totalReturnAmount := sdk.ZeroInt()

	var ubdTime time.Time
	ubds := make([]stakingtypes.UnbondingDelegation, 0, len(liquidVals))
	for i, val := range liquidVals {
		// skip zero weight liquid validator
		if !unbondingAmounts[i].IsPositive() {
			continue
		}

		var ubd stakingtypes.UnbondingDelegation
		var returnAmount math.Int
		var weightedShare math.LegacyDec

		// calculate delShares from tokens with validation
		weightedShare, err = k.stakingKeeper.ValidateUnbondAmount(ctx, proxyAcc, val.GetOperator(), unbondingAmounts[i].TruncateInt())
		if err != nil {
			return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), err
		}

		if !weightedShare.IsPositive() {
			continue
		}

		// unbond with weightedShare
		ubdTime, returnAmount, ubd, err = k.LiquidUnbond(ctx, proxyAcc, liquidStaker, val.GetOperator(), weightedShare, true)
		if err != nil {
			return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), err
		}

		ubds = append(ubds, ubd)
		totalReturnAmount = totalReturnAmount.Add(returnAmount)
	}

	return ubdTime, totalReturnAmount, ubds, sdk.ZeroInt(), nil
}
```

**Remediation:**  The function `DivideByCurrentWeight` could take the active status of each validator into account and use that to calculate the sum of all liquid tokens of inactive validators first and return the full amounts in the outputs. The leftover could then be taken from active validators pro rata.

**Status:**   Fixed


- - -

## Low Risk

### [PRST-2] Whitelisted validator target weight is uncapped and unitless

**Severity:** Low

**Path:** x/liquidstake/types/params.go:validateWhitelistedValidators#L101-L126

**Description:**

The function `validateWhitelistedValidators` will validate a new set of whitelisted validators as proposed by Governance through the `UpdateParams` message.

However, this function only checks whether the target weight is positive, the actual value is not checked or scaled against anything.

This makes it so that a validator’s target weight by itself is meaningless, you have to compare it to the total target weight of all validators. Furthermore, the target weight is uncapped, so one validator with an enormous target weight would disable all other validators and take all power for themselves.

```func validateWhitelistedValidators(i interface{}) error { 
	wvs, ok := i.([]WhitelistedValidator)
	if !ok {
		return fmt.Errorf("invalid parameter type: %T", i)
	}
	valsMap := map[string]struct{}{}
	for _, wv := range wvs {
		_, valErr := sdk.ValAddressFromBech32(wv.ValidatorAddress)
		if valErr != nil {
			return valErr
		}

		if wv.TargetWeight.IsNil() {
			return fmt.Errorf("liquidstake validator target weight must not be nil")
		}
		if !wv.TargetWeight.IsPositive() {
			return fmt.Errorf("liquidstake validator target weight must be positive: %s", wv.TargetWeight)
		}

		if _, ok := valsMap[wv.ValidatorAddress]; ok {
			return fmt.Errorf("liquidstake validator cannot be duplicated: %s", wv.ValidatorAddress)
		}
		valsMap[wv.ValidatorAddress] = struct{}{}
	}
	return nil
}
```

**Remediation:**  We would recommend to implement some kind of cap and unit for the target weight. For example, 1 `Dec` would mean `1x` normal stake for a validator and there could be a cap of `10x` or `100x` as desired (or configurable by Governance).

**Status:** Fixed

- - -

### [PRST-3] Whitelisted validators cannot be inactivated

**Severity:** Low

**Path:** x/liquidstake/types/params.go:validateWhitelistedValidators#L101-L126

**Description:**

According to the documentation of a `WhitelistedValidator`, a validator is deemed inactive if their target weight is set to zero. This can be seen in `x/liquidstake/types/liquidstake.pb.go` on lines 122-125:

```
// WhitelistedValidator consists of the validator operator address and the
// target weight, which is a value for calculating the real weight to be derived
// according to the active status. In the case of inactive, it is calculated as
// zero.
```
However, this is not possible when updating the whitelisted validator set through `UpdateParams`. When it is validating the new set in `validateWhitelistedValidators` on lines 116-118:

```
if !wv.TargetWeight.IsPositive() {
	return fmt.Errorf("liquidstake validator target weight must be positive: %s", wv.TargetWeight)
}
```
The check will return an error if the target weight is `<= 0`, `isPositive` only returns true if the value is greater than zero.

```
func validateWhitelistedValidators(i interface{}) error { 
	wvs, ok := i.([]WhitelistedValidator)
	if !ok {
		return fmt.Errorf("invalid parameter type: %T", i)
	}
	valsMap := map[string]struct{}{}
	for _, wv := range wvs {
		_, valErr := sdk.ValAddressFromBech32(wv.ValidatorAddress)
		if valErr != nil {
			return valErr
		}

		if wv.TargetWeight.IsNil() {
			return fmt.Errorf("liquidstake validator target weight must not be nil")
		}
		if !wv.TargetWeight.IsPositive() {
			return fmt.Errorf("liquidstake validator target weight must be positive: %s", wv.TargetWeight)
		}

		if _, ok := valsMap[wv.ValidatorAddress]; ok {
			return fmt.Errorf("liquidstake validator cannot be duplicated: %s", wv.ValidatorAddress)
		}
		valsMap[wv.ValidatorAddress] = struct{}{}
	}
	return nil
}
```

**Remediation:**  Either the documentation should be amended to correctly state that inactive validators would not be included in the set, or the validation code should also allow a target weight of zero.

**Status:** Fixed


- - -

### [PRST-7] Total unbonding amounts can be greater than total liquid tokens in unstake

**Severity:** Low

**Path:** x/liquidstake/keeper/liquidstake.go:LiquidUnstake

**Description:**

The function LiquidUnstake allows a user to unstake by burning their shares and receiving unbondings for xprt.

The amount of xprt is calculated using the StkXPRTToNativeToken function, which takes the net asset value from GetNetAmountState. Afterwards, the amount is divided among validators based on their liquid token amounts and the total liquid token amount using DivideByCurrentWeight.

However, the net asset value calculation uses the module’s xprt balance, as well as unclaimed rewards in the total net amount, which is used to convert the user’s shares to the xprt amount. As a result, the unbondingAmount could be greater than totalLiquidTokens in the call to DivideByCurrentWeight.

In this case, too much would be unnecessarily unbonded from the validators or the execution would fail.

```func (k Keeper) LiquidUnstake(
	ctx sdk.Context, proxyAcc, liquidStaker sdk.AccAddress, unstakingStkXPRT sdk.Coin,
) (time.Time, math.Int, []stakingtypes.UnbondingDelegation, math.Int, error) {
	[..]
	// Get NetAmount states
	nas := k.GetNetAmountState(ctx)

	if unstakingStkXPRT.Amount.GT(nas.StkxprtTotalSupply) {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), types.ErrInvalidStkXPRTSupply
	}

	// UnstakeAmount = NetAmount * StkXPRTAmount/TotalSupply * (1-UnstakeFeeRate)
	unbondingAmount := types.StkXPRTToNativeToken(unstakingStkXPRT.Amount, nas.StkxprtTotalSupply, nas.NetAmount)
	unbondingAmount = types.DeductFeeRate(unbondingAmount, params.UnstakeFeeRate)
	unbondingAmountInt := unbondingAmount.TruncateInt()
	[..]
	liquidVals := k.GetAllLiquidValidators(ctx)
	totalLiquidTokens, liquidTokenMap := liquidVals.TotalLiquidTokens(ctx, k.stakingKeeper, false)
	[..]
	unbondingAmounts, crumb := types.DivideByCurrentWeight(liquidVals, unbondingAmount, totalLiquidTokens, liquidTokenMap)
	if !unbondingAmount.Sub(crumb).IsPositive() {
		return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), types.ErrTooSmallLiquidUnstakingAmount
	}

	totalReturnAmount := sdk.ZeroInt()

	var ubdTime time.Time
	ubds := make([]stakingtypes.UnbondingDelegation, 0, len(liquidVals))
	for i, val := range liquidVals {
		// skip zero weight liquid validator
		if !unbondingAmounts[i].IsPositive() {
			continue
		}

		var ubd stakingtypes.UnbondingDelegation
		var returnAmount math.Int
		var weightedShare math.LegacyDec

		// calculate delShares from tokens with validation
		weightedShare, err = k.stakingKeeper.ValidateUnbondAmount(ctx, proxyAcc, val.GetOperator(), unbondingAmounts[i].TruncateInt())
		if err != nil {
			return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), err
		}

		if !weightedShare.IsPositive() {
			continue
		}

		// unbond with weightedShare
		ubdTime, returnAmount, ubd, err = k.LiquidUnbond(ctx, proxyAcc, liquidStaker, val.GetOperator(), weightedShare, true)
		if err != nil {
			return time.Time{}, sdk.ZeroInt(), []stakingtypes.UnbondingDelegation{}, sdk.ZeroInt(), err
		}

		ubds = append(ubds, ubd)
		totalReturnAmount = totalReturnAmount.Add(returnAmount)
	}

	return ubdTime, totalReturnAmount, ubds, sdk.ZeroInt(), nil
}

func DivideByCurrentWeight(lvs LiquidValidators, input math.LegacyDec, totalLiquidTokens math.Int, liquidTokenMap map[string]math.Int) (outputs []math.LegacyDec, crumb math.LegacyDec) {
	if !totalLiquidTokens.IsPositive() {
		return []math.LegacyDec{}, sdk.ZeroDec()
	}

	totalOutput := sdk.ZeroDec()
	unitInput := input.QuoTruncate(math.LegacyNewDecFromInt(totalLiquidTokens))
	for _, val := range lvs {
		output := unitInput.MulTruncate(math.LegacyNewDecFromInt(liquidTokenMap[val.OperatorAddress])).TruncateDec()
		totalOutput = totalOutput.Add(output)
		outputs = append(outputs, output)
	}

	return outputs, input.Sub(totalOutput)
}

func StkXPRTToNativeToken(stkXPRTAmount, stkXPRTTotalSupplyAmount math.Int, netAmount math.LegacyDec) (nativeTokenAmount math.LegacyDec) {
	return math.LegacyNewDecFromInt(stkXPRTAmount).MulTruncate(netAmount).Quo(math.LegacyNewDecFromInt(stkXPRTTotalSupplyAmount)).TruncateDec()
}

func (k Keeper) GetNetAmountState(ctx sdk.Context) (nas types.NetAmountState) {
	totalRemainingRewards, totalDelShares, totalLiquidTokens := k.CheckDelegationStates(ctx, types.LiquidStakeProxyAcc)

	totalUnbondingBalance := sdk.ZeroInt()
	ubds := k.stakingKeeper.GetAllUnbondingDelegations(ctx, types.LiquidStakeProxyAcc)
	for _, ubd := range ubds {
		for _, entry := range ubd.Entries {
			// use Balance(slashing applied) not InitialBalance(without slashing)
			totalUnbondingBalance = totalUnbondingBalance.Add(entry.Balance)
		}
	}

	nas = types.NetAmountState{
		StkxprtTotalSupply:    k.bankKeeper.GetSupply(ctx, k.LiquidBondDenom(ctx)).Amount,
		TotalDelShares:        totalDelShares,
		TotalLiquidTokens:     totalLiquidTokens,
		TotalRemainingRewards: totalRemainingRewards,
		TotalUnbondingBalance: totalUnbondingBalance,
		ProxyAccBalance:       k.GetProxyAccBalance(ctx, types.LiquidStakeProxyAcc).Amount,
	}

	nas.NetAmount = nas.CalcNetAmount()
	nas.MintRate = nas.CalcMintRate()
	return
}
```

**Remediation:**  The LiquidUnstake function has a branch where the user receives the xprt from the module’s balance, but this only happens if the totalLiquidTokens is 0 and there is enough balance. We would recommend to have the function first use any existing balance to convert the xprt amount and use the remaining value to unbond from validators.

**Status:**  Acknowledged

- - -