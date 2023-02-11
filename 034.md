0x52

high

# IchiVaultSpell#closePosition will leave LP tokens in the contract if amountLpWithdraw != 0

## Summary

At the end of withdrawInternal it only returns collateral and borrow token. All LP that is withdrawn from the bank but not withdrawn from the IchiVault will be lost.

## Vulnerability Detail

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

IchiVaultSpell#closePosition allows the user to specify both how much LP to withdraw from the bank `lpTakeAmt` and how much of that LP to then withdraw from the IchiVault. The doTakeCollateral subcall removes the LP from the bank and transfers it to IchiVaultSpell.

        uint256 amtLPToRemove = vault.balanceOf(address(this)) -
            amountLpWithdraw;

        // 3. Withdraw liquidity from ICHI vault
        vault.withdraw(amtLPToRemove, address(this));

        // 4. Swap withdrawn tokens to initial deposit token

        ...

        // 5. Withdraw isolated collateral from Bank
        doWithdraw(collToken, amountShareWithdraw);

        // 6. Repay
        doRepay(borrowToken, amountRepay);

        _validateMaxLTV(strategyId);

        // 7. Refund
        doRefund(borrowToken);
        doRefund(collToken);

Inside `withdrawInternal`, `amtLPToRemove` is withdrawn from the IchiVault. The issue is that it never transfers the rest of the LP to the user. This will cause large loss of funds for the user as their LP will be unrecoverable.

## Impact

User will lose funds when closing a position and amountLpWithdraw != 0

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L276-L330

## Tool used

Manual Review

## Recommendation

Transfer the rest of the LP to user at the end of IchiVaultSpell#withdrawInternal:

        doRefund(borrowToken);
        doRefund(collToken);
    +   doRefund(vault);