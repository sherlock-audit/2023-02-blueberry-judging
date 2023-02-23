Jeiwan

high

# `getDebtValue` doesn't accrue borrow interest, breaking integrations and allowing more than maximal LTV

## Summary
The `getDebtValue` function calculates the debt of a position without accruing position's borrow interest beforehand. This affects two functions:
1. the `isLiquidatable` function will return `false` for a position that can be liquidated;
1. the `reducePosition` will allow the user to borrow more than is allowed by the maximal LTV.
## Vulnerability Detail
The [getDebtValue](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L451) function calculates the debt of a position. To do that the function check the [share of the position in the debt of every bank](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L466-L468). Bank's debt is the amount of tokens that was borrowed + the borrow interest that was accumulated over time. Since borrowing is integrated with Compound, it's required to synchronized bank's total debt value with that of the related cToken–this is done in the [accrue](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L254-L256) function. However, bank's debt is not synchronized in the `getDebtValue` function: the borrow interest accrued since the last `accrue` call won't be counted; bank's debt will be lower than the debt of the cToken. As a result, the functions that use `getDebtValue` will be impacted:
1. [getPositionRisk](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477) will return a healthier risk of the position, failing to detect positions that are already below the healthy threshold;
1. [isLiquidatable](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L497), which depends on `getPositionRisk`, will fail to detect liquidable positions, due to not taking into account the increased debt;
1. [reducePosition](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266) will let users to remove a portion of their collateral pass the [maximal LTV check](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L273): users, basically, will be allowed to borrow more funds than allowed by the strategy.
## Impact
1. the `isLiquidatable` function will return `false` for a position that can be liquidated;
1. the `reducePosition` will allow the user to borrow more than is allowed by the maximal LTV.

Overall, bad debt may accumulate for the protocol due to undercounted debt.
## Code Snippet
[BlueBerryBank.sol#L451](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L451)
## Tool used
Manual Review
## Recommendation
In the `getDebtValue` function, consider accruing borrow interest for all banks in which the position has debts.