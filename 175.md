koxuan

medium

# lack of slippage control can cause uniswap swap to be susceptible to sandwich attacks

## Summary
When closing a position in IchiVaultSpell, withdrawInternal is called which allows user to swap borrowed tokens into initial deposit tokens. The slippage control is set to the maximum and minimum of the allowed max and min sqrt ratio, which is equivalent to setting 0 for sqrtPriceLimitX96 in uniswap router. With no slippage control, swaps done by user are susceptible to sandwich attacks.

## Vulnerability Detail
In step 4 of `withdrawInternal`, notice that the sqrt ratio price limit is set to either maximum or minimum depending on the direction of the swap. This is equivalent to setting 0 for sqrtPriceLimitX96 in uniswap router. With no slippage control, swaps done by user are susceptible to sandwich attacks. Note that there is no expected minimum out check done and hence there is no mechanism to control the slippage.
 

```solidity
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
```

## Impact
User swaps are susceptible to sandwich attacks.

## Code Snippet
[IchiVaultSpell.sol#L300-L317](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L300-L317)


## Tool used

Manual Review

## Recommendation

Recommend following uniswap's way of implementing the expected minimum amount out check as sqrtPriceLimitX96 as a slippage control is hard to set by user, or a twap oracle for sqrtPriceLimitX96 to reference. 
