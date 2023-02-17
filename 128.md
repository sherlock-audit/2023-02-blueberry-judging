obront

medium

# Check for stale data before trusting Chainlink's response

## Summary

When `ChainlinkAdapterOracle.sol` fetches the price from Chainlink, it is missing crucial checks recommended by Chainlink to ensure the data is fresh and accurate.

## Vulnerability Detail



According to [Chainlink's documentation](https://docs.chain.link/data-feeds/price-feeds/historical-data), the following check is necessary to ensure that the data is fresh:

> If answeredInRound is less than roundId, the answer is being carried over.

The current `getPrice()` function does check the timestamp it was last updated, but does not implement a check on the round of the answer:
```solidity
function getPrice(address _token) external view override returns (uint256) {
    ...
    (, int256 answer, , uint256 updatedAt, ) = registry.latestRoundData(
        token,
        USD
    );
    if (updatedAt < block.timestamp - maxDelayTime)
        revert PRICE_OUTDATED(_token);

    return (answer.toUint256() * 1e18) / 10**decimals;
}
```

## Impact

Chainlink could return stale data that would be accepted by the protocol, either allowing users to collateralize more than intended (because of `_validateMaxLTV` checks) or liquidating users who should not be liquidated (because of `isLiquidatable` checks).

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L66-L84

## Tool used

Manual Review

## Recommendation

Add the following checks:

```diff
function getPrice(address _token) external view override returns (uint256) {
    ...
-    (, int256 answer, , uint256 updatedAt, ) = registry.latestRoundData(
+    (uint80 roundId, int256 answer, , uint256 updatedAt, uint80 answeredInRound) = registry.latestRoundData( 
        token,
        USD
    );
    if (updatedAt < block.timestamp - maxDelayTime)
        revert PRICE_OUTDATED(_token);
+   if (answeredInRound < roundID)
+        revert PRICE_OUTDATED(_token);

    return (answer.toUint256() * 1e18) / 10**decimals;
}
```