Bauer

high

# Missing deadline checks allow pending transactions to be mailciously executed

## Summary
The ```BlueBerryBank``` contract does not allow users to submit a deadline for there actions. In ```closePosition()``` function call  there is a swap on Uniswap V3. This missing check enable pending transactions to be mailciously executed at a later point.

## Vulnerability Detail
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
The ```closePosition()``` is used to withdraw assets from ICHI Vault. The protocol is first to Withdraw liquidity from ICHI vault, then swap withdrawn tokens to initial deposit token, next withdraw collateral from Bank, then pay off debt, and finally refund. The swap operations is performed on UniswapV3. AMMs provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity(for example see [Uniswap V2](https://github.com/Uniswap/v2-periphery/blob/0335e8f7e1bd1e8d8329fd300aea2ef2f36dd19f/contracts/UniswapV2Router02.sol#L229) and [Uniswap V3](https://github.com/Uniswap/v3-periphery/blob/6cce88e63e176af1ddb6cc56e029110289622317/contracts/SwapRouter.sol#L119)). The most common solution is to include a deadline timestamp as a parameter.If such an option is not present ,users can unkowingly perform bad trades:

1.User want to close his postion, in the ```withdrawInternal()``` function call the protocol swap token0 to token1 on Uniswap v3 if amountToSwap > 0.
2.The transaction is submitted to the mempool, however, User chose a transaction fee that is too low for miners to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer.
3.When the average gas fee dropped far enough for user's transaction to become interesting again for miners to include it, his swap will be executed. In the meantime, the price of token0 could have drastically changed. The output token might be significantly lower. He has unknowingly performed a bad trade due to the pending transaction she forgot about.

An even worse way this issue can be maliciously exploited is through MEV:

The transaction is still pending in the mempool. Average fees are still too high for miners to be interested in it. The price of token0 has gone up significantly since the transaction was signed, meaning the protocol would receive a lot more token1 when the swap is executed. But that also means that her maximum slippage value  is outdated and would allow for significant slippage.
A MEV bot detects the pending transaction. Since the outdated maximum slippage value now allows for high slippage, the bot sandwiches the user, resulting in significant profit for the bot and significant loss for the user.


## Impact
See above

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L305-L317

## Tool used

Manual Review

## Recommendation
Introduce a deadline parameter to all ```execute()```.


