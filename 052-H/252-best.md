rbserver

high

# User is able to reduce a position by withdrawing more collateral than she or he should be allowed to

## Summary
Calling the `IchiVaultSpell.reducePosition` function with an existing `strategyId` that is not associated with the position but is for the same collateral token, which the maximum LTV for this unassociated `strategyId` is higher than the maximum LTV for the position's associated `strategyId`, can allow the user to reduce the position by withdrawing more collateral than she or he should be allowed to.

## Vulnerability Detail
Calling the `IchiVaultSpell.reducePosition` function executes `_validateMaxLTV(strategyId)`. Calling the `IchiVaultSpell._validateMaxLTV` function then executes `if (debtValue > (collValue * maxLTV[strategyId][collToken]) / DENOMINATOR) revert EXCEED_MAX_LTV()`. When the position has some `debtValue`, its collateral can only be reduced by up to a certain amount before exceeding the maximum LTV corresponding to the position's associated `strategyId`. However, the `IchiVaultSpell.reducePosition` function can be called with an existing `strategyId` that is not associated with the position but is for the same collateral token in which the `maxLTV[strategyId][collToken]` for this unassociated `strategyId` is higher than the `maxLTV[strategyId][collToken]` for the position's associated `strategyId`. In this way, the position's collateral can be withdrawn by an amount that is higher than it should be allowed.

For example, for the same collateral token, if calling the `IchiVaultSpell.withdrawInternal` function with the position's associated `strategyId` can allow up to `N` collateral tokens to be withdrawn before exceeding the maximum LTV corresponding to such associated `strategyId`, then calling the `IchiVaultSpell.reducePosition` function with an existing `strategyId` that is not associated with the position can possibly allow up to more than `N` collateral tokens to be withdrawn before exceeding the maximum LTV corresponding to such unassociated `strategyId`.

## Impact
For the described scenario, the user is able to reduce a position by withdrawing more collateral than she or he should be allowed to.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266-L274
```solidity
    function reducePosition(
        uint256 strategyId,
        address collToken,
        uint256 collAmount
    ) external {
        doWithdraw(collToken, collAmount);
        doRefund(collToken);
        _validateMaxLTV(strategyId);
    }
```

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L101-L113
```solidity
    function _validateMaxLTV(uint256 strategyId) internal view {
        uint256 debtValue = bank.getDebtValue(bank.POSITION_ID());
        (, address collToken, uint256 collAmount, , , , , ) = bank
            .getCurrentPositionInfo();
        uint256 collPrice = bank.oracle().getPrice(collToken);
        uint256 collValue = (collPrice * collAmount) /
            10**IERC20Metadata(collToken).decimals();

        if (
            debtValue >
            (collValue * maxLTV[strategyId][collToken]) / DENOMINATOR
        ) revert EXCEED_MAX_LTV();
    }
```

## Tool used
Manual Review

## Recommendation
If the position has a positive `debtValue`, the `IchiVaultSpell._validateMaxLTV` function can be updated to revert if it is called with a `strategyId` that is not associated with the position.