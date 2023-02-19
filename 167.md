sinarette

medium

# Loss of Precision in IchiLpOracle

## Summary
Price calculation in ````IchiLpOracle#getPrice```` would make some loss of precisions.

## Vulnerability Detail
```solidity
    uint256 totalReserve = (r0 * px0) /
        10**t0Decimal +
        (r1 * px1) /
        10**t1Decimal;

    return (totalReserve * 1e18) / totalSupply;
```
The price calculation of IchiLpOracle first divides the calculated token price into token decimals, then multiplies it by 1e18, after summing up the prices of tokens. This could lead to significant loss in precision.

## Impact
As many functions of the lending protocol, such as risk calculation depends on the oracle, less precise oracle prices could lead to unintended actions like liquidations.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol/#L19-L40

## Tool used

Manual Review

## Recommendation
```solidity
-    uint256 totalReserve = (r0 * px0) /
-       10**t0Decimal +
-       (r1 * px1) /
-       10**t1Decimal;
- 
-   return (totalReserve * 1e18) / totalSupply;

+   uint256 totalReserve = (r0 * px0 * 1e18) / 10**t0Decimal +
+       (r1 * px1 * 1e18) / 10**t1Decimal;
+
+   return totalReserve / totalSupply;
```
