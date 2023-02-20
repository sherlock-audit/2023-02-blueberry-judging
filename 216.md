Aymen0909

medium

# `maxLTV` can be set greater than `1e4` leading to incorrect behaviours of `IchiVaultSpell` functions

## Summary

The function `addCollateralsSupport` is used to add/update a collatral `maxLTV` value but it allows the value of `maxLTV` to be set greater than `1e4`, this causes the function `_validateMaxLTV` to behave in an unexpected manner and thus leading to incorrect behaviours in the open/close operations of the vault.

## Vulnerability Detail

The function `addCollateralsSupport` is used to add/update a collatral `maxLTV` value, but it does not check if the new value is greater than `1e4` which is supposed to be the base (the max value).

The value of `maxLTV` is used in the function `_validateMaxLTV` to verify that the debt value is always less than the collateral value multiplied by `maxLTV / 1e4` and it should revert if not, so if the `maxLTV` value is set greater than `1e4` this function can have incorrect results : reverting when it shouldn't or succedding when it should revert.

As the function `_validateMaxLTV` is called in the functions depositInternal/withdrawInternal/reducePosition which are later called by the open/close functions of the vault, this error will cause unexpected behavior of those functions thus allowing users to close/open position when they are not allowed to or in the contrary blocking users from closing/opening positions when they should be allowed to.

## Impact

If for some reason the value of `maxLTV` is set greater than `1e4` it can impact the protocol as it will compromise the Ichi vault open/close operations and cause a loss of funds for the users or the protocol, this issue can potentially lead to the following :

- some users are allowed to open/close positions when they should not be able to.

- users are not allowed to open/ close positions when they should be able to do so.

- users are not able to reduce their position value when they should be able to do so.

- users are able to reduce their position value when they should not be able to do so.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L84-L99

## Tool used

Manual Review

## Recommendation

Add a check statement in the function `addCollateralsSupport` to ensure that the value of `maxLTV` is always less or equal than `1e4` : 

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
        if (maxLTVs[i] > 1e4) revert INVALID_MAX_LTV();
        maxLTV[strategyId][collaterals[i]] = maxLTVs[i];
    }

    emit CollateralsSupportAdded(strategyId, collaterals, maxLTVs);
}
```
