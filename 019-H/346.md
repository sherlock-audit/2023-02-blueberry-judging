koxuan

high

# amountLpWithdrawn not given back to user

## Summary
In `withdrawInternal`,user will specify an amount of lp to withdraw from Ichi. The rest will be removed for loan repayment. However, after repaying the loan, the `amountLpWithdrawn` is not being withdrawn from IchiBank and hence the amount will stay in `IchiVaultSpell` where the next withdrawal can steal from.

## Vulnerability Detail
Notice step 2, amtLPToRemove is based on the ichi vault liquidity balance that the position has minus the amountLpWithdraw. amtLPToRemove will be removed from IchiVault and after a swap a repayment of the loan will be done. Notice that no where in the code does the `amountLpWithdrawn` that stayed in the `IchiVaultSpell` gets withdrawn or sent back to user. 

```solidity
     function withdrawInternal(
        uint256 strategyId,
        address collToken,
        address borrowToken,
        uint256 amountRepay,
        uint256 amountLpWithdraw,
        uint256 amountShareWithdraw
    ) internal {
        Strategy memory strategy = strategies[strategyId];
        IICHIVault vault = IICHIVault(strategy.vault);
        uint256 positionId = bank.POSITION_ID();


        // 1. Compute repay amount if MAX_INT is supplied (max debt)
        if (amountRepay == type(uint256).max) {
            amountRepay = bank.borrowBalanceCurrent(positionId, borrowToken);
        }


        // 2. Calculate actual amount to remove
        uint256 amtLPToRemove = vault.balanceOf(address(this)) -
            amountLpWithdraw;


        // 3. Withdraw liquidity from ICHI vault
        vault.withdraw(amtLPToRemove, address(this));


        // 4. Swap withdrawn tokens to initial deposit token
        bool isTokenA = vault.token0() == borrowToken;
        uint256 amountToSwap = IERC20(
            isTokenA ? vault.token1() : vault.token0()
        ).balanceOf(address(this));
        if (amountToSwap > 0) {
            swapPool = IUniswapV3Pool(vault.pool());
            swapPool.swap(
                address(this),
                // if withdraw token is Token0, then swap token1 -> token0 (false)
                !isTokenA,
                int256(amountToSwap),
                isTokenA
                    ? UniV3WrappedLibMockup.MAX_SQRT_RATIO - 1 // Token0 -> Token1
                    : UniV3WrappedLibMockup.MIN_SQRT_RATIO + 1, // Token1 -> Token0
                abi.encode(address(this))
            );
        }


        // 5. Withdraw isolated collateral from Bank
        doWithdraw(collToken, amountShareWithdraw);


        // 6. Repay
        doRepay(borrowToken, amountRepay);


        _validateMaxLTV(strategyId);


        // 7. Refund
        doRefund(borrowToken);
        doRefund(collToken);
    }
```




## Impact
Loss of fund to user who specified `amountLpWithdrawn`.

## Code Snippet)
[IchiVaultSpell.sol#L276-L330](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L276-L330)

## Tool used

Manual Review

## Recommendation
Recommend sending back user the `amountLpWithdrawn`.
