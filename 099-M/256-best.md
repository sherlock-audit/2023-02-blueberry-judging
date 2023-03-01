tives

medium

# No maxLTV validation in addCollateralsSupport

## Summary

No maxLTV validation in addCollateralsSupport

## Vulnerability Detail

You can accidentally set very high LTV in `addCollateralsSupport` and `_validateMaxLTV` would then allow arbitrary amounts of loans. User can then open too big positions and will be liquidated or protocol would become insolvent.

## Impact

Too big loans without collateral.

## Code Snippet

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
        // @audit no max LTV check here
        if (maxLTVs[i] == 0) revert ZERO_AMOUNT();
        maxLTV[strategyId][collaterals[i]] = maxLTVs[i];
    }

    emit CollateralsSupportAdded(strategyId, collaterals, maxLTVs);
}
```
[link](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol/#L84)

```solidity
_validateMaxLTV() {
	if (
		  debtValue >
		  (collValue * maxLTV[strategyId][collToken]) / DENOMINATOR
	) revert EXCEED_MAX_LTV();
```

## Tool used

Manual Review

## Recommendation

Verify collateral max LTV while adding it’s support to the protocol.