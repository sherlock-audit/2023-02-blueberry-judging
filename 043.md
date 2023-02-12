Saeedalipoor01988

medium

# USDT is not supported because of approval mechanism

## Summary
 USDT is not supported because of approval mechanism

## Vulnerability Detail
When using the approval mechanism in USDT, the approval must be set to 0 before it is updated. USDT has a big market cap and is one of the must popular stable coins, in the future if you decide to add this token, you can't manage processes with this token.

## Impact
Can't lend USDT, the most popular stablecoin, as as the approval will revert.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L647
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L652

https://github.com/code-423n4/2022-05-rubicon-findings/issues/100
## Tool used

Manual Review

## Recommendation
Set the allowance to 0 before setting it to the new value.