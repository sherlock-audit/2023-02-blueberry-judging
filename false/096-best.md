SPYBOY

high

# Dos on `addBank()` because of validation of allBanks.lenght

## Summary
In the `BlueBerryBank.sol` contract there is a function `addBank()` to add a new bank using a different token. which adds tokens to the Dynamic array `allBanks`. It also checks the length of the array `allBanks` which is restricted to 256. 
POC:
1) Bob is the admin he starts adding new banks in the contract he has added 256 banks.
2) After some time some of the tokens fail (may be rug pulled / Hacked) . let's say 10 tokens and bob decides to blacklist those  10 tokens using the function  `whitelistTokens`.
3) Those Black listed tokens are still there in  `allBanks` array and in the function `addBanks()` there is validation to check the length of the array `allBanks` is 256. Hence now bob can't add new banks.
validation check in `addBank()`:
```solidity
if (allBanks.length >= 256) revert BANK_LIMIT();
```
## Vulnerability Detail

## Impact
After adding 256 banks using the function `addBank()` and blacklisting some tokens admin can't add a new banks because the length of `allBanks` is restricted to 256. and there is no option to remove blacklisted tokens from `allbanks` array.  which will lead to Dos on `addBanks()`

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L190-L217
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L169-L181

## Tool used

Manual Review

## Recommendation
Add functionality to remove tokens from `allbanks` array after blacklisting tokens