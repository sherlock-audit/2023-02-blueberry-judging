sakshamguruji

medium

# Protocol's usability becomes very limited when access to Chainlink oracle data feed is blocked

## Summary

The current implementation of the chainlink oracle might cause a DoS as the getPrice function is not wrapped inside a try catch block.


## Vulnerability Detail

Based on the current implementation, when the protocol wants to use Chainlink oracle data feed for getting a collateral token's price, the fixed price for the token should not be set. When the fixed price is not set for the token, calling the Oracle contract's  getPrice function will execute 

`(, int256 answer, , uint256 updatedAt, ) = registry.latestRoundData( token, USD );`
. This is in the  ChainLinkAdapterOracle https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L76. As https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/ mentions, it is possible that Chainlink’s "multisigs can immediately block access to price feeds at will". When this occurs, executing `latestRoundData` reverts , which causes denial of service for the functions using the
getPrice function here https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L66

## Impact

DoS While calling the getPrice function.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Put the getPrice function inside a try-catch block. The logic for getting the collateral token's price from the Chainlink oracle data feed should be placed in the try block while some fallback logic when the access to the Chainlink oracle data feed is denied should be placed in the catch block.
