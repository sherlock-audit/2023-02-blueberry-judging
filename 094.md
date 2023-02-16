SPYBOY

medium

# Chainlink's latestRoundData  return stale or incorrect result

## Summary
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L76

## Vulnerability Detail

## Impact
On ChainlinkAdapterOracle.sol, you are using latestRoundData, but there is no check if the return value indicates stale data. 
```solidity
function getPrice(address _token) external view override returns (uint256) {
        // remap token if possible
        address token = remappedTokens[_token];
        if (token == address(0)) token = _token;

        uint256 maxDelayTime = maxDelayTimes[token];
        if (maxDelayTime == 0) revert NO_MAX_DELAY(_token);

        // try to get token-USD price
        uint256 decimals = registry.decimals(token, USD);
        (, int256 answer, , uint256 updatedAt, ) = registry.latestRoundData(
            token,
            USD
        );
        if (updatedAt < block.timestamp - maxDelayTime)
            revert PRICE_OUTDATED(_token);

        return (answer.toUint256() * 1e18) / 10**decimals;
    }
```
This could lead to stale prices according to the Chainlink documentation:
https://docs.chain.link/data-feeds/price-feeds/historical-data
Related report:
https://github.com/code-423n4/2021-05-fairside-findings/issues/70

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L76
## Tool used

Manual Review

## Recommendation
Add the below check for returned data
```solidity
function getPrice(address _token) external view override returns (uint256) {
        // remap token if possible
        address token = remappedTokens[_token];
        if (token == address(0)) token = _token;

        uint256 maxDelayTime = maxDelayTimes[token];
        if (maxDelayTime == 0) revert NO_MAX_DELAY(_token);

        // try to get token-USD price
        uint256 decimals = registry.decimals(token, USD);
        (uint80 roundID, int256 answer, uint256 timestamp, uint256 updatedAt, ) = registry.latestRoundData(
            token,
            USD
        );
        //Solution
        require(updatedAt >= roundID, "Stale price");
        require(timestamp != 0,"Round not complete");
        require(answer > 0,"Chainlink answer reporting 0");

        if (updatedAt < block.timestamp - maxDelayTime)
            revert PRICE_OUTDATED(_token);

        return (answer.toUint256() * 1e18) / 10**decimals;
    }
```