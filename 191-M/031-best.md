cergyk

medium

# Reentrancy by lack of validation of borrowToken in withdrawInternal and depositInternal

## Summary
in functions such as `withdrawInternal`, `depositInternal`, there is a complete lack of validation of user input `borrowToken`. as long as amount is 0, there is no call made to the `bank` which would check that borrowToken is not in the whitelist.

## Vulnerability Detail
This lack of check can be used to make reentrancy calls directly into IchiVaultSpells (since BlueberryBank is in an executing state).

An example call which can reenter:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L141

## Impact
Since the malicious user can only access the bank in the context of his own position, he can only put his own position in a weird state. I did not find any critical impact this reentrancy could make on its own, but it would be better to remove it anyway.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Validate borrow token in all cases.