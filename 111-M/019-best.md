0x52

medium

# Collateral cannot be withdrawn if it can't be priced even if the position has no outstanding debt

## Summary

Collateral cannot be withdrawn if it can't be adequately priced by an oracle. This includes withdrawals for an account with no debt. This collateral is unfairly and indefinitely trapped until the oracle begins to function again.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L595-L607

At the end of every execute call it validates that the end position is not liquidatable via isLiquidatable

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L497-L505

This then establishes the positions current risk level using getPositionRisk

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L482-L493

If the oracle is not functioning for the underlying token then the call with always revert when trying to value the underlying tokens. This traps the underlying in the position regardless of whether the position is carrying any debt. Until the oracle is working properly again, the user has no way of retrieving their collateral. Users that have no debt or can eliminate their debt in a single transaction should not be subject to collateral valuation because no debt means no risk.

This delay, which is potentially indefinite, can cause material damage to the user. As an example if their asset is a bridge asset that is depegging then every moment counts when selling the asset. In the event like this the aggregator oracle may freeze due to [differences](https://docs.blueberry.garden/lending-protocol/price-oracle#price-feed-alternative) between underlying oracles in which asset is being tracked (bridged vs underlying).

## Impact

User collateral is unfairly trapped

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495

## Tool used

Manual Review

## Recommendation

Shortcircuit getPositionRisk to return 0 if the position has no debt. This avoids the scenario where the underlying collateral can't be priced because the entire valuation is bypassed. It is safe because if there is no debt at all then by definition there's no risk or possible liquidation:

        Position storage pos = positions[positionId];
    +   uint256 ov = getDebtValue(positionId);
    +   if (ov == 0) return 0;
        uint256 pv = getPositionValue(positionId);
    -   uint256 ov = getDebtValue(positionId);
        uint256 cv = oracle.getUnderlyingValue(
            pos.underlyingToken,
            pos.underlyingAmount
        );

        if (cv == 0) risk = 0;
        else if (pv >= ov) risk = 0;
        else {
            risk = ((ov - pv) * DENOMINATOR) / cv;
        }