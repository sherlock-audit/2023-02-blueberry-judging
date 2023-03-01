8olidity

high

# The attacker can bypass the original logic of `reduceposition()`

## Summary

The attacker can bypass the original logic of `reduceposition()`

## Vulnerability Detail
In the `reduceposition()` function, the `dowithdraw()` function will be called, and the data corresponding to the `positions [POSITION_ID]` is operated in the `dowithdraw()` function. Then send the contract's `colltoken` to the address of `bank.executor()`. Finally, the LTV will be checked according to the input of `Strategyid`.

```solidity
function reducePosition(
    uint256 strategyId,
    address collToken,
    uint256 collAmount  
) external {
    doWithdraw(collToken, collAmount); // POSITION_ID
    doRefund(collToken);
    _validateMaxLTV(strategyId); // POSITION_ID
}
```

But there is a problem here. `strategyid` is introduced by users. The attacker can choose the appropriate `strategyid` to bypass the `debtValue >(collValue * maxLTV[strategyId][collToken]) / DENOMINATOR`. Once this condition is bypassed, the attacker can refund assets, because other read data are from `positions[POSITION_ID]`.

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

That is, the attacker can input a specific `strategyid` to affect the data of `bank.position_id`.
## Impact
The attacker can bypass the original logic of `reduceposition()`
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266-L274
## Tool used

Manual Review

## Recommendation
Consider this situation