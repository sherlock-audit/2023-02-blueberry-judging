Jeiwan

high

# Liquidations are enabled when repayments are disabled, causing borrowers to lose funds without a chance to repay

## Summary
Debt repaying can be temporary disabled by the admin of `BlueBerryBank`, however liquidations are not disabled during this period. As a result, users' positions can accumulate more borrow interest, go above the liquidation threshold, and be liquidated, while users aren't able to repay the debts.
## Vulnerability Detail
The owner of [BlueBerryBank](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L22) can disable different functions of the contract, [including repayments](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L233-L235). However, while [repayments are disabled](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L747) liquidations are still [allowed](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L516-L517). As a result, when repayments are disabled, liquidator can liquidate any position, and borrowers won't be able to protect against that by repaying their debts. Thus, borrowers will be forced to lose their collateral.
## Impact
Positions will be forced to liquidations while their owners won't be able to repay debts to avoid liquidations.
## Code Snippet
[BlueBerryBank.sol#L740](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L740)
## Tool used
Manual Review
## Recommendation
Consider disallowing liquidations when repayments are disabled. Alternatively, consider never disallowing repayments so that users could maintain their positions in a healthy risk range anytime.