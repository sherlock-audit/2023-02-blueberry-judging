Nyx

medium

# Position shouldn't be liquidatable when repay is not allowed.

## Summary
When repay is not allowed, liquidation shouldn't be possible. 
## Vulnerability Detail
Bob's position was changed to liquidatable, but he can't add collateral because repay is closed. Anyone can liquidate his position anytime.
## Impact
Users might be liquidated unexpectedly.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L747
## Tool used

Manual Review

## Recommendation
liquidate() should revert when repay is not allowed, or repay() should never be paused.