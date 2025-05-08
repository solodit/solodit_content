**Auditor**

[@IAm0x52](https://twitter.com/IAm0x52)

# Findings

## High Risk

### [H-01] `InterestRateStrategyV1` ownership can be hijacked

**Details**

[Ownable.sol#L84-L85](https://github.com/Vectorized/solady/blob/9298d096feb87de9a8873a704ff98f6892064c65/src/auth/Ownable.sol#L84-L85)

    /// @dev Override to return true to make `_initializeOwner` prevent double-initialization.
    function _guardInitializeOwner() internal pure virtual returns (bool guard) {}

When utilizing Solady's `Ownable` contract with an upgradable contract, `_guardInitializeOwner` must be overridden to return `true` as communicated in the dev comment above. `InterestRateStrategyV1` is upgradable but does not do this, allowing the contract ownership to by hijacked. Consequences of this could be malicious upgrades to apply large amounts of interest to positions to liquidate all positions and the like.

**Lines of Code**

[Ownable.sol#L84-L85](https://github.com/Vectorized/solady/blob/9298d096feb87de9a8873a704ff98f6892064c65/src/auth/Ownable.sol#L84-L85)

**Recommendation**

`_guardInitializeOwner` should be overridden to return `true`.

**Remediation**

Fixed in [edb531f](https://github.com/hyperstable/contracts/commit/edb531f7b48ac805e461c1663994e002fe4a3c44). Solady `Ownable` was replaced with OZ `OwnableUpgradable`.

### [H-02] `EmissionScheduler#epochEmission` lacks access control and can be called by anyone to cause epoch to mint no emissions

**Details**

[EmissionScheduler.sol#L63-L80](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/EmissionScheduler.sol#L63-L80)

    @>  function epochEmission(uint256 _pegSupply, uint256 _veSupply) external returns (uint256, uint256, uint256) {
            uint256 currentEpoch = _currentEpoch();

            if (currentEpoch == lastEpoch) {
                return (0, 0, 0);
            }

    @>      lastEpoch = currentEpoch;

            uint256 toEmit = (_pegSupply - _veSupply).mulDiv(2, EMISSION_PRECISION);

            lastEpochEmission = toEmit;

            uint256 rebase = _calculateRebase(toEmit, _pegSupply, _veSupply);
            uint256 teamEmission = _calculateTeamEmission(toEmit, rebase);

            return (toEmit, rebase, teamEmission);
        }

We see above that `lastEpoch` is updated to `currentEpoch` when called. Also notice that there is no access control on calls to this function and therefore it can be called by anyone. The result in that when the legitimate call is made it will return `(0, 0, 0)`, completely wipe out all emissions owed for the epoch.

**Lines of Code**

[EmissionScheduler.sol#L63-L80](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/EmissionScheduler.sol#L63-L80)

**Recommendation**

`epochEmission` should only be callable by `minter`.

**Remediation**

Fixed in [7866674](https://github.com/hyperstable/contracts/commit/786667407e84ff76111aa0a8213b698f791f532d). Creates `_onlyMinter` which provides access control and applies it to `EmissionScheduler#epochEmission`.

### [H-03] `RewardDistributor#claim` will revert for tokens with expired locks leading to loss of rewards

**Details**

[RewardsDistributor.sol#L277-L287](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/RewardsDistributor.sol#L277-L287)

        function claim(uint256 _tokenId) external returns (uint256) {
            if (block.timestamp >= time_cursor) _checkpoint_total_supply();
            uint256 _last_token_time = last_token_time;
            _last_token_time = _last_token_time / WEEK * WEEK;
            uint256 amount = _claim(_tokenId, voting_escrow, _last_token_time);
            if (amount != 0) {
    @>          IVotingEscrow(voting_escrow).deposit_for(_tokenId, amount);
                token_last_balance -= amount;
            }
            return amount;
        }

We see above that when claiming for a token, the contract always attempts to deposit them to the current `tokenId`.

[vePeg.sol#L732-L739](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/vePeg.sol#L732-L739)

        function deposit_for(uint256 _tokenId, uint256 _value) external nonreentrant {
            LockedBalance memory _locked = locked[_tokenId];

            require(_value > 0); // dev: need non-zero value
            require(_locked.amount > 0, "No existing lock found");
    @>      require(_locked.end > block.timestamp, "Cannot add to expired lock. Withdraw");
            _deposit_for(_tokenId, _value, 0, _locked, DepositType.DEPOSIT_FOR_TYPE);
        }

Note that this call will `revert` if the token lock has expired. This means the above claim call will also `revert`.

[vePeg.sol#L795-L807](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/vePeg.sol#L795-L807)

        function increase_unlock_time(uint256 _tokenId, uint256 _lock_duration) external nonreentrant {
            assert(_isApprovedOrOwner(msg.sender, _tokenId));

            LockedBalance memory _locked = locked[_tokenId];
            uint256 unlock_time = (block.timestamp + _lock_duration) / WEEK * WEEK; // Locktime is rounded down to weeks

    @>      require(_locked.end > block.timestamp, "Lock expired");
            require(_locked.amount > 0, "Nothing is locked");
            require(unlock_time > _locked.end, "Can only increase lock duration");
            require(unlock_time <= block.timestamp + MAXTIME, "Voting lock can be 4 years max");

            _deposit_for(_tokenId, 0, unlock_time, _locked, DepositType.INCREASE_UNLOCK_TIME);
        }

Additionally once a token has expired the lock cannot be extended. As a result once a lock has expired, all rewards will be permanently locked and can never be claimed.

**Lines of Code**

[RewardsDistributor.sol#L277-L287](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/RewardsDistributor.sol#L277-L287)

**Recommendation**

`RewardDistributor#claim` should transfer tokens directly to owner in the event that the lock is expired.

**Remediation**

Fixed in [3012a71](https://github.com/hyperstable/contracts/commit/3012a71458ded1fdef57003b622bbd90d96f6b13). Adds additional logic to `RewardsDistributor#claim` to transfer rewards directly to token owner if the lock has expired.

## Medium Risk

### [M-01] `PositionManager#liquidatePosition` fails to update vaultDebt and vaultCollateral

**Details**

**First discovered by Dev team during audit period**

[PositionManager.sol#L169-L185](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/core/PositionManager.sol#L169-L185)

        function liquidatePosition(address _vault, address _target, uint256 _debtToRepay, uint256 _sharesToLiquidate)
            external
        {
            if (msg.sender != liquidationManager) {
                revert OnlyLiquidatorManager();
            }
            // Debt and collateral shares are adjusted for the liquidated account
            _debtSnapshot[_target][_vault] -= _debtToRepay;
            collateralShares[_target][_vault] -= _sharesToLiquidate;


    @>      uint256 updatedVaultDebt = _vaultDebtSnapshot[_vault] - _debtToRepay;
    @>      uint256 updatedVaultCollateral = vaultCollateral[_vault] - _sharesToLiquidate;


            interestRateStrategy.updateVaultInterestRate(
                _vault, updatedVaultCollateral.mulWad(IVault(_vault).sharePrice()), updatedVaultDebt, IVault(_vault).MCR()
            );
        }

In the above lines we see that `_vaultDebtSnapshot` and `vaultCollateral` are not update when performing the liquidation. This leaves phantom collateral and debt in the system which can lead to inaccurate interest rates.

**Lines of Code**

[PositionManager.sol#L179-L180](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/core/PositionManager.sol#L179-L180)

**Recommendation**

`_vaultDebtSnapshot` and `vaultCollateral` should be updated

**Remediation**

Fixed in [1277f39](https://github.com/hyperstable/contracts/commit/1277f396dcd14f00c6258eb1f9668f43be1d311f). Updated values are now correctly written to storage.

### [M-02] Airdrop supply methodology has been changed leading to excess token emissions

**Details**

[MerkleClaim.sol#L48-L69](https://github.com/velodrome-finance/v1/blob/de6b2a19b5174013112ad41f07cf98352bfe1f24/contracts/redeem/MerkleClaim.sol#L48-L69)

        function claim(
            address to,
            uint256 amount,
            bytes32[] calldata proof
        ) external {
            // Throw if address has already claimed tokens
            require(!hasClaimed[to], "ALREADY_CLAIMED");

            // Verify merkle proof, or revert if not in tree
            bytes32 leaf = keccak256(abi.encodePacked(to, amount));
            bool isValidLeaf = MerkleProof.verify(proof, merkleRoot, leaf);
            require(isValidLeaf, "NOT_IN_MERKLE");

            // Set address to claimed
            hasClaimed[to] = true;

            // Claim tokens for address
    @>      require(VELO.claim(to, amount), "CLAIM_FAILED");

            // Emit claim event
            emit Claim(to, amount);
        }

Above is the original `VELO` `MerkleClaim` code. Observe that when claiming, `VELO` is minted on demand. This is contrasted with `PegAirdrop` (`MerkleClaim` fork) which holds all the tokens and simply distributes them as users claim.

[EmissionScheduler.sol#L63-L80](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/EmissionScheduler.sol#L63-L80)

        function epochEmission(uint256 _pegSupply, uint256 _veSupply) external returns (uint256, uint256, uint256) {
            uint256 currentEpoch = _currentEpoch();

            if (currentEpoch == lastEpoch) {
                return (0, 0, 0);
            }

            lastEpoch = currentEpoch;

    @>      uint256 toEmit = (_pegSupply - _veSupply).mulDiv(2, EMISSION_PRECISION);

            lastEpochEmission = toEmit;

            uint256 rebase = _calculateRebase(toEmit, _pegSupply, _veSupply);
            uint256 teamEmission = _calculateTeamEmission(toEmit, rebase);

            return (toEmit, rebase, teamEmission);
        }

This difference is quite important as `epochEmission` is based on the float percentage of `supply`. As a result these tokens which are undistributed will be incorrectly counted against the floating token `supply`. This will lead to excess token emissions until the claim period is over governance is able to recover them.

**Lines of Code**

[EmissionScheduler.sol#L72](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/EmissionScheduler.sol#L72)

**Recommendation**

Consider removing the balance of `PegAirdrop` from the emission calculation or using an on-demand minting strategy similar to `VELO`.

**Remediation**

Fixed in [85be083](https://github.com/hyperstable/contracts/commit/85be083579af51f7c04b39233d8daab96bb40ff1). Distribution has been changed to on-demand minting.

### [M-03] Curve gauge rewards can be griefed

**Details**

[Gauge.sol#L123-L131](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/Gauge.sol#L123-L131)

        function claimFees() external lock {
    @>      require(msg.sender == IVotingEscrow(_ve).team(), "only team");

            if (staking == address(0)) {
                return;
            }

            ICurveGauge(staking).claimRewards(address(this), msg.sender);
        }

`Gauge#claimFees` attempts to lock down claims so that only the team can claim however if you look at the curve gauge code, the claim process is permissionless if `_receiver == address(0)`. When claimed with `_receiver == address(0)` it will send the rewards to the holder which in this case is the `gauge` contract itself.

[LiquidityGaugeV5.vy#L416-L426](https://github.com/curvefi/curve-dao-contracts/blob/567927551903f71ce5a73049e077be87111963cc/contracts/gauges/LiquidityGaugeV5.vy#L416-L426)

        def claim_rewards(_addr: address = msg.sender, _receiver: address = ZERO_ADDRESS):
            """
            @notice Claim available reward tokens for _addr
            @param _addr Address to claim for
            @param _receiver Address to transfer rewards to - if set to
                            ZERO_ADDRESS, uses the default reward receiver
                            for the caller
            """
            if _receiver != ZERO_ADDRESS:
    @>          assert _addr == msg.sender  # dev: cannot redirect when claiming for another user
            self._checkpoint_rewards(_addr, self.totalSupply, True, _receiver)

As a result the `gauge` rewards can be griefed and sent directly to the `gauge` contract instead of being sent to the team which will result in them being permanently unrecoverable.

**Lines of Code**

[Gauge.sol#L123-L131](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/governance/Gauge.sol#L123-L131)

**Recommendation**

LP should be staked via a proxy contract. This way all rewards can be sent to the team and there is never any chance they are mixed with user rewards or stranded.

**Remediation**

Fixed in [1aeb762](https://github.com/hyperstable/contracts/commit/1aeb762d777ebd59321713497dd5a6d17b2db01a). reward_receiver is set to team address direct all permissionless claims there instead of the gauge contract.

### [M-04] `AdaptiveIRM#_curve` multiples instead of dividing leading to "V" shaped curve instead of expected "L" shaped curve

**Details**

[AdaptiveIRM.sol#L110-L115](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/libraries/AdaptiveIRM.sol#L110-L115)

        function _curve(int256 _rateAtTarget, int256 err) private pure returns (int256) {
            // Non negative because 1 - 1/C >= 0, C - 1 >= 0.
    @>      int256 coeff = err < 0 ? INT_WAD - INT_WAD.sMulWad(CURVE_STEEPNESS) : CURVE_STEEPNESS - INT_WAD;
            // Non negative if _rateAtTarget >= 0 because if err < 0, coeff <= 1.
            return (coeff.sMulWad(err) + INT_WAD).sMulWad(int256(_rateAtTarget));
        }

Above is the `_curve` calculation for the forked `adaptiveIrm`. Below is the original `adaptiveIRM` contract from `Morpho`.

[AdaptiveCurveIrm.sol#L136-L141](https://github.com/morpho-org/morpho-blue-irm/blob/0e99e647c9bd6d3207f450144b6053cf807fa8c4/src/adaptive-curve-irm/AdaptiveCurveIrm.sol#L136-L141)

        function _curve(int256 _rateAtTarget, int256 err) private pure returns (int256) {
            // Non negative because 1 - 1/C >= 0, C - 1 >= 0.
    @>      int256 coeff = err < 0 ? WAD - WAD.wDivToZero(ConstantsLib.CURVE_STEEPNESS) : ConstantsLib.CURVE_STEEPNESS - WAD;
            // Non negative if _rateAtTarget >= 0 because if err < 0, coeff <= 1.
            return (coeff.wMulToZero(err) + WAD).wMulToZero(int256(_rateAtTarget));
        }

Notice that the `WAD.wDivToZero` has been erroneously replaced with `INT_WAD.sMulWad`. err is the distance between the current utilization and the target utilization. As a result of this change the interest rate curve will decrease as it's approaching from the left rather than increasing as intended. This will give a "V" shaped interest rate curve rather than an "L" shaped one.

**Lines of Code**

[AdaptiveIRM.sol#L110-L115](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/libraries/AdaptiveIRM.sol#L110-L115)

**Recommendation**

Change the mul to a div

**Remediation**

Fixed in [6cc2ad9](https://github.com/hyperstable/contracts/commit/6cc2ad9f1289195ece0e7c2d4cd7e2dc58e771db) as recommended.

### [M-05] `AdaptiveIRM` has been integrated incorrectly and will not work as expected

**Details**

[Morpho.sol#L483-L509](https://github.com/morpho-org/morpho-blue/blob/eb65770e12c4cdae5836c0027e35b9c82794c3eb/src/Morpho.sol#L483-L509)

        function _accrueInterest(MarketParams memory marketParams, Id id) internal {
            uint256 elapsed = block.timestamp - market[id].lastUpdate;
            if (elapsed == 0) return;

            if (marketParams.irm != address(0)) {
    @>          uint256 borrowRate = IIrm(marketParams.irm).borrowRate(marketParams, market[id]);
    @>          uint256 interest = market[id].totalBorrowAssets.wMulDown(borrowRate.wTaylorCompounded(elapsed));
                market[id].totalBorrowAssets += interest.toUint128();
                market[id].totalSupplyAssets += interest.toUint128();

                uint256 feeShares;
                if (market[id].fee != 0) {
                    uint256 feeAmount = interest.wMulDown(market[id].fee);
                    // The fee amount is subtracted from the total supply in this calculation to compensate for the fact
                    // that total supply is already increased by the full interest (including the fee amount).
                    feeShares =
                        feeAmount.toSharesDown(market[id].totalSupplyAssets - feeAmount, market[id].totalSupplyShares);
                    position[id][feeRecipient].supplyShares += feeShares;
                    market[id].totalSupplyShares += feeShares.toUint128();
                }

                emit EventsLib.AccrueInterest(id, borrowRate, interest, feeShares);
            }

            // Safe "unchecked" cast.
            market[id].lastUpdate = uint128(block.timestamp);
        }

First let's observe how the original `Morpho` markets integrate with the `AdaptiveIRM`. Notice that the average borrow rate is utilized directly BEFORE any state change has happened.

[AdaptiveIRM.sol#L94-L104](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/libraries/AdaptiveIRM.sol#L94-L104)

        } else {
            int256 linearAdaptation = ADJUSTMENT_SPEED.sMulWad(err) * (int256(block.timestamp) - s.lastUpdate[_vault]);

            if (linearAdaptation == 0) {
                avgRateAtTarget = startRateAtTarget;
                endRateAtTarget = startRateAtTarget;
            } else {
                endRateAtTarget = _newRateAtTarget(startRateAtTarget, linearAdaptation);
                int256 midRateAtTarget = _newRateAtTarget(startRateAtTarget, linearAdaptation / 2);
    @>          avgRateAtTarget = (startRateAtTarget + endRateAtTarget + 2 * midRateAtTarget) / 4;
            }

This is done because the `avgRateAtTarget` is a trapezoidal approximation between the start and end interest rates, which dynamically accounts for the changing interest rate.

[PositionManager.sol#L70-L87](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/core/PositionManager.sol#L70-L87)

        function deposit(address _toVault, uint256 _amountToDeposit) external {
            _isValidVault(_toVault);

            IVault vault = IVault(_toVault);
            IERC20 asset = IERC20(vault.asset());

            asset.transferFrom(msg.sender, address(this), _amountToDeposit);
            uint256 shares = vault.deposit(_amountToDeposit, address(this));

    @>      (, uint256 updatedVaultDebt) = _accruePositionDebt(vault, msg.sender);

            collateralShares[msg.sender][_toVault] += shares;
            uint256 updatedVaultCollateral = vaultCollateral[_toVault] += shares;

    @>      interestRateStrategy.updateVaultInterestRate(
                _toVault, updatedVaultCollateral.mulWad(vault.sharePrice()), updatedVaultDebt, vault.MCR()
            );
        }

Take `PositionManager#deposit` as a comparative example. We see that the position debt is accrued before the interest rate is updated.

[InterestRate.sol#L153-L171](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/libraries/InterestRate.sol#L153-L171)

        function calculateInterestIndex(InterestRateStorage storage s, address _vault)
            internal
            view
            returns (uint256, uint256)
        {
            uint256 interestIndex = s.activeInterestIndex[_vault];
            if (s.lastInterestIndexUpdate[_vault] == block.timestamp) {
                return (interestIndex, 0);
            }

            uint256 factor;
            if (s.interestRate[_vault] > 0) {
                uint256 timeDelta = block.timestamp - s.lastInterestIndexUpdate[_vault];
    @>          factor = timeDelta * s.interestRate[_vault];
                interestIndex += interestIndex.mulDivUp(factor, INTEREST_PRECISION);
            }

            return (interestIndex, factor);
        }

Notice in `InterestRate#calculateInterestIndex` that the cached interest rate is used. This is the stale rate based on the PREVIOUS average interest rate rather than the CURRENT. As a result an incorrect amount of interest will be charge to the users.

**Lines of Code**

[InterestRate.sol#L153-L171](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/libraries/InterestRate.sol#L153-L171)

**Recommendation**

There is no need to cache the interest rate for a vault as it should always be calculated dynamically from the IRM. The only thing that needs to be updated is `s.endRateAt` and `s.lastUpdate` as is done in `AdaptiveIRM#updateInterestRateAtTarget`.

**Remediation**

Structurally fixed in [fddfdec](https://github.com/hyperstable/contracts/commit/fddfdec369d12b00520f98d88310a177a8c87da6). Although debt and collateral are always 0 in changes, additional changes in next edition of contracts will utilize proper values.
