cergyk

medium

# An attacker can block a bank by calling `repayOnBehalfOf` directly on cToken.

## Summary
An attacker can block a bank by calling `repayOnBehalfOf` directly on cToken.

## Vulnerability Detail
The attacker has to repay all the debt, setting bank.totalDebt to 0, and keep some shares of his own.

In that case it is impossible to borrow more funds since:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L720-L723
will revert with division by 0

Also impossible to repay to decrease totalShares to 0, since only attacker can decide repay his own share of debt (by calling repay with amount=0).

The attacker cannot be liquidated since his share of debt is 0.

## Impact
An attacker can cause grieving by blocking the bank of a token.

## Code Snippet

## Tool used

Manual Review

## Recommendation
return amount shares on borrow if totalDebt == 0 as well, instead of reverting.
Or disallowing repayOnBehalfOf in the comptroller.