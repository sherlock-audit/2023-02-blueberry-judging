8olidity

medium

# No check  sequencer is down in Chainlink feeds

## Summary
No check  sequencer is down in Chainlink feeds
## Vulnerability Detail
`answer` data returned in `latestRoundData()`, representing sequencer ,

// Answer == 0: Sequencer is up
// Answer == 1: Sequencer is down

There is no check that the sequencer is down
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
    ); //@audit 
    if (updatedAt < block.timestamp - maxDelayTime)
        revert PRICE_OUTDATED(_token);

    return (answer.toUint256() * 1e18) / 10**decimals;
}
```
It is recommended to follow the code example of Chainlink:

**[https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code)**


## Impact
No check  sequencer is down in Chainlink feeds
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L66-L84
## Tool used

Manual Review

## Recommendation

```solidity
        bool isSequencerUp = answer == 0;
        if (!isSequencerUp) {
            revert SequencerDown();
        }
```
