cergyk

high

# Vault shares can be left in spell during withdrawInternal

## Summary
Vault shares can be withdrawn when using `withdrawInternal` in IchiVaultSpell.sol, but when done, if `amountLpWithdraw != vault.balanceOf(address(this))`, some vault balance stays in IchiVaultSpell, and can be swept by anybody by calling `reducePosition` with `collAmount == 0` and `collToken==vault`

## Vulnerability Detail
Only `vault.balanceOf(address(this)) - amountLpWithdraw` is withdrawn from vault. The rest of the balance stays in Spell.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L294-L298

## Impact
Users lose some of their funds.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Refund the remaining balance to the user, or put it back as collateral in the bank