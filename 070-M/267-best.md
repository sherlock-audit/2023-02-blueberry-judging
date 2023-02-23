rbserver

medium

# Users, who use hard and soft vaults that are previously deployed and have passed previous withdrawal fee window, are required to pay withdrawal fee again within restarted withdrawal fee window when calling `HardVault.withdraw` and `SoftVault.withdraw` functions of these previously deployed vaults after `ProtocolConfig.startVaultWithdrawFee` function is called for newly deployed hard and soft vaults

## Summary
After the `ProtocolConfig.startVaultWithdrawFee` function is called for the newly deployed hard and soft vaults, the users, who use the hard and soft vaults that are previously deployed and have passed the previous withdrawal fee window, are required to pay the withdrawal fee again within the restarted withdrawal fee window when calling the `HardVault.withdraw` and `SoftVault.withdraw` functions of these previously deployed vaults.

## Vulnerability Detail
When calling the `HardVault.withdraw` and `SoftVault.withdraw` functions, the withdrawal fees will be sent to the treasury when `block.timestamp < config.withdrawVaultFeeWindowStartTime() + config.withdrawVaultFeeWindow()` is true. After the withdrawal fee window is passed, calling the `HardVault.withdraw` and `SoftVault.withdraw` functions should not be charged with the withdrawal fees for these hard and soft vaults. However, when new vaults are deployed, the `ProtocolConfig.startVaultWithdrawFee` function might be called for charging the withdrawal fees within the withdrawal fee window for the new vaults. Yet, for these vaults that are previously deployed and have passed the previous withdrawal fee window already, the withdrawal fee window becomes effective again and calling the `HardVault.withdraw` and `SoftVault.withdraw` functions are charged with the withdrawal fees within the restarted withdrawal fee window for these previously deployed vaults.

## Impact
It is unfair to the users, who use the hard and soft vaults that are previously deployed and have passed the previous withdrawal fee window, because they were required to pay the withdrawal fee within the previous withdrawal fee window and are required to do so again within the restarted withdrawal fee window when calling the `HardVault.withdraw` and `SoftVault.withdraw` functions of these previously deployed vaults.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L43-L45
```solidity
    function startVaultWithdrawFee() external onlyOwner {
        withdrawVaultFeeWindowStartTime = block.timestamp;
    }
```

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/HardVault.sol#L91-L116
```solidity
    function withdraw(address token, uint256 shareAmount)
        external
        override
        nonReentrant
        returns (uint256 withdrawAmount)
    {
        if (shareAmount == 0) revert ZERO_AMOUNT();
        IERC20Upgradeable uToken = IERC20Upgradeable(token);
        _burn(msg.sender, uint256(uint160(token)), shareAmount);
        withdrawAmount = shareAmount;

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
When the hard or soft vault is initialized, the end time of the withdrawal fee window for this vault can be recorded in a state variable in this vault's contract. Such state variable can then be used in the respective `HardVault.withdraw` or `SoftVault.withdraw` function for determining whether the withdrawal fee should be charged or not, instead of using `config.withdrawVaultFeeWindowStartTime()` and `config.withdrawVaultFeeWindow()`.