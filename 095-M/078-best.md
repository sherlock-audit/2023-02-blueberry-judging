rvierdiiev

medium

# Borrowers continue accruing interests when repay is paused

## Summary
Borrowers continue accruing interests when repay is paused
## Vulnerability Detail
BlueBerryBank.bankStatus variable can be set by owner to [restrict repaying of loans](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L233-L235).

When `repay` function is called it checks if it's [allowed to repay](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L747).
The problem is that when repay is paused, borrowers continue accruing interests for their loans and can't repay, so their total payment is increasing. This is not fair to them.
## Impact
Total debt of borrowers increases when repay is not allowed.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L747
## Tool used

Manual Review

## Recommendation
Users should not pay interests for the time when they were unable to repay debt.