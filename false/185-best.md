8olidity

high

# maxLTV can be covered

## Summary
maxLTV can be covered
## Vulnerability Detail
In function `addcollateralssupport()`, administrators can add new Collaterals. This is his normal function. But there is no considering coverage. If the corresponding `maxLTV[Strategyid][Collaterals]` is worth it, because the function does not determine whether the corresponding key is worth it. When the corresponding key is worth it and modify it. There is no prompt. Administrators are likely to forget the previous settings. Cover the original value.
```solidity
function addCollateralsSupport(
    uint256 strategyId,
    address[] memory collaterals,
    uint256[] memory maxLTVs
) external existingStrategy(strategyId) onlyOwner { 
    if (collaterals.length != maxLTVs.length || collaterals.length == 0)
        revert INPUT_ARRAY_MISMATCH();

    for (uint256 i = 0; i < collaterals.length; i++) {
        if (collaterals[i] == address(0)) revert ZERO_ADDRESS();
        if (maxLTVs[i] == 0) revert ZERO_AMOUNT();
        maxLTV[strategyId][collaterals[i]] = maxLTVs[i]; //@audit  
    }

    emit CollateralsSupportAdded(strategyId, collaterals, maxLTVs);
}
```
## Impact
maxLTV can be covered
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L84-L99
## Tool used

Manual Review

## Recommendation
When `maxLTV[Strategyid][COLLERARS [i]] ≠ 0`, a warning is issued, and it is prompted that it cannot be modified