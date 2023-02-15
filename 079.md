mert_eren

high

# A whale can make grieving attack to strategies that use same vault by transfer vault token to spell.

## Summary
A whale can make grieving attack to strategies that use same vault by transfer vault token to spell.   
## Vulnerability Detail
In ichiVaultSpell's depositInternal function it check total vault token in spell contract. If totalVaultToken*vaultPrice exceed strategies maxPosition size than reverts. So a whale can take large amount of vault token and transfer to the spell contract and can make a dos of openposition for strategies which use this specific vault.
## Impact
https://imgur.com/a/6C7Gagw
This test work in ichiVaultSpellTest.ts
As seen normal user can't openPosition and take EXCEED_MAX_POS warning even if it shouldn't.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L153-L157
## Tool used

Manual Review

## Recommendation
Instead of using balanceOf function can be recorded within the contract.