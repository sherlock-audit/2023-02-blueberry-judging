obront

medium

# If a token's oracle goes down or price falls to zero, liquidations will be frozen

## Summary

In some extreme cases, oracles can be taken offline or token prices can fall to zero. In these cases, liquidations will be frozen (all calls will revert) for any debt holders holding this token, even though they may be some of the most important times to allow liquidations to retain the solvency of the protocol.

## Vulnerability Detail

Chainlink has taken oracles offline in extreme cases. For example, during the UST collapse, Chainlink paused the UST/ETH price oracle, to ensure that it wasn't providing inaccurate data to protocols.

In such a situation (or one in which the token's value falls to zero), all liquidations for users holding the frozen asset would revert. This is because any call to `liquidate()` calls `isLiquidatable()`, which calls `getPositionRisk()`, which calls the oracle to get the values of all the position's tokens (underlying, debt, and collateral).

Depending on the specifics, one of the following checks would cause the revert:
- the call to Chainlink's `registry.latestRoundData` would fail
- `if (updatedAt < block.timestamp - maxDelayTime) revert PRICE_OUTDATED(_token);`
- `if (px == 0) revert PRICE_FAILED(token);`

If the oracle price lookup reverts, liquidations will be frozen, and the user will be immune to liquidations. Although there are ways this could be manually fixed with fake oracles, by definition this happening would represent a cataclysmic time where liquidations need to be happening promptly to avoid the protocol falling into insolvency. 

## Impact

Liquidations may not be possible at a time when the protocol needs them most. As a result, the value of user's asset may fall below their debts, turning off any liquidation incentive and pushing the protocol into insolvency.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L517

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L497-L505

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L488

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L182-L189

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L95-L99

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L66-L84

## Tool used

Manual Review

## Recommendation

Ensure there is a safeguard in place to protect against this possibility. 