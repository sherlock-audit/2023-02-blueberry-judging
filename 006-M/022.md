cergyk

high

# High slippage tolerance when swapping on uniswapV3 can lead to frontrunning

## Summary
High slippage tolerance when swapping on uniswapV3 can lead to frontrunning and loss of funds for the user.

## Vulnerability Detail

the swap is defined here:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L306-L315

we see that the slippage control:
```solidity
isTokenA
    ? UniV3WrappedLibMockup.MAX_SQRT_RATIO - 1 // Token0 -> Token1
        : UniV3WrappedLibMockup.MIN_SQRT_RATIO + 1, // Token1 -> Token0
```
is set to the maximum/minimum.

## Impact
The user may lose part (half?) of their funds on slippage.

## Code Snippet

## Tool used

Manual Review

## Recommendation
use an oracle to define appropriate slippage control