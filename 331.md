berndartmueller

medium

# Missing performance fee deduction when closing an `IchiVaultSpell` position

## Summary

Closing an `IchiVaultSpell` position and realizing potential profits denominated in `borrowToken` should be subject to a performance fee of `10%`. However, the `IchiVaultSpell` contract does not deduct any such fees when closing a position, resulting in missed potential profits for the protocol.

## Vulnerability Detail

The `Blueberry Parameters & Flows.pdf` document states that any profits made by a strategy should be subject to a performance fee of `10%`. However, when a position is closed in the `IchiVaultSpell` contract, the contract does not deduct the performance fee from the realized profits.

When closing an `IchiVaultSpell` position, the `IchiVaultSpell.withdrawInternal` function swaps the withdrawn tokens to the borrowed token, repays the borrowed token debt, and refunds the remaining `borrowToken` balance to the current `bank.EXECUTOR()` in [L328](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L328). The refunded `borrowToken` tokens represent the realized profits from the position and should be subject to a performance fee of `10%`.

Please note that a performance fee is deducted from farmed rewards in [L391](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L391).

## Impact

The Blueberry protocol misses out on potential performance fees realized when closing a `IchiVaultSpell` position.

## Code Snippet

[spell/IchiVaultSpell.sol#L328](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L328)

```solidity
File: contracts/spell/IchiVaultSpell.sol
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
318:
319:     // 5. Withdraw isolated collateral from Bank
320:     doWithdraw(collToken, amountShareWithdraw);
321:
322:     // 6. Repay
323:     doRepay(borrowToken, amountRepay);
324:
325:     _validateMaxLTV(strategyId);
326:
327:     // 7. Refund
328:     doRefund(borrowToken); // @audit-info There are no performance fees deducted from potential profits in `borrowToken`.
329:     doRefund(collToken);
330: }
```

## Tool used

Manual Review

## Recommendation

Consider using `doCutRewardsFee(borrowToken)` at the end of the `IchiVaultSpell.withdrawInternal` function to deduct performance fees from potential profits.
