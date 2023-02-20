sakshamguruji

medium

# Add Duplicate Strategy for A Vault

## Summary

The owner can add duplicate strategies for a vault as there are no checks to verify if a strategy exists for a vault

## Vulnerability Detail

The owner can add a duplicate strategy in the strategies array here  https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L80 and provide a lower/higher maxPosSize . The users might interact with the updates strategy and get unwanted returns.

## Impact

There would exist 2 (or more) strategies for the same vault but different maxPosSize , The users might interact with the updates strategy and get unwanted returns.


## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L77-L81

## Tool used

Manual Review

## Recommendation

Add a check to see if the strategy has already been added or not