cergyk

high

# Solvency checks do not accrue debt on all tokens

## Summary
Solvency check on a user position are necessary to ensure that he doesn't put protocol at risk

## Vulnerability Detail
Debt on compound style tokens is not accrued when evaluating global position risk for a user, making it possible to take borrows on other markets past health threshold.

`getPositionRisk`:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495

`getDebtValue`:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L451-L475
is based on unaccrued bank.totalDebt value.

## Impact
An unsuspecting user can make their position outright liquidatable after a borrow (isLiquidatable returns false before accrual, but true after accrual, so a call to `liquidate` is successful).

## Code Snippet

## Tool used

Manual Review

## Recommendation
Call accrue on all tokens when calling execute on `BlueberryBank`