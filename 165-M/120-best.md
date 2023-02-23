Ruhum

high

# Positions won't be liquidatable at the correct threshold because of an accounting issue in `withdrawLend()`

## Summary
When a deposit is withdrawn from the SoftVault contract it takes a fee. When the user's position is modified that fee is not taken into account causing the position to be reduced by fewer tokens than it should. The liquidation threshold will be reached later than it should be for that position.

## Vulnerability Detail
1. Alice deposits 100 tokens and receives 100 shares
2. Alice burns 50 shares and receives 45 tokens (5 tokens are taken as a fee)
3. Alice's position is reduced by 45 tokens instead of 50

## Impact
Alice's position won't be liquidatable at the correct time. 

## Code Snippet
Both the HardVault and SoftVault can take a fee on withdrawal: [SoftVault](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L94-L123) & [HardVault](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/HardVault.sol#L91-L116)
```sol
    function withdraw(uint256 shareAmount)
        external
        override
        nonReentrant
        returns (uint256 withdrawAmount)
    {
        if (shareAmount == 0) revert ZERO_AMOUNT();

        _burn(msg.sender, shareAmount);

        uint256 uBalanceBefore = uToken.balanceOf(address(this));
        if (cToken.redeem(shareAmount) != 0) revert REDEEM_FAILED(shareAmount);
        uint256 uBalanceAfter = uToken.balanceOf(address(this));

        withdrawAmount = uBalanceAfter - uBalanceBefore;
        // Cut withdraw fee if it is in withdrawVaultFee Window (2 months)
        if (
            block.timestamp <
            config.withdrawVaultFeeWindowStartTime() +
                config.withdrawVaultFeeWindow()
        ) {
            uint256 fee = (withdrawAmount * config.withdrawVaultFee()) /
                DENOMINATOR;
            uToken.safeTransfer(config.treasury(), fee);
            withdrawAmount -= fee;
        }
        uToken.safeTransfer(msg.sender, withdrawAmount);

        emit Withdrawn(msg.sender, withdrawAmount, shareAmount);
    }
```

The fee is subtracted from the final return value `withdrawAmount`. That value is used to reduce the caller's position in `BlueBerryBank.withdrawLend()`: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L664-L704

```sol
    /**
     * @dev Withdraw isolated collateral tokens lent to bank. Must only be called from spell while under execution.
     * @param token Isolated collateral token address
     * @param shareAmount The amount of vaule share token to withdraw.
     */
    function withdrawLend(address token, uint256 shareAmount)
        external
        override
        inExec
        poke(token)
    {
        Position storage pos = positions[POSITION_ID];
        Bank storage bank = banks[token];
        if (token != pos.underlyingToken) revert INVALID_UTOKEN(token);
        if (shareAmount == type(uint256).max) {
            shareAmount = pos.underlyingVaultShare;
        }

        uint256 wAmount;
        if (address(ISoftVault(bank.softVault).uToken()) == token) {
            ISoftVault(bank.softVault).approve(
                bank.softVault,
                type(uint256).max
            );
            wAmount = ISoftVault(bank.softVault).withdraw(shareAmount);
        } else {
            wAmount = IHardVault(bank.hardVault).withdraw(token, shareAmount);
        }

        wAmount = wAmount > pos.underlyingAmount
            ? pos.underlyingAmount
            : wAmount;

        pos.underlyingVaultShare -= shareAmount;
        // @audit wAmount is actual withdrawal - fees, e.g. 50 - 5 = 45 
        pos.underlyingAmount -= wAmount;
        bank.totalLend -= wAmount;

        wAmount = doCutWithdrawFee(token, wAmount);

        IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);
    }
```

The position's `underlyingAmount` value is used to determine its risk: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495

```sol
    function getPositionRisk(uint256 positionId)
        public
        view
        returns (uint256 risk)
    {
        Position storage pos = positions[positionId];
        uint256 pv = getPositionValue(positionId);
        uint256 ov = getDebtValue(positionId);
        uint256 cv = oracle.getUnderlyingValue(
            pos.underlyingToken,
            pos.underlyingAmount
        );

        if (cv == 0) risk = 0;
        else if (pv >= ov) risk = 0;
        else {
            risk = ((ov - pv) * DENOMINATOR) / cv;
        }
    }

```

Because `cv` is a larger value than it should be, `risk` will be a lower number. It won't reach the liquidation threshold because the internal accounting is broken.

## Tool used

Manual Review

## Recommendation
The vault should return the full amount and the fee should be distributed through the BlueBerryBank contract.
