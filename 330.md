berndartmueller

medium

# Closing an `IchiVaultSpell` position is susceptible to slippage when swapping tokens

## Summary

The token swap performed in the `IchiVaultSpell.withdrawInternal` function is vulnerable to slippage and can result in a reduction in the expected profits when closing an `IchiVaultSpell` position.

## Vulnerability Detail

The `withdrawInternal` function in the `IchiVaultSpell` contract redeems the Ichi Vault LP tokens and subsequently swaps the withdrawn tokens to the `borrowToken` token to repay the outstanding debt. The remaining `borrowToken` tokens are potential profits for the user and will be refunded later on.

The `sqrtPriceLimitX96` parameter for the `IUniswapV3Pool.swap` function is set to `UniV3WrappedLibMockup.MAX_SQRT_RATIO - 1` or `UniV3WrappedLibMockup.MIN_SQRT_RATIO + 1` depending on the `borrowToken` token. Using these constants is equal to using `sqrtPriceLimitX96 = 0` when utilizing the Uniswap V3 `SwapRouter` (see [SwapRouter.sol#L105-L107](https://github.com/Uniswap/v3-periphery/blob/6cce88e63e176af1ddb6cc56e029110289622317/contracts/SwapRouter.sol#L105-L107)).

Consequently, slippage can occur, causing users to receive fewer `borrowToken` tokens than expected and reducing their profits.

Please note that having the `sqrtPriceLimitX96` parameter provided by the user can result in a token swap that is only partially filled, and the remaining unswapped tokens are locked in the contract.

## Impact

A user closing a position can possibly receive fewer profits than expected due to slippage when swapping the withdrawn tokens to the borrowed token.

## Code Snippet

[spell/IchiVaultSpell.sol#L312-L314](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L312-L314)

```solidity
276: function withdrawInternal(
277:     uint256 strategyId,
278:     address collToken,
279:     address borrowToken,
280:     uint256 amountRepay,
281:     uint256 amountLpWithdraw,
282:     uint256 amountShareWithdraw
283: ) internal {
...      // [...]
299:
300:     // 4. Swap withdrawn tokens to initial deposit token
301:     bool isTokenA = vault.token0() == borrowToken;
302:     uint256 amountToSwap = IERC20(
303:         isTokenA ? vault.token1() : vault.token0()
304:     ).balanceOf(address(this));
305:     if (amountToSwap > 0) {
306:         swapPool = IUniswapV3Pool(vault.pool());
307:         swapPool.swap(
308:             address(this),
309:             // if withdraw token is Token0, then swap token1 -> token0 (false)
310:             !isTokenA,
311:             int256(amountToSwap),
312:             isTokenA
313:                 ? UniV3WrappedLibMockup.MAX_SQRT_RATIO - 1 // Token0 -> Token1
314:                 : UniV3WrappedLibMockup.MIN_SQRT_RATIO + 1, // Token1 -> Token0
315:             abi.encode(address(this))
316:         );
317:     }
...      // [...]
```

## Tool used

Manual Review

## Recommendation

Consider adding a user-controllable slippage protection parameter (e.g. `amountOutMinimum` - as used by [Uniswap V3 `SwapRouter.sol#L128`](https://github.com/Uniswap/v3-periphery/blob/6cce88e63e176af1ddb6cc56e029110289622317/contracts/SwapRouter.sol#L128)) to specify the minimum amount of `borrowToken` tokens to receive after the swap.
