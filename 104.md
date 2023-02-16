rvierdiiev

medium

# BlueBerryBank.getPositionRisk shows risk 0, when underlying price is 0

## Summary
BlueBerryBank.getPositionRisk shows risk 0, when underlying price is 0
## Vulnerability Detail
BlueBerryBank.getPositionRisk function is responsible for providing risk of position. Depending on this risk position can be liquidated.
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

The problem that in case if cv == 0, returned risk is 0.
If cv == 0 that means that price of underlying token became 0 and that risk should be maximum.

1.Suppose that user borrowed using some underlying token tokenA.
2.Then smth happened with that token, so it costs nothing now.
3.But position is not liquidatable as it will return risk 0 and is considered as healthy.
## Impact
Lost of borrowed funds for protocol.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495
## Tool used

Manual Review

## Recommendation
In case if underlying token price became 0 you need to return risk that is already liquidatable.