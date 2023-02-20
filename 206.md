rbserver

high

# When calling `BlueBerryBank.withdrawLend` function, it is possible that extra amount, which is the difference between withdrawn amount from soft vault and position's `underlyingAmount`, remains in `BlueBerryBank` contract without belonging to anyone

## Summary
When the `BlueBerryBank.withdrawLend` function is called, the withdrawn amount from the soft vault can be bigger than the position's `underlyingAmount`. Such extra amount would remain in the `BlueBerryBank` contract without belonging to anyone.

## Vulnerability Detail
When calling the `BlueBerryBank.withdrawLend` function, if `wAmount` is from the `bank.softVault`, the `SoftVault.withdraw` function would execute `cToken.redeem(shareAmount)`, and it is possible that such `wAmount` is bigger than `pos.underlyingAmount`. When this happens, `wAmount` is set to `pos.underlyingAmount` because the `BlueBerryBank.withdrawLend` function executes `wAmount = wAmount > pos.underlyingAmount ? pos.underlyingAmount : wAmount`. Then, the `wAmount` are distributed to the treasury and eventually to the user. However, the extra amount that is the difference between the `wAmount` from the `bank.softVault` and `pos.underlyingAmount` remains in the `BlueBerryBank` contract without belonging to anyone.

## Impact
As described in the Vulnerability Detail section, the extra amount that is the difference between the withdrawn amount from the soft vault and the position's `underlyingAmount` would remain in the `BlueBerryBank` contract without belonging to anyone. Thus, such extra amount is lost.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704
```solidity
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
        pos.underlyingAmount -= wAmount;
        bank.totalLend -= wAmount;

        wAmount = doCutWithdrawFee(token, wAmount);

        IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);
    }
```

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L94-L123
```solidity
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

## Tool used

Manual Review

## Recommendation
The `BlueBerryBank.withdrawLend` function can be updated to transfer the described extra amount to a trusted party for preventing the loss of such amount.