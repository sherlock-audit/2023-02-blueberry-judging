Ch_301

high

# Malicious users will lock a big amount of funds on BANK

## Summary

## Vulnerability Detail
any user can invoke [reducePosition()](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266-L274) even if it a newly opened position with no money on it.
Just set this `collAmount` param to any number and all the tokens will lock on **BlueBerryBank.sol** 
[Because of this logic](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L693-L695)

```solidity
        wAmount = wAmount > pos.underlyingAmount
            ? pos.underlyingAmount
            : wAmount;
```
the rest of the tokens will stay there
 
## Impact
Malicious users could `withdraw()` all the tokens on **SoftVault** and lock it on **BlueBerryBank.sol** 

## Code Snippet

## Tool used

Manual Review

## Recommendation
Add more checks here
```solidity
        wAmount = wAmount > pos.underlyingAmount
            ? pos.underlyingAmount
            : wAmount;
```
