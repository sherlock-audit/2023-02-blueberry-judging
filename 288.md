Jeiwan

high

# Partial liquidations leave the risk unchanged, causing more liquidations

## Summary
A partial liquidation doesn't change the risk of a position (it's supposed to reduce it to bring the position below the liquidation threshold). The position can be liquidation again and again until it's fully liquidated. As a result, no partial liquidations are possible in the current version of the protocol.
## Vulnerability Detail
The [risk of a position is calculated as](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L493):
> ((ov - pv) * DENOMINATOR) / cv

Where, `ov` is the position's debt, `pv` is the value of the LP tokens of the position, `cv` is the value of the isolated collateral of the position. I.e. it's the ratio of a debt value to a collateral value, where debt is calculated as the borrowed amount (including accrued interest) minus the value of the LP tokens (which are redeemable for the tokens that were borrowed).

A position can be [liquidated](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L517) when [its risk is equal or greater than the liquidation threshold](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L504). Liquidating a positions requires [repaying](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L523) of a portion or a full borrowed amount. In exchange, the liquidator gets proportional amounts of position's: [LP tokens](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L529), [isolated collateral](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L530), and [vault shares](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L531). The proportion is determined as:
```solidity
uint256 liqSize = (pos.collateralSize * share) / oldShare;
```

Where `share` is the share of the debt that was repaid and `oldShare` is the initial debt share of the position. If `share` equals `oldShare`, then the whole debt was repaid; if `share` is less than `oldShare`, the the debt was repaid partially.

As can be seen from the above, when a debt is repaid partially, all the variables in the risk formula are reduced by the same proportion (`share / oldShare`), which means that the risk remains unchanged, and it stays above the liquidation threshold. Consider the following example:
1. assume that the debt is 270, the value of LP tokens is 100, and the collateral value is 200;
1. the risk is then: `(270 - 100) / 200 = 0.85`;
1. assume that the liquidation threshold is 0.85, thus the position is liquidatable;
1. a liquidator liquidates 25% of the debt;
1. now, the risk is: `((270 * 0.75) - (100 * 0.75)) / (200 * 0.75) = 0.85`, thus the position can be liquidated again and again until it's fully liquidated.

This situations is covered by the tests: in the [bank.test.ts](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/test/bank.test.ts#L184) file, the position risk before the partial liquidation is 117.05 % and after the partial liquidation it's still 117.05 %, even though the debt and the value of the position are reduced.
## Impact
Users can lose all their collateral and LP tokens as a result of a liquidation, since partial liquidations doesn't reduce the risk of their positions.
## Code Snippet
[BlueBerryBank.sol#L529-L535](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L529-L535)
## Tool used
Manual Review
## Recommendation
Consider awarding liquidator with a collateral amount that's calculated based on the amount of debt repaid and the price of the debt token in terms of the collateral token, with a small bonus (the bonus is the reward that liquidators get for liquidating positions).

For the risk of a position to improve as a result of a liquidation, either the debt-to-collateral ratio must decrease, or the collateral-to-debt ratio must increase. Since it's desirable to liquidate a specific amount of debt, the former options the only possible. Thus, the reduction in collateral must be smaller than the reduction in debt, and liquidity providers must not receive the entire `share / oldShare` proportion of the isolated collateral of a position.