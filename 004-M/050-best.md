koxuan

medium

# chainlink oracle does not check answer returned is not 0

## Summary
Chainlink price feed can return 0 and there is no check on it.

## Vulnerability Detail

Chainlink price feed `latestRoundData` is used to retrieve price feed from chainlink. `toUint256()` makes sure that the answer is not negative and there is a check that makes sure the price is not stale. However, no checks are done to ensure that the answer returned is not 0.

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
}

```

## Impact
Incorrect price can be returned from chainlink which will cause problems downstream when new spells are added.

## Code Snippet
[ChainlinkAdapterOracle.sol#L66-L85](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L66-L85)

## Tool used

Manual Review

## Recommendation

Check answer returned from chainlink and make sure it is more than 0.
