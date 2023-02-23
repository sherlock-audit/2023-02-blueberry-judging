obront

high

# Users can get around MaxLTV because of lack of strategyId validation

## Summary

When a user withdraws some of their underlying token, there is a check to ensure they still meet the Max LTV requirements. However, they are able to arbitrarily enter any `strategyId` that they would like for this check, which could allow them to exceed the LTV for their real strategy while passing the approval.

## Vulnerability Detail

When a user calls `IchiVaultSpell.sol#reducePosition()`, it removes some of their underlying token from the vault, increasing the LTV of any loans they have taken.

As a result, the `_validateMaxLTV(strategyId)` function is called to ensure they remain compliant with their strategy's specified LTV:
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
To summarize, this check:
- Pulls the position's total debt value
- Pulls the position's total value of underlying tokens
- Pulls the specified maxLTV for this strategyId and underlying token combination
- Ensures that `underlyingTokenValue * maxLTV > debtValue`

But there is no check to ensure that this `strategyId` value corresponds to the strategy the user is actually invested in, as we can see the `reducePosition()` function:
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
Here is a quick proof of concept to explain the risk:
- Let's say a user deposits 1000 DAI as their underlying collateral.
- They are using a risky strategy (let's call it strategy 911) which requires a maxLTV of 2X (ie `maxLTV[911][DAI] = 2e5`)
- There is another safer strategy (let's call it strategy 411) which has a maxLTV of 5X (ie `maxLTV[411][DAI] = 4e5`)
- The user takes the max loan from the risky strategy, borrowing $2000 USD of value. 
- They are not allowed to take any more loans from that strategy, or remove any of their collateral.
- Then, they call `reducePosition()`, withdrawing 1600 DAI and entering `411` as the strategyId. 
- The `_validateMaxLTV` check will happen on `strategyId = 411`, and will pass, but the result will be that the user now has only 400 DAI of underlying collateral protecting $2000 USD worth of the risky strategy, violating the LTV.

## Impact

Users can get around the specific LTVs and create significantly higher leverage bets than the protocol has allowed. This could cause the protocol to get underwater, as the high leverage combined with risky assets could lead to dramatic price swings without adequate time for the liquidation mechanism to successfully protect solvency.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266-L274

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L101-L113

## Tool used

Manual Review

## Recommendation

Since the collateral a position holds will always be the vault token of the strategy they have used, you can validate the `strategyId` against the user's collateral, as follows:

```solidity
address positionCollToken = bank.positions(bank.POSITION_ID()).collToken;
address positionCollId = bank.positions(bank.POSITION_ID()).collId;
address unwrappedCollToken = IERC20Wrapper(positionCollToken).getUnderlyingToken(positionCollId);
require(strategies[strategyId].vault == unwrappedCollToken, "wrong strategy");
```