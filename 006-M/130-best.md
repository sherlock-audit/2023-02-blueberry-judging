obront

high

# Withdrawals from IchiVaultSpell have no slippage protection so can be frontrun, stealing all user funds

## Summary

When a user withdraws their position through the `IchiVaultSpell`, part of the unwinding process is to trade one of the released tokens for the other, so the borrow can be returned. This trade is done on Uniswap V3. The parameters are set in such a way that there is no slippage protection, so any MEV bot could see this transaction, aggressively sandwich attack it, and steal the majority of the user's funds.

## Vulnerability Detail

Users who have used the `IchiVaultSpell` to take positions in Ichi will eventually choose to withdraw their funds. They can do this by calling `closePosition()` or `closePositionFarm()`, both of which call to `withdrawInternal()`, which follows loosely the following logic:
- sends the LP tokens back to the Ichi vault for the two underlying tokens (one of which was what was borrowed)
- swaps the non-borrowed token for the borrowed token on UniV3, to ensure we will be able to pay the loan back
- withdraw our underlying token from the Compound fork
- repay the borrow token loan to the Compound fork
- validate that we are still under the maxLTV for our strategy
- send the funds (borrow token and underlying token) back to the user

The issue exists in the swap, where Uniswap is called with the following function:
```solidity
if (amountToSwap > 0) {
    swapPool = IUniswapV3Pool(vault.pool());
    swapPool.swap(
        address(this),
        !isTokenA,
        int256(amountToSwap),
        isTokenA
            ? UniV3WrappedLibMockup.MAX_SQRT_RATIO - 1 
            : UniV3WrappedLibMockup.MIN_SQRT_RATIO + 1, 
        abi.encode(address(this))
    );
}
```
The 4th variable is called `sqrtPriceLimitX96` and it represents the square root of the lowest or highest price that you are willing to perform the trade at. In this case, we've hardcoded in that we are willing to take the worst possible rate (highest price in the event we are trading 1 => 0; lowest price in the event we are trading 0 => 1). 

The `IchiVaultSpell.sol#uniswapV3SwapCallback()` function doesn't enforce any additional checks. It simply sends whatever delta is requested directly to Uniswap.
```solidity
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata data
) external override {
    if (msg.sender != address(swapPool)) revert NOT_FROM_UNIV3(msg.sender);
    address payer = abi.decode(data, (address));

    if (amount0Delta > 0) {
        if (payer == address(this)) {
            IERC20Upgradeable(swapPool.token0()).safeTransfer(
                msg.sender,
                uint256(amount0Delta)
            );
        } else {
            IERC20Upgradeable(swapPool.token0()).safeTransferFrom(
                payer,
                msg.sender,
                uint256(amount0Delta)
            );
        }
    } else if (amount1Delta > 0) {
        if (payer == address(this)) {
            IERC20Upgradeable(swapPool.token1()).safeTransfer(
                msg.sender,
                uint256(amount1Delta)
            );
        } else {
            IERC20Upgradeable(swapPool.token1()).safeTransferFrom(
                payer,
                msg.sender,
                uint256(amount1Delta)
            );
        }
    }
}
```
While it is true that there is an `amountRepay` parameter that is inputted by the user, it is not sufficient to protect users. Many users will want to make only a small repayment (or no repayment) while unwinding their position, and thus this variable will only act as slippage protection in the cases where users intend to repay all of their returned funds.

With this knowledge, a malicious MEV bot could watch for these transactions in the mempool. When it sees such a transaction, it could perform a "sandwich attack", trading massively in the same direction as the trade in advance of it to push the price out of whack, and then trading back after us, so that they end up pocketing a profit at our expense.

Because many of the ICHI token pairs have small amounts of liquidity (for example, ICHI-WBTC has under $350k), such an attack could feasible take the majority of the funds, leaving the user with close to nothing. See more details on liquidity here: https://info.uniswap.org/#/tokens/0x111111517e4929d3dcbdfa7cce55d30d4b6bc4d6

## Impact

Users withdrawing their funds through the `IchiVaultSpell` who do not plan to repay all of the tokens returned from Uniswap could be sandwich attacked, losing their funds by receiving very little of their borrowed token back from the swap.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L300-L317

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L407-L442

## Tool used

Manual Review

## Recommendation

Have the user input a slippage parameter to ensure that the amount of borrowed token they receive back from Uniswap is in line with what they expect. 

Alternatively, use the existing oracle system to estimate a fair price and use that value in the `swap()` call.