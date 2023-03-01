ctf_sec

medium

# Missing deadline check allows pending transaction to be maliciously executed When performing Uniswap V3 Swap

## Summary

Missing deadline check allows pending transaction to be maliciously executed When performing Uniswap V3 Swap

## Vulnerability Detail

In the current implementation, when IChiVaultSpell.sol is called, the code below executes

```solidity
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
```

which calls withdrawInternal:

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
```

the code above performs a Uniswap V3 trade by directly trading on the Uniswap V3 Pool, which has no deadline check

https://github.com/Uniswap/v3-core/blob/05c10bf6d547d6121622ac51c457f93775e1df09/contracts/UniswapV3Pool.sol#L607

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

AMMs provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity. The most common solution is to include a deadline timestamp as a parameter (for example see [Uniswap V2](https://github.com/Uniswap/v2-periphery/blob/0335e8f7e1bd1e8d8329fd300aea2ef2f36dd19f/contracts/UniswapV2Router02.sol#L229) and [Uniswap V3](https://github.com/Uniswap/v3-periphery/blob/6cce88e63e176af1ddb6cc56e029110289622317/contracts/SwapRouter.sol#L119)). If such an option is not present, users can unknowingly perform bad trades:

- Alice wants to swap 100 tokens for 1 ETH and later sell the 1 ETH for 1000 DAI.
- The transaction is submitted to the mempool, however, Alice chose a transaction fee that is too low for miners to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer.

When the average gas fee dropped far enough for Alice’s transaction to become interesting again for miners to include it, her swap will be executed. In the meantime, the price of ETH could have drastically changed. She will still get 1 ETH but the DAI value of that output might be significantly lower. She has unknowingly performed a bad trade due to the pending transaction she forgot about.

An even worse way this issue can be maliciously exploited is through MEV:

- The swap transaction is still pending in the mempool. Average fees are still too high for miners to be interested in it. 
- The price of tokens has gone up significantly since the transaction was signed, meaning Alice would receive a lot more ETH when the swap is executed. But that also means that her maximum slippage value (sqrtPriceLimitX96 and minOut in terms of the IChiVaultSpell contracts) is outdated and would allow for significant slippage.
- A MEV bot detects the pending transaction. Since the outdated maximum slippage value now allows for high slippage, the bot sandwiches Alice, resulting in significant profit for the bot and significant loss for Alice.

## Impact

The slippage protection can be outdated when performing Uniswap Trade due to missing deadline check and user can receive suffer from trade loss when closing the position.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L275-L318

## Tool used

Manual Review

## Recommendation

We recommend the protocol use Uniswap V3 Router to perform the swap which has the deadline protection or consider add the deadline protection in the code.
