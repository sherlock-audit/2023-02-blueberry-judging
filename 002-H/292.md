Jeiwan

high

# `UniswapV3AdapterOracle` always reports 0 price, allowing maximal borrowing and removal of entire collateral

## Summary
`UniswapV3AdapterOracle` imports and uses `UniV3WrappedLibMockup`, however the latter is a stub that doesn't implement the functions defined in it. As a result, the `UniswapV3AdapterOracle.getPrice` function will always return 0. If it's used to calculated the debt value of a position, the value will be 0, allowing the owner of the position to remove all their collateral while keeping the borrowed tokens.
## Vulnerability Detail
[UniswapV3AdapterOracle](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/UniswapV3AdapterOracle.sol#L15) is an oracle that's used to obtain prices from Uniswap V3 pools' oracles. The contract defines the `getPrice` function, which calls [UniV3WrappedLibMockup.consult](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/UniswapV3AdapterOracle.sol#L54) and [UniV3WrappedLibMockup.getQuoteAtTick](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/UniswapV3AdapterOracle.sol#L58-L59). However, the functions are [not implemented in the contract](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/libraries/UniV3/UniV3WrappedLibMockup.sol#L14-L25):
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
## Impact
`UniswapV3AdapterOracle` cannot be used in production, its `getPrice` function will always return 0, which will lead to incorrect calculations of collateral and debt values. If the oracle is used to calculate the value of a debt, the value will be 0, and the borrower will be able to remove all their collateral.
## Code Snippet
[UniswapV3AdapterOracle.sol#L54-L64](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/UniswapV3AdapterOracle.sol#L54-L64)
[UniV3WrappedLibMockup.sol#L14-L25](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/libraries/UniV3/UniV3WrappedLibMockup.sol#L14-L25)
## Tool used
Manual Review
## Recommendation
In `UniswapV3AdapterOracle`, consider using `UniV3WrappedLib` instead of `UniV3WrappedLibMockup`.