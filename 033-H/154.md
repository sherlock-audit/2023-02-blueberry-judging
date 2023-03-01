obront

high

# IchiLPOracle does not use correct LP Pricing, vulnerable to flash loan attack

## Summary

The `IchiLpOracle` contract uses the reserves and token prices to calculate the value of the ICHI LP token. However, it doesn't implement the correct "Fair LP Pricing" formula (which is correctly implemented in UniswapV2Oracle), leaving it susceptible to a flash loan attack.

## Vulnerability Detail

When using the reserves and token prices of an LP pool to calculate the value of the LP tokens, we can use the following function:

`price of LP token = (sum of all assets (price of asset * reserves of asset)) / total supply of LP tokens`

However, as outlined in the famous [Fair LP Token Pricing article](https://blog.alphaventuredao.io/fair-lp-token-pricing/) the naive implementation of this formula is vulnerable to flash loans.

> For example, if an asset price is derived from AMM spot price, then an attacker can skew the spot price with a flash-loan sandwich attack. Similarly for the reserve derived from AMM reserves, e.g. from Uniswap's getReserves()

>A flash-loan sandwich attack involves 3 steps:
>1. Flash loan and pump the underlying asset to the pool (temporarily faking reserve balances)
>2. Attack the target protocol (getting false reserve balances)
>3. Dump the asset back and return the borrowed asset.

While the `UniswapV2Oracle.sol` calculations are implemented correctly, protecting against this possibility, the `IchiLpOracle.sol` implementation is the vulnerable naive implementation.

This allows a user to take a flash loan and arbitrarily increase the protocol's view of the LP token value during a transaction, which allows them to avoid liquidations and steal funds.

## Impact

A user can manipulate the `IchiLpOracle` contract to avoid liquidations and steal the protocol's funds by temporarily inflating the value of the Ichi LP tokens during their transaction.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L19-L39

## Tool used

Manual Review

## Recommendation

Adjust the core in the `getPrice()` formula to match the code in the other AMM oracles, properly implementing the Fair LP Pricing formula and ensuring protection against flash loans.