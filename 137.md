cducrest-brainbot

false

# Bank totalLend accounting wrong

## Summary

In `BlueBerryBank.sol`, the value of `bank.totalLend` is probably wrong after liquidations.

## Vulnerability Detail

The value of `bank.totalLend` should track the total amount lent to the bank, but liquidations do not update this value when they probably should. This value is not currently used anywhere in the code so it is hard to tell if this is definitely wrong or can be abused. However if spells are later added that rely on the correct calculation of this value for critical accounting, it will become a problem.

## Impact

Currently low impact as this value is unused by other parts of the code. If the protocol intends to add spells or other functionalities that rely on this value, the impact may be higher.

## Code Snippet

lend function increases bank.totalLend, pos.underlyingAmount, and deposits token to the vault :

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L620-L662

withdrawLend function decreases bank.totalLend, pos.underlyingAmount, and removes token from the vault:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704

liquidate function decreases pos.underlyingAmount, and transfer vault tokens to the liquidator (they no longer are held by the bank), but does not decrease bank.totalLend:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572

## Tool used

Manual Review

## Recommendation

Update this value correctly or remove it altogether.