0x52

high

# IchiLpOracle is extemely easy to manipulate due to how IchiVault calculates underlying token balances

## Summary

`IchiVault#getTotalAmounts` uses the `UniV3Pool.slot0` to determine the number of tokens it has in it's position. `slot0` is the most recent data point and is therefore extremely easy to manipulate. Given that the protocol specializes in leverage, the effects of this manipulation would compound to make malicious uses even easier.

## Vulnerability Detail

[ICHIVault.sol](https://etherscan.io/token/0x683f081dbc729dbd34abac708fa0b390d49f1c39#code#L3098)

    function _amountsForLiquidity(
        int24 tickLower,
        int24 tickUpper,
        uint128 liquidity
    ) internal view returns (uint256, uint256) {
        (uint160 sqrtRatioX96, , , , , , ) = IUniswapV3Pool(pool).slot0();
        return
            UV3Math.getAmountsForLiquidity(
                sqrtRatioX96,
                UV3Math.getSqrtRatioAtTick(tickLower),
                UV3Math.getSqrtRatioAtTick(tickUpper),
                liquidity
            );
    }

`IchiVault#getTotalAmounts` uses the `UniV3Pool.slot0` to determine the number of tokens it has in it's position. [slot0](https://docs.uniswap.org/contracts/v3/reference/core/interfaces/pool/IUniswapV3PoolState#slot0) is the most recent data point and can easily be manipulated.

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L27-L36

`IchiLPOracle` directly uses the token values returned by `vault#getTotalAmounts`. This allows a malicious user to manipulate the valuation of the LP. An example of this kind of manipulation would be to use large buys/sells to alter the composition of the LP to make it worth less or more. 

## Impact

Ichi LP value can be manipulated to cause loss of funds for the protocol and other users

## Code Snippet

## Tool used

Manual Review

## Recommendation

Token balances should be calculated inside the oracle instead of getting them from the `IchiVault`. To determine the liquidity, use a TWAP instead of `slot0`.