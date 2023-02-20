tsvetanovv

medium

# It is possible front-run grieving attack in `liquidate()`

## Summary
It is possible front-run grieving attack in `liquidate()`. An attacker can apply grieving attack by preventing users to liquidate a position.

## Vulnerability Detail
In [BlueBerryBank.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511) we have function `liquidate()`. 
This function liquidate a position and pay debt for its owner and take the collateral.
`liquidate()` is vulnerable to front-run grieving attack. 
- Suppose Alice (an honest user) calls the function to liquidate a position and pay his debt.
- Bob (a malicious user) notices Alice's transaction in the Mempool. So, Bob applies front-run attack and calls the function [liquidate()](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511) with `uint256 amountCall` more than Alice needed.
- The function invokes [repayInternal](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L760) function which perform repay action.
```solidity
523: (uint256 amountPaid, uint256 share) = repayInternal(
524:            positionId,
525:            debtToken,
526:            amountCall
527:        );
```
- It gets to the point where is check if what Alice paid is more than the old debt.
```solidity
776: if (paid > oldDebt) revert REPAY_EXCEEDS_DEBT(paid, oldDebt);
```

This is also the moment when you get a front-run grieving attack.

## Impact
Malicious user could prevent the user to liquidate his position.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L760

## Tool used

Manual Review

## Recommendation

All this can be avoided if you add following condition at the beginning of `liquidate()`.
If `amountCall` is more than needed, `amountCall` to be equal to the maximum you need.
