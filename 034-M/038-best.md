koxuan

medium

# withdrawVaultFeeWindowStartTime is too rigid and does not serve its purpose

## Summary
 Withdrawal fee is placed on user if he withdraws early from Vault. However, due to how the withdrawVaultFeeWindowStartTime is implemented, users might pay fees when he should not and not pay fees when he should have.

## Vulnerability Detail
When withdrawing from vault, block.timstamp is checked against the `config.withdrawVaultFeeWindowStartTime() + config.withdrawVaultFeeWindow ` to determine whether a withdrawal fee should be imposed. Default `withdrawVaultFeeWindow` is 60 days. 
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
}

```

In `ProtocolConfig`, `startVaultWithdrawFee` is used to set the `withdrawVaultFeeWindowStartTime` to current time. If user deposits after the `withdrawVaultFeeWindow`, he will be able to withdraw without penalty regardless of how long he has deposited. Owner also could not reset the `withdrawVaultFeeWindowStartTime` as that will punish earlier depositors who have deposited more than the `withdrawVaultFeeWindow`.

```solidity
    function startVaultWithdrawFee() external onlyOwner {
        withdrawVaultFeeWindowStartTime = block.timestamp;
    }

```



## Impact
Basically, the vault is disincentivizing users from depositing in the vault during the withdrawVaultFeeWindow and incentivising people to do it after the window, and also owner cannot call `startWithdrawVaultFeeWindowStartTime` again to reset the `withdrawVaultFeeWindowStartTime` to current block.timestamp as that will punish earlier depositors who have deposited more than the `withdrawVaultFeeWindow`.

## Code Snippet 
[SoftVault.sol#L94-L124](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L94-L124)
[ProtocolConfig.sol#L43-L45](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L43-L45)

## Tool used

Manual Review

## Recommendation

Recommend keeping a deposit snapshot for the time and amount of deposit for every deposit made so that the protocol can track whether deposit is within `withdrawVaultFeeWindow`. 
