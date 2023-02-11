0x52

medium

# IchiVaultSpell#withdrawInternal withdraws the incorrect amount of shares from the IchiVault

## Summary

IchiVaultSpell#closePosition allows the user to specify `amountLpWithdraw` which is specifically stated to be `Amount of ICHI Vault LP to withdraw from ICHI Vault`. The implementation is exactly the opposite of this and in IchiVaultSpell#withdrawInternal this amount actually represents the amount NOT to withdraw from the IchiVault. This is problematic for other protocols or power users who wish to integrate with the contract as they will receive more or less LP than expected, causing serious issues with their internal accounting.

## Vulnerability Detail

    /**
     * @notice External function to withdraw assets from ICHI Vault
     * @param collToken Token address to withdraw (e.g USDC)
     * @param borrowToken Token address to withdraw (e.g USDC)
     * @param lpTakeAmt Amount of ICHI Vault LP token to take out from Bank
     * @param amountRepay Amount to repay the loan
     * @param amountLpWithdraw Amount of ICHI Vault LP to withdraw from ICHI Vault
     * @param amountShareWithdraw Amount of Isolated collateral to withdraw from Compound
     */
    function closePosition(
        uint256 strategyId,
        address collToken,
        address borrowToken,
        uint256 lpTakeAmt,
        uint256 amountRepay,
        uint256 amountLpWithdraw,
        uint256 amountShareWithdraw
    )
        external
        existingStrategy(strategyId)
        existingCollateral(strategyId, collToken)
    {
        // 1. Take out collateral
        doTakeCollateral(strategies[strategyId].vault, lpTakeAmt);

        withdrawInternal(
            strategyId,
            collToken,
            borrowToken,
            amountRepay,
            amountLpWithdraw,
            amountShareWithdraw
        );
    }

Above is IchiVaultSpell#closePosition which passes amountLpWithdraw through into withdrawInternal.

        // 2. Calculate actual amount to remove
        uint256 amtLPToRemove = vault.balanceOf(address(this)) -
            amountLpWithdraw;

        // 3. Withdraw liquidity from ICHI vault

        //@audit-info doesn't need approval for this
        vault.withdraw(amtLPToRemove, address(this));

`withdrawInternal` does NOT withdraw `amountLpWithdraw` from the `IchiVault`. Instead it withdraws all but `amountLpWithdraw`. This is extremely different than intended and can lead to significant issues to anyone that integrates with Blueberry.

## Impact

Other protocols or power users who wish to integrate with the contract as they will receive more or less LP than expected

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L276-L330

## Tool used

Manual Review

## Recommendation

Fix `withdrawInternal` so that it withdraws the proper amount of tokens or change `amountLpWithdraw` to reflect what it actually does