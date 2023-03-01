obront

high

# Users can be liquidated prematurely because calculation understates value of underlying position

## Summary

When the value of the underlying asset is calculated in `getPositionRisk()`, it uses the `underlyingAmount`, which is the amount of tokens initially deposited, without any adjustment for the interest earned. This can result in users being liquidated early, because the system undervalues their assets.

## Vulnerability Detail

A position is considered liquidatable if it meets the following criteria: 

```solidity
((borrowsValue - collateralValue) / underlyingValue) >= underlyingLiqThreshold
```
The value of the underlying tokens is a major factor in this calculation. However, the calculation of the underlying value is performed with the following function call:
```solidity
uint256 cv = oracle.getUnderlyingValue(
    pos.underlyingToken,
    pos.underlyingAmount
);
```
If we trace it back, we can see that `pos.underlyingAmount` is set when `lend()` is called (ie when underlying assets are deposited). This is the only place in the code where this value is moved upward, and it is only increased by the amount deposited. It is never moved up to account for the interest payments made on the deposit, which can materially change the value.

## Impact

Users can be liquidated prematurely because the value of their underlying assets are calculated incorrectly.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L485-L488

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L182-L189

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L644

## Tool used

Manual Review

## Recommendation

Value of the underlying assets should be derived from the vault shares and value, rather than being stored directly.