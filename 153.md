obront

medium

# Complete debt size is not paid off for fee on transfer tokens, but users aren't warned

## Summary

The protocol seems to be intentionally catering to fee on transfer tokens by measuring token balances before and after transfers to determine the value received. However, the mechanism to pay the full debt will not succeed in paying off the debt if it is used with a fee on transfer token.

## Vulnerability Detail

The protocol is clearly designed to ensure it is compatible with fee on transfer tokens. For example, all functions that receive tokens check the balance before and after, and calculate the difference between these values to measure tokens received:
```solidity
function doERC20TransferIn(address token, uint256 amountCall)
    internal
    returns (uint256)
{
    uint256 balanceBefore = IERC20Upgradeable(token).balanceOf(
        address(this)
    );
    IERC20Upgradeable(token).safeTransferFrom(
        msg.sender,
        address(this),
        amountCall
    );
    uint256 balanceAfter = IERC20Upgradeable(token).balanceOf(
        address(this)
    );
    return balanceAfter - balanceBefore;
}
```

There is another feature of the protocol, which is that when loans are being repaid, the protocol gives the option of passing `type(uint256).max` to pay your debt in full:
```solidity
if (amountCall == type(uint256).max) {
    amountCall = oldDebt;
}
```
However, these two features are not compatible. If a user paying off fee on transfer tokens passes in `type(uint256).max` to pay their debt in full, the full amount of their debt will be calculated. But when that amount is transferred to the contract, the amount that the result increases will be slightly less. As a result, the user will retain some balance that is not paid off.

## Impact

The feature to allow loans to be paid in full will silently fail when used with fee on transfer tokens, which may trick users into thinking they have completely paid off their loans, and accidentally maintaining a balance.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L760-L775

## Tool used

Manual Review

## Recommendation

I understand that it would be difficult to implement a mechanism to pay fee on transfer tokens off in full. That adds a lot of complexity that is somewhat fragile.

The issue here is that the failure is silent, so that users request to pay off their loan in full, get confirmation, and may not realize that the loan still has an outstanding balance with interest accruing.

To solve this, there should be a confirmation that any user who passes `type(uint256).max` has paid off their debt in full. Otherwise, the function should revert, so that users paying fee on transfer tokens know that they cannot use the "pay in full" feature and must specify the correct amount to get their outstanding balance down to zero.