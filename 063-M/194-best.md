8olidity

medium

# missing approval

## Summary
missing approval
## Vulnerability Detail
In the `ichivaultSpell` contract `withdrawInternal()`, `vault.pool` will be called for `swap()` operation, but there is no check whether the current contract is approved to authorize the operation.
```solidity
 if (amountToSwap > 0) {
      swapPool = IUniswapV3Pool(vault.pool()); //@audit-info missing approval  
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
```
## Impact
missing approval
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L306-L307
## Tool used

Manual Review

## Recommendation
Add approve  to the pool address