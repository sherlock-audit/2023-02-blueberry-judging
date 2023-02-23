Dug

medium

# Opening a position can be blocked

## Summary

A griefer can prevent users from successfully calling `openPosition`/`openPositionFarm` by inflating `curPosSize`.

## Vulnerability Detail

When a position is opened using the `IchiVaultSpell`, the `curPosSize` is checked against `strategy.maxPositionSize` and the transaction will revert if it is larger.

It is possible for a griefer to front-run transactions attempting to open a position. The griefer could transfer `borrowToken` or `vault` LP tokens directly to the `IchiVaultSpell` contract. These additional funds will be included in the `curPosSize` for the next position opened using that strategy.

## Impact

A griefer can send enough funds to disable new positions from being opened by making the `curPosSize` too large. A flashbot's bundle could potentially be used to enable the griefer to ensure that they get their funds back.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L122-L157

## Tool used

Manual Review

## Recommendation

Use the specific `borrowAmount` when adding liquidity and calculate the before/after balance of the LP token when making the `vault` deposit.
