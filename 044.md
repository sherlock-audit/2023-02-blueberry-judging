Matin

medium

# Existence of a previously declared strategy is not checked

## Summary
Function ```addStrategy()``` inside contract ```IchiVaultSpell.sol``` doesn't check the existence of a previously declared strategy.
## Vulnerability Detail
The function ```addStrategy()```, which can be called only by the owner, creates a new pair of address and the maximum size of a position corresponding to a vault. It has some checks on input values to ensure those are not zero. However, there is not any vault-existence check before pushing the new strategy struct to the ```strategies``` array which is responsible to keep the above-mentioned parameters. Thus, the modifier ```existingStrategy()``` which checks only the length of the ```strategies``` array, may be bypassed.
Such a scenario would happen:
1 - the owner pushes some data to the ```strategies``` array initially.
2 - some duplicate and repetitive vaults may be added by mistake, thus the amount of the ```strategies``` array increases.
3 - other functions such as ```openPosition()``` which has the modifier ```existingStrategy()``` would not revert for wrong strategy Id.

## Impact
Code malfunction, wrong data return, and maybe loss of funds
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L77-L82
## Tool used

Manual Review

## Recommendation
Consider adding a mapping for addresses of vaults and precheck it before assigning new inputs for  ```strategies``` array