cducrest-brainbot

medium

# getDebtValue might run out of gas

## Summary

The function to get the debt value of a position loops over all the banks related to that position (bounded by 256 banks) and can run out of gas.

## Vulnerability Detail

The `getDebtValue` function of `BlueBerryBank.sol` loops over all the banks related to the given position. For each it will query the debt value of the user for the given token. This will call `getPrice` for the token which, depending on the settings of the oracle can call `AggregatorOracle`'s `getPrice()` function doing an aggregate price over three oracles.

If we count the most expensive operation gas-wise, for each bank we have: 3 sload in `BlueBerryBank.getDebtValue`, 1 sload in `CoreOracle._getPrice`, 2 + nr_sources sload in `AggregatorOracle.getPrice`, and nr_sources times the cost of `getPrice` on the specific oracle.

For the uniswapv3 oracle the cost of `getPrice` is 6 sload + the cost of `consult` + the cost of `getQuoteAtTick` + the cost of the cost of `getPrice()` on the bast token with which the initial token is paired.

The code for [`consult` and `getQuoteAtTick` is here](https://github.com/Uniswap/v3-periphery/blob/51f8871aaef2263c8e8bbf4f3410880b6162cdea/contracts/libraries/OracleLibrary.sol#L16), the code observes the pool state as described [here](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Oracle.sol). I am unsure about the cost of these functions, let's call it `x = cost(consult + getQuoteAtTick + base.getPrice`, I assume that `x ~= 10k gas`, which sounds conservative.

If we imagine an aggregator oracle with three price sources with a similar cost as UniswapV3Oracle and `x ~= 10k gas`
`total_cost = nr_bank * ((6 + nr_sources) * sload + nr_sources * (6 sload + x))`
`total_cost = nr_bank * 86700`

For a chain with a gas limit of 8m, this means the code would run out of gas for less than 100 banks. These eyeballing calculations were conservative to see how far we are from a block gas limit, and show the possibility of running out of gas.

## Impact

The `getDebtValue`function is called whenever trying to figure out if a function is liquidatable. If this function cannot run through, a position cannot be liquidated and the owner cannot execute any operation on it (since `isLiquidatable` is called at the end of `execute`). This results in the locking of a position that can never be closed.

## Code Snippet

`getDebtValue` loop:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L451-L475

`CoreOracle`'s `getPrice()` function: 

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L95-L105

Relevant part of `AggregatorOracle`'s `getPrice()`:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/AggregatorOracle.sol#L89-L111

`UniswapV3AdapterOracle`'s `getPrice()`:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/UniswapV3AdapterOracle.sol#L41-L69

## Tool used

Manual Review

## Recommendation

Limit the number of banks for a position to below 50.