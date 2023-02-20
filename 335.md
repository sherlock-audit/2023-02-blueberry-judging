peanuts

medium

# IchiLpOracle will malfunction if token0 or token1 decimal is not 18

## Summary

IchiLpOracle will malfunction if token0 or token1 decimal is not 18.

## Vulnerability Detail

IchiLPOracle.sol `getPrice()` takes the price and decimals of t0 and t1. The function then calculates the totalReserve and returns (totalReserve * 1e18) / totalSupply. 

```solidity
        (uint256 r0, uint256 r1) = vault.getTotalAmounts();
        uint256 px0 = base.getPrice(address(token0));
        uint256 px1 = base.getPrice(address(token1));
        uint256 t0Decimal = IERC20Metadata(token0).decimals();
        uint256 t1Decimal = IERC20Metadata(token1).decimals();


        uint256 totalReserve = (r0 * px0) /
            10**t0Decimal +
            (r1 * px1) /
            10**t1Decimal;


        return (totalReserve * 1e18) / totalSupply;
```

The Ichi Token, which is accepted by the protocol, has 9 decimals:

https://etherscan.io/token/0x903bef1736cddf2a537176cf3c64579c3867a881

## Impact

Lp Token price will be calculated incorrectly.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L19-L40

## Tool used

Manual Review

## Recommendation

Normalize r0 and r1 decimals to be 18 first before performing calculations.