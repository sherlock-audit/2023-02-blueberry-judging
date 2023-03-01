tsvetanovv

medium

# Missing deadline checks when perform a swap

## Summary
The protocol does not allow users to submit a deadline for their actions which execute swaps on Uniswap V3. This missing feature enables pending transactions to be maliciously executed at a later point.

## Vulnerability Detail
AMMs provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity. The most common solution is to include a deadline timestamp as a parameter (for example see [Uniswap V3](https://github.com/Uniswap/v3-periphery/blob/6cce88e63e176af1ddb6cc56e029110289622317/contracts/SwapRouter.sol#L119)).

## Impact
If deadline checks is not present, users can unknowingly perform bad trades:
- Alice wants to swap 100 tokens for 1 ETH and later sell the 1 ETH for 1000 DAI.
- The transaction is submitted to the mempool, however, Alice chose a transaction fee that is too low for miners to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer.
- When the average gas fee dropped far enough for Alice's transaction to become interesting again for miners to include it, her swap will be executed. In the meantime, the price of ETH could have drastically changed. She will still get 1 ETH but the DAI value of that output might be significantly lower. She has unknowingly performed a bad trade due to the pending transaction she forgot about.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L407
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

## Tool used

Manual Review

## Recommendation

Introduce a deadline parameter to all functions which potentially perform a swap on the user's behalf.