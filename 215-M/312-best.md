tives

medium

# UniV3 sandwiching attacks are possible in withdrawal functions

## Summary

Sandwiching attacks possible in withdrawing functions

## Vulnerability Detail

Protocol uses MAX/MIN price ratios for UniV3 swaps. According to UniV3 documentation, these ratios define the price after the swap.

```solidity
/// @param sqrtPriceLimitX96 The Q64.96 sqrt price limit. If zero for one, the price cannot be less than this
/// value after the swap. If one for zero, the price cannot be greater than this value after the swap
```

This indicates that user will accept any price for her swap. Adversaries could sandwich these swaps for profit from the user’s withdrawal amount.

## Impact

User loses withdrawal amount due to no price control.

## Code Snippet

```solidity
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
```
[link](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol/#L307)
## Tool used

Manual Review

## Recommendation

You could calculate the SQRT_RATIO according to oracle prices. And deviate from it by some percentage amount.