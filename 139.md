cducrest-brainbot

medium

# takeCollateral does not trigger interest accrual

## Summary

In `BlueBerryBank.sol` the `takeCollateral` function does not trigger interest accrual with `poke(token)` like `borrow`, `repay`, `lend`, or `withdrawLend` do.  This could lead to wrong calculation of position's debt and whether the position is liquidatable.

## Vulnerability Detail

This means that if a user executes only `takeCollateral` on its position, the value of `bank.totalDebt` for the position's debt token will be lower than what it should be. This results in the debt value of the position being lower than what it should be and a position seen as not liquidatable while it should be liquidatable. 

Whether a position is liquidatable or not is checked at the end of the `execute` function, the execution should revert if the position is liquidatable.

## Impact

A user may be allowed to `takeCollateral` out of its position while if interests on the debt were accrued, they would not be able to.

Currently, in the only available spell `IchiVaultSpell` there is no way to execute `takeCollateral` without executing other functions which will call `poke(token)` and accrue the interests correctly. However, it is easily imaginable that other spells later added to the protocol will allow to call `takeCollateral` independently, as is suggested by `BasicSpell.sol` function `doTakeCollateral()`. This is why I suggest a severity of medium.

## Code Snippet

`takeCollateral` does no trigger interest accrual on the bank's debt:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L820-L847

bank.totalDebt is used to calculate a position's debt: 

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L451-L475

The position's debt is used to calculate the risk:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495

The risk is used to calculate whether a debt is liquidatable:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L497-L505

## Tool used

Manual Review

## Recommendation

Review how token interests are triggered. Probably need to accrue interests on debt token of a position at the beginning of `execute`.