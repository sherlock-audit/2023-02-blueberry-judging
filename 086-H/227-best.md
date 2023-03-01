koxuan

high

# potential gains of collateral are not given back to position owner when withdrawLend is called

## Summary
According to the docs, borrow tokens will bring potential gains or losses to position owners depending on the price of the collateral when closing a position. However, a logic in the `withdrawLend` function will only return position owner the original price that they have borrowed. 

## Vulnerability Detail

According to the docs https://docs.blueberry.garden/earn/what-are-liquidations, 

So when would you have to worry about it? Well, since you've deposited crypto tokens, the value of your collateral is volatile and able to change as token prices move. 

However, as we see in `withdrawLend`, if the withdrawn amount is more than the underlyingAmount, the potential gains are not given back to the position owner. 

```solidity
        if (address(ISoftVault(bank.softVault).uToken()) == token) {
            ISoftVault(bank.softVault).approve(
                bank.softVault,
                type(uint256).max
            );
            wAmount = ISoftVault(bank.softVault).withdraw(shareAmount);
        } else {
            wAmount = IHardVault(bank.hardVault).withdraw(token, shareAmount);
        }


        wAmount = wAmount > pos.underlyingAmount
            ? pos.underlyingAmount
            : wAmount;
```


## Impact
Loss of fund to position owner when closing position.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L683-L695
## Tool used

Manual Review

## Recommendation

Recommend returning user their collateral and their potential gains.
