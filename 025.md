cergyk

high

# Error in liquidation amounts calculation can lead to draining of protocol funds

## Summary
When liquidating, the liquidator receives a share of the collateral deposited by the user. Unfortunately the liquidator receives too much collateral from the position.

## Vulnerability Detail
As may be seen in the liquidation process:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L523-L531

the liquidator receives a share of the collateral proportional to the share of debt he repaid, only the share accounts for the debt in only 1 token. The liquidated user may have debt in multiple tokens/banks, and the liquidator should receive a share of the collateral proportional to the share of the whole debt he helped repay. 

## Impact
A malicious user can drain the fund by taking valid borrows in multiple tokens, then autoliquidating by repaying the whole debt associated to one token (which can be a very small amount). In that case the malicious user would get all their collateral back.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Calculate liqSize and other amounts paid to the liquidator normalized by `getDebtValue()` (accross all debt tokens).