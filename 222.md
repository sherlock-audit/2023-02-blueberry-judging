ctf_sec

high

# UniV3WrappedLibMockup is incorrectly used.

## Summary

UniV3WrappedLibMockup is incorrectly used.

## Vulnerability Detail

The UniV3WrappedLibMockup has no implementation, but it is still used in UniswapV3AdapterOracle.sol

```solidity
 /// @dev Return the USD based price of the given input, multiplied by 10**18.
    /// @param token The ERC-20 token to check the value.
    function getPrice(address token) external view override returns (uint256) {
        // Maximum cap of maxDelayTime is 2 days(172,800), safe to convert
        uint32 secondsAgo = uint32(maxDelayTimes[token]);
        if (secondsAgo == 0) revert NO_MEAN(token);

        address stablePool = stablePools[token];
        if (stablePool == address(0)) revert NO_STABLEPOOL(token);

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

which calls:

```solidity
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
```

which incorrectly calls:

```solidity
// OracleLibrary
function consult(address pool, uint32 secondsAgo)
    external
    view
    returns (int24 arithmeticMeanTick, uint128 harmonicMeanLiquidity)
{}
```

and

```solidity
function getQuoteAtTick(
    int24 tick,
    uint128 baseAmount,
    address baseToken,
    address quoteToken
) external pure returns (uint256 quoteAmount) {}
```

and quoteTokenAmountForStable would be 0 which means UniswapV3Adapter.sol#getPrice return 0 given the code:

```solidity
return
    (quoteTokenAmountForStable * base.getPrice(token1)) /
    10**stableDecimals;
```

 and malfunction and the protocol is not able to use Uniswap V3 as the oracle source which block the majority of token price oracle because Uniswap V3 has thick liquidity for majority of trading pairs.

## Impact

the protocol is not able to use Uniswap V3 as the oracle source which block the majority of token price oracle because Uniswap V3 has thick liquidity for majority of trading pairs.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/UniswapV3AdapterOracle.sol#L38-L69

## Tool used

Manual Review

## Recommendation

We recommend the protocol use UniV3WrappedLib instead of UniV3WrappedLibMockUp
