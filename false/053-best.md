chaduke

medium

# ``reducePosition`` fails to do input sanity check for strategy and collateral.

## Summary
In contrast to other external functions, the ``reducePosition`` does not use ``existingStrategy(strategyId)`` and  ``existingCollateral(strategyId, collToken)``
to perform the sanity check of strategy and collateral. 

## Vulnerability Detail
All other external functions have used ``existingStrategy(strategyId)`` and  ``existingCollateral(strategyId, collToken)`` to perform the sanity check for strategy and collateral. ``reducePosition()`` has not done so. 

[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266-L274](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266-L274) 



## Impact
It is possible that the input ``strategy`` and ``collateral`` are invalid due to lack of existence check.

## Code Snippet
see above

## Tool used
VScode

Manual Review

## Recommendation
Add both checks below
```diff
 function reducePosition(
        uint256 strategyId,
        address collToken,
        uint256 collAmount
    ) external
+ existingStrategy(strategyId)
+ existingCollateral(strategyId, collToken)
{
        doWithdraw(collToken, collAmount);
        doRefund(collToken);
        _validateMaxLTV(strategyId);
    }

```