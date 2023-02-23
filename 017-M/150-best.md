cducrest-brainbot

medium

# Blacklisted users cannot close their position

## Summary

If a user owns a position for which the collateral or borrow token has a blacklist (e.g. USDC) they are at risk of being blacklisted and not being able to close the position at any point in the future.

## Vulnerability Detail

Among other things, `IchiVaultSpell.sol`'s functions `reducePosition` and `withdrawInternal` (called when closing a position) will call `doRefund(token)` which attempts to send the token back to the executor (i.e. the user closing its position).

If the token blacklisted the owner of the position, the transfer of token will revert and the position won't be able to be closed. 

There needs to be a function to transfer ownership of a position so that it can be closed without the transfer reverting.

## Impact

Ghost positions with outstanding debt may appear from users being blacklisted by collateral / borrow token.

## Code Snippet

withdrawInternal in `IchiVaultSpell` calls doRefund:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L328-L329

reducePosition calls deRefund: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L272

doRefund transfers token to executor:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L56-L61

executor defined as current position owner:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L116-L122

## Tool used

Manual Review

## Recommendation

Add method to transfer ownership of a position.
