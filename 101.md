rvierdiiev

medium

# Liquidation is allowed when repay and deposit is paused

## Summary
Liquidation is allowed when repay and deposit is paused. Because of that borrower can't deposit or repay to make his position non liquidatable.
## Vulnerability Detail
When someonce calls `BlueBerryBank.liquidate`, then `isLiquidatable` function is called in order to see if this position can be liquidated. The main thing that is checked is position risk.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495
```solidity
    function getPositionRisk(uint256 positionId)
        public
        view
        returns (uint256 risk)
    {
        Position storage pos = positions[positionId];
        uint256 pv = getPositionValue(positionId);
        uint256 ov = getDebtValue(positionId);
        uint256 cv = oracle.getUnderlyingValue(
            pos.underlyingToken,
            pos.underlyingAmount
        );


        if (cv == 0) risk = 0;
        else if (pv >= ov) risk = 0;
        else {
            risk = ((ov - pv) * DENOMINATOR) / cv;
        }
    }
```
For us now important that position risk depends on position debt, and underlying amount.
So user can decrease liquidation risk by decreasing debt: repaying and increasing underlying token amount: depositing.

Bot of this functions [can be paused](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L233-L241), however liquidation is allowed any time.

As result when depositing or repaying (or both) will be paused then position owner loses control on position and he can be liquidated as liquidation doesn't check if smth is paused. 
## Impact
Position owner loses control on position and it can be liquidated.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
I believe you should restrict liquidations when depositing or repaying is paused.