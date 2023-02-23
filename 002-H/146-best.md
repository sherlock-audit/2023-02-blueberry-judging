obront

high

# UniswapV3AdapterOracle incorrectly uses library, so will return 0 for all prices

## Summary

When `UniswapV3AdapterOracle.sol` is used to generate the price of an asset, it uses the `UniV3WrappedLibMockup` library to perform certain calculations. However, this points to an imported library (rather than a deployed contract) with unimplemented functions that return zero, leading the oracle to return zero for all calls.

## Vulnerability Detail

The `UniswapV3AdapterOracle#getPrice()` function is intended to get the price of the LP tokens of a pool. It does this by pulling the tokens that are included in the pool, pulling the decimals from the tokens, and using `UniV3WrappedLibMockup` to perform some calculations on these variables.

The return value of these calculations is multiplied by the oracle's price for one of the underlying pool tokens, and the result is the value of the LP tokens.

```solidity
function getPrice(address token) external view override returns (uint256) {
    ...

    address token0 = IUniswapV3Pool(stablePool).token0();
    address token1 = IUniswapV3Pool(stablePool).token1();
    token1 = token0 == token ? token1 : token0; // get stable token address
    uint256 stableDecimals = uint256(IERC20Metadata(token1).decimals());
    uint256 tokenDecimals = uint256(IERC20Metadata(token).decimals());
    (int24 arithmeticMeanTick, ) = UniV3WrappedLibMockup.consult(
        stablePool,
        secondsAgo
    );

    uint256 quoteTokenAmountForStable = UniV3WrappedLibMockup
        .getQuoteAtTick(
            arithmeticMeanTick,
            uint128(10**tokenDecimals),
            token,
            token1
        );

    return
        (quoteTokenAmountForStable * base.getPrice(token1)) /
        10**stableDecimals;
}
```
Since `UniV3WrappedLibMockup` doesn't wrap a real address, it appears to be a library that should implement the functions itself to perform these calculations. However, if we look within the library, the functions are unimplemented:

```solidity
function consult(address pool, uint32 secondsAgo)
    external
    view
    returns (int24 arithmeticMeanTick, uint128 harmonicMeanLiquidity)
{}

function getQuoteAtTick(
    int24 tick,
    uint128 baseAmount,
    address baseToken,
    address quoteToken
) external pure returns (uint256 quoteAmount) {}
```
The result is that each of these functions will return zero. When this return value is multiplied by the oracle's price for `token1` (regardless of what the price is), the result will be a value of 0 for the LP token.

To simplify validation of this issue, I've created a plug-and-play Forge test that POCs the bug. In the test, the underlying token has a value of 999e18, but the LP token still gets a value returned of 0.

https://gist.github.com/zobront/57e866ea4b09b26980d0341984d17bb5

## Impact

The Blueberry protocol relies on accurate oracles to function, as they are used to check liquidations, value collateral, check LTV ratios, etc. If an oracle was incorrect, it could risk either (a) allowing a user to borrow an infinite number of these LP tokens, because they are measured as valueless or (b) liquidate a user who holds these tokens as collateral, because the collateral is deemed worthless.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/UniswapV3AdapterOracle.sol#L41-L69

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/libraries/UniV3/UniV3WrappedLibMockup.sol#L14-L25

## Tool used

Manual Review

## Recommendation

Either properly implement the library to perform the calculations you're looking for, or simply turn `UniV3WrappedLibMockup` into an interface that wraps a live contract address where you can get these values.