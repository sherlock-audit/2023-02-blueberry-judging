carrot

medium

# `reducePosition` in IchiVaultSpell checks max LTV against stale debt values

## Summary
The function `reducePosition` in the `IchiVaultSpell` contract withdraws the underlying token, and refunds it. It then checks if the max LTV ratio is maintained. It however does not poke the borrow token account, and thus uses stale values of the debt.
## Vulnerability Detail
The `poke` function which calls the `accrue` function in the Bank contract is responsible for updating the debt values, and thus accounting for the interest accrued up to that instant of time.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L251-L257

This updates the bank's totalDebt, which in turn reflects the debt of the user through the share ratio. This function is called to update the debt numbers in all spell functions except one.

The `reducePosition` function reduces the underlying amount, and thus can hit max LTV cap easily. However, as can be seen below, `_validateMaxLTV` function uses `bank.getDebtValue`
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L101-L113
Which in turn uses the `bank.totalDebt` value without updating it.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L466-L467

Thus it doesn't account for the interest accroed by the debt since the last poke of the debt token, and can return a stale value of LTV, allowing user to withdraw more underlying than intended. The maxLTV can get violated in an infrequently used account that accrues debt over time, but this oversight lets users exit a function call leaving the contract in an unintended state which violates some of the assumptions.
## Impact
The user violates the maxLTV while actively calling a function
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L101-L113
## Tool used

Manual Review

## Recommendation
Call `accrue` on the borrow token before calculating max LTV